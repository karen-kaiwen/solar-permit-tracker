# Solar Permit Tracker — Automated Monthly Case Report (R)

Automated comparison of ~90,000 solar energy permit records per month, replacing manual Excel filtering with a reproducible R pipeline. Reduces monthly reporting time by **20-30%**.

![R](https://img.shields.io/badge/R-276DC3?style=flat&logo=r&logoColor=white)
![tidyverse](https://img.shields.io/badge/tidyverse-1A162D?style=flat&logo=r&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

---

## Table of Contents

- [Project Overview](#project-overview)
- [System Architecture](#system-architecture)
- [Technical Challenges & Solutions](#technical-challenges--solutions)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Setup & Usage](#setup--usage)
- [Output Files](#output-files)
- [Known Limitations](#known-limitations)
- [License](#license)

---

## Project Overview

Each month, a report of newly approved solar permit cases must be submitted to management. The original process involved manually filtering two large Excel exports (~90,000 rows combined) — which frequently caused Excel crashes, required repeated cross-referencing, and was prone to data errors.

This R script automates the entire workflow. Each month, only two variables need to be updated (file paths + report date), and the script handles data cleaning, merging, comparison, and Excel export automatically.

---

## System Architecture

```
┌──────────────────────┐          ┌──────────────────────┐
│   Approval Filings   │          │ Equipment Registrations│
│ approve_filling.xlsx │          │ equipment_reg.xlsx    │
└──────────┬───────────┘          └──────────┬───────────┘
           │                                 │
           ▼                                 ▼
   filter_solar_cases()              filter_solar_cases()
   (date conversion / filtering /    (date conversion / filtering /
    column selection)                 column selection)
           │                                 │
           └────────────────┬────────────────┘
                            │
                            ▼
                  Data Quality Check (stopifnot)
                  left_join(by = "permit_case_id")
                            │
                            ▼
                  ┌───────────────────┐
                  │  Reference Report │
                  └─────────┬─────────┘
                            │
                  Compare against last month
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
       ┌──────────┐    ┌──────────┐    ┌──────────┐
       │  New     │    │  New     │    │  New     │
       │ Approvals│    │Equipment │    │   Grid   │
       │  .xlsx   │    │  .xlsx   │    │  .xlsx   │
       └──────────┘    └──────────┘    └──────────┘
```

---

## Technical Challenges & Solutions

### Excel Serial Date Conversion
- **Problem**: Some date fields in the source files were stored as Excel serial numbers (e.g. `45291`). Reading them directly returned raw integers, making year filtering and date arithmetic impossible.
- **Solution**: Applied `as.Date(as.numeric(...), origin = "1899-12-30")` uniformly to convert serial numbers back to proper Date objects, ensuring correct date comparisons throughout.

### Reusable Filter Logic
- **Problem**: Both the approval filings and equipment registrations required the same six business filter conditions (contact person, year, case type, installation location, equipment type, generation type). Maintaining duplicate logic in two places risked inconsistency when conditions changed.
- **Solution**: Extracted shared filtering into a single `filter_solar_cases(df, date_col)` function. Any rule change updates both datasets automatically.

### Invisible Whitespace in Column Names
- **Problem**: The previous month's reference file contained invisible whitespace characters in column headers, causing `anti_join` to fail silently and produce incorrect new-case results.
- **Solution**: Applied `rename_with(~ gsub("\\s+", "", .x))` immediately on import to strip all whitespace from column names, ensuring consistent join keys.

### Silent Row Inflation from Duplicate Case IDs
- **Problem**: Source files are exported from an external government system outside this script's control. If the equipment registration table contained multiple rows for the same permit case ID, `left_join` would silently expand that record into multiple rows — inflating the report count with no error message.
- **Solution**: Added `stopifnot` duplicate-ID checks on both cleaned datasets before the join. If duplicates are detected, execution halts immediately to prevent a corrupted report from being generated.

```r
stopifnot("Duplicate permit case IDs detected!" = !any(duplicated(approval_data$permit_case_id)))
```

---

## Features

| Feature | Description |
|---------|-------------|
| 🔍 **Shared Filter Function** | Six business filter conditions encapsulated in `filter_solar_cases()` — applied consistently to both source datasets |
| 📅 **Date Format Normalization** | Handles both Excel serial numbers and string date formats automatically |
| ✅ **Pre-merge Data Validation** | `stopifnot` duplicate checks halt execution before a corrupted report is produced |
| 🔗 **Cross-table Join** | Merges approval and equipment records on permit case ID via `left_join` |
| 📊 **Three-category Comparison** | Separately identifies new approvals, new equipment registrations, and new grid connections |
| 📤 **Auto-named Export** | Output filenames include the report date automatically, preventing accidental overwrites |

---

## Prerequisites

- R 4.0 or higher
- Install required packages:

```r
install.packages(c("tidyverse", "lubridate", "readxl", "writexl", "openxlsx"))
```

- Three source files:
  - Approval filings table (`.xlsx`)
  - Equipment registration table (`.xlsx`)
  - Last month's reference report (`.xlsx`, first row is a header note — loaded with `skip = 1`)

---

## Setup & Usage

1. Open the script and update the path configuration block at the top:

```r
PATH_APPROVALS    <- "YOUR/PATH/approve_filling_YYYYMMDD.xlsx"
PATH_EQUIPMENT    <- "YOUR/PATH/equipment_reg_YYYYMMDD.xlsx"
PATH_LAST_MONTH   <- "YOUR/PATH/last_month_reference.xlsx"

REPORT_DATE       <- "YYYYMMDD"   # Report date for output filenames
```

2. Run the full script. If duplicate case IDs are found in either source file, execution will stop and display an error message.
3. Once completed without errors, output files will be saved to the working directory.

---

## Output Files

| File | Contents |
|------|----------|
| `reference_{DATE}.xlsx` | Full merged dataset (approvals + equipment registrations) |
| `new_approvals_{DATE}.xlsx` | Cases approved this month, not present in last month's records |
| `new_registrations_{DATE}.xlsx` | Equipment registrations new this month |
| `new_grid_connections_{DATE}.xlsx` | Cases with existing approval but new or missing equipment registration |

---

## Known Limitations

- Date conversion logic depends on consistent source file formatting; if the source format changes, the `mutate` conversion steps must be updated accordingly.
- File paths are configured as local absolute paths; update them when running on a different machine.
- The script does not validate whether expected columns exist in the source files; if the source schema changes, `select` and `rename` references must be updated to match.

---

## License

MIT License

This codebase has been de-identified for public sharing. All file paths, agency names, and case identifiers are placeholders — replace them with your actual values before use.

## Acknowledgements

Developed with AI pair programming assistance from [Gemini](https://gemini.google.com/) (Google).
System architecture, core logic, and requirements designed by the author.
