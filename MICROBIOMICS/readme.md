# Environmental Sequencing Metadata Database

## Overview

This project organizes environmental sequencing metadata into a relational-style structure that can be managed in spreadsheets while behaving like a proper database.

The goals are to:

- create stable unique identifiers
- reduce copy-paste errors
- separate field metadata from laboratory and sequencing metadata
- support linking across sample types, assays, and sequencing runs
- make future expansion easier

The core entity chain is:

`Location-Year → Sampling Plot → BioSample → Extraction → Sequencing`

In practice, sequencing metadata is best handled as a run table plus a per-sample sequencing record table.

---

## Core entities

### 1. Location-Year (`LY`)

Represents a unique field location in a specific year.

Examples:

- `LY_T_2020`
- `LY_A_2022`
- `LY_T_2023`

Primary key:

- `location_year_id`

Typical metadata:

- location name
- location code
- year
- harvest date
- site notes

This is a master/reference table.

---

### 2. Sampling Plot (`SP`)

Represents a unique plot or field unit within a location-year.

Examples:

- `LY_T_2023__SP_001`
- `LY_A_2022__SP_051`

Primary key:

- `sampling_plot_id`

Foreign key:

- `location_year_id`

Typical metadata:

- plot number
- block
- subplot
- lane / parcel / grid location
- cultivar
- fertilizer treatment
- spray treatment
- density
- other agricultural metadata

This is a master/reference table and should be the main source of plot metadata.

---

### 3. BioSample (`BS`)

Represents one collected biological sample.

Examples:

- `LY_T_2023__SP_001__BS_C_20230627_01`
- `LY_A_2022__SP_051__BS_W_20220704_01`

Primary key:

- `biosample_id`

Foreign key:

- `sampling_plot_id`

Typical metadata:

- sample date
- sample material / sample type
- processing state (washed, crushed, etc.)
- replicate number
- original sample label
- notes

This table stores biological sampling metadata.

---

### 4. Extraction (`EX`)

Represents one DNA extraction from one biosample.

Examples:

- `...__EX_20240626_01`
- `...__EX_20221012_01`

Primary key:

- `extraction_id`

Foreign key:

- `biosample_id`

Typical metadata:

- extraction date
- extraction batch
- DNA plate number
- plate position
- concentration values
- extraction notes

This table stores extraction metadata.

---

### 5. Sequencing Run (`RUN`)

Represents one sequencing instrument run or run folder.

Examples:

- `RUN_seq240826_X6DU5`
- `RUN_seq230707_DWDQH`

Primary key:

- `run_id`

Typical metadata:

- run date
- platform
- run folder
- sequencing center
- assay category
- notes

---

### 6. Sequencing Record (`SR`)

Represents one extracted sample sequenced in one run.

Examples:

- `SR__...__16S__RUN_seq240826_X6DU5`
- `SR__...__WGS__RUN_seq230315_PF2CP`

Primary key:

- `sequencing_record_id`

Foreign keys:

- `extraction_id`
- `run_id`

Typical metadata:

- assay type
- amplicon
- library prep method
- library plate number
- index IDs
- index pair
- `seq_sample_id`
- `correct_seq_sample_id`
- read file names
- notes

This table stores sequencing metadata.

---

## Key relationships

- one `Location-Year` has many `Sampling Plots`
- one `Sampling Plot` has many `BioSamples`
- one `BioSample` can have one or more `Extractions`
- one `Extraction` can have one or more `Sequencing Records`
- one `Sequencing Run` contains many `Sequencing Records`

---

## Controls

Controls must be clearly separated from field-derived samples.

Examples include:

- ZymoBIOMICS microbial community DNA standard
- Ultrapure-water
- alex-control
- 1xPBS
- DES

Recommended handling:

- keep controls in the same `biosample`, `extraction`, and `sequencing_record` tables
- flag them with `is_control = TRUE`
- store a `control_type_id`
- do not assign them to real field plots
- do not mix them with agricultural plot metadata

Recommended control fields:

- `is_control`
- `control_type_id`
- `control_class`
- `sample_origin_type`

Examples:

- `sample_origin_type = control`
- `control_class = positive`
- `control_class = negative`
- `control_class = blank`

---

## Recommended table structure

### `location_year`
- `location_year_id` (PK)
- `location_name`
- `location_code`
- `year`
- `harvest_date`
- `notes`

### `sampling_plot`
- `sampling_plot_id` (PK)
- `location_year_id` (FK)
- `plot_number`
- `sample_number`
- `block`
- `subplot`
- `grid_number`
- `grid_point`
- `cultivar`
- `fertilizer`
- `spray`
- `density`
- `notes`

### `biosample`
- `biosample_id` (PK)
- `sampling_plot_id` (FK, nullable for controls)
- `sample_date`
- `sample_material`
- `processing_state`
- `replicate_number`
- `source_label`
- `is_control`
- `control_type_id`
- `notes`

### `extraction`
- `extraction_id` (PK)
- `biosample_id` (FK)
- `extraction_date`
- `dna_plate_number`
- `plate_position`
- `dna_concentration`
- `batch_id`
- `notes`

### `sequencing_run`
- `run_id` (PK)
- `run_date`
- `platform`
- `run_folder`
- `assay_category`
- `notes`

### `sequencing_record`
- `sequencing_record_id` (PK)
- `extraction_id` (FK)
- `run_id` (FK)
- `assay_type`
- `amplicon`
- `library_prep`
- `lib_plate_number`
- `plate_position`
- `s5_index_id`
- `n7_index_id`
- `index_pair`
- `seq_sample_id`
- `correct_seq_sample_id`
- `forward_read_file`
- `reverse_read_file`
- `notes`

---

## ID naming conventions

Recommended principles:

- IDs must be unique and stable
- IDs should include parent context where helpful
- IDs should not depend only on dates
- IDs should not depend only on assay names
- IDs should not be reused for multiple records

Examples:

- `LY_T_2023`
- `LY_T_2023__SP_003`
- `LY_T_2023__SP_003__BS_C_20230627_01`
- `LY_T_2023__SP_003__BS_C_20230627_01__EX_20240626_01`
- `RUN_seq240826_X6DU5`
- `SR__LY_T_2023__SP_003__BS_C_20230627_01__EX_20240626_01__16S__RUN_seq240826_X6DU5`

---

## Best practices for spreadsheet data entry

- maintain one sheet per table
- keep one primary key column in every sheet
- use foreign keys instead of copying metadata by hand
- use dropdowns or lookup validation for controlled vocabularies
- never type plot metadata into sequencing sheets manually
- preserve raw source labels in separate columns
- distinguish clearly between raw labels and canonical corrected IDs
- keep control samples flagged explicitly
- avoid placeholder IDs that collapse different records into one fake record
