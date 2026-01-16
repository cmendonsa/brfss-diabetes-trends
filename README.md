# BRFSS Diabetes Trends Analysis (2015-2024)

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)
[![Code style: PEP 8](https://img.shields.io/badge/code%20style-pep8-orange.svg)](https://www.python.org/dev/peps/pep-0008/)

## Overview

This repository processes Behavioral Risk Factor Surveillance System (BRFSS) data spanning 2015–2024 to analyze diabetes trends and other health indicators. The project includes a comprehensive data processing pipeline that downloads raw data from the CDC, harmonizes variables across years, normalizes inconsistent coding schemes, decodes numeric values to human-readable labels, and outputs a unified dataset suitable for longitudinal analysis.

### Key Features

- **Automated CDC Data Download**: Intelligently downloads and caches XPT files with exponential backoff retry logic
- **Schema Harmonization**: Maps year-specific column names to canonical variable names using JSON configuration
- **Code Normalization**: Converts inconsistent coding across years (e.g., day-count fields) to standardized formats
- **Comprehensive Decoding**: Transforms numeric codes to descriptive labels using bidirectional mapping dictionaries
- **Memory-Efficient Processing**: Streams multi-year data sequentially to avoid memory overflow on large datasets
- **Robust Error Handling**: Gracefully handles download failures, malformed data, and missing columns
- **Detailed Logging**: Comprehensive logging tracks download progress, processing steps, and data transformations

## Table of Contents

- [Data Source](#data-source)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage](#usage)
- [Data Schema](#data-schema)
- [Configuration](#configuration)
- [Architecture](#architecture)
- [Contributing](#contributing)
- [License](#license)

## Data Source

Raw survey data is sourced from the Centers for Disease Control and Prevention (CDC) BRFSS annual surveys in SAS Transport format (.XPT):

- **Official Source**: [CDC BRFSS Annual Data](https://www.cdc.gov/brfss/annual_data/annual_data.htm)
- **Data Period**: 2015–2024 (10 years)
- **File Format**: SAS Transport (.XPT) files, one per year
- **Coverage**: Approximately 400,000 interviews per year from all U.S. states and territories

## Project Structure

```
brfss-diabetes-trends/
├── README.md                      # This file
├── LICENSE                        # GPL-3.0 license
├── requirements.txt               # Python package dependencies
├── .github/
│   └── copilot-instructions.md    # AI assistant instructions
├── analysis/
│   └── 1_data_download.ipynb      # Main processing pipeline
├── config/
│   ├── VAR_MAP.json               # Variable name mappings (canonical → raw)
│   ├── VALUE_MAP.json             # Per-year code overrides
│   └── VALUE_TEXT_MAP.json        # Universal code → label mappings
├── data_codebooks/                # CDC official codebook references (HTML)
│   ├── LLCP 2019_ Codebook Report.html
│   ├── USCODE22_LLCP_102523.HTML
│   ├── USCODE23_LLCP_021924.HTML
│   └── USCODE24_LLCP_082125.HTML
├── data_raw/                      # Cached raw data (ignored by git)
│   ├── 2015/ ... 2024/            # Year-specific directories
│   └── *.zip, *.XPT               # Downloaded and extracted files
└── data_processed/                # Output location for processed CSVs
    └── BRFSS_2015_2024_HARMONIZED_DECODED.csv  # Final unified dataset
```

## Installation

### Prerequisites

- Python 3.9 or higher
- pip package manager
- ~2 GB disk space for downloaded data
- ~500 MB RAM (simultaneous processing of single years)

### Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/callistus-neo/brfss-diabetes-trends.git
   cd brfss-diabetes-trends
   ```

2. **Create a virtual environment (recommended):**
   ```bash
   # Using venv
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   
   # Or using conda
   conda create -n brfss python=3.9
   conda activate brfss
   ```

3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

## Quick Start

### Running the Complete Pipeline

```bash
# Launch Jupyter
jupyter notebook analysis/1_data_download.ipynb

# Or use JupyterLab
jupyter lab analysis/1_data_download.ipynb
```

Then execute all cells in the notebook. The pipeline will:

1. Download raw XPT files for years 2015–2024 into `data_raw/{year}/`
2. Read each year's data and validate column mappings
3. Normalize variable names to canonical format
4. Normalize inconsistent coding (especially day-count fields)
5. Decode numeric values to human-readable labels
6. Stream results to `data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv`

Expected output:
```
[2026-01-16 14:23:15] INFO - Mappings loaded successfully.
[2026-01-16 14:23:15] INFO - - VAR_MAP: 20 fields
[2026-01-16 14:23:15] INFO - - VALUE_MAP: 12 fields
[2026-01-16 14:23:15] INFO - - VALUE_TEXT_MAP: 20 fields
[2026-01-16 14:23:16] INFO - Processing year 2015...
...
[2026-01-16 14:56:42] INFO - [WROTE] 2024: 32,401 rows → 'BRFSS_2015_2024_HARMONIZED_DECODED.csv'
```

### Processing Specific Years

```python
# Load and process a single year
from pathlib import Path
import sys
sys.path.append('analysis')

# Execute notebook cells or import functions
df_2020 = load_year(2020)
print(df_2020.head())
print(df_2020.info())
```

## Usage

### Main Entry Point: `load_multi_year()`

Processes all years (2015–2024) sequentially and writes to CSV:

```python
output_file = load_multi_year(
    output_path="data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv"
)
```

### Single-Year Processing: `load_year(year)`

Process an individual year:

```python
df_2023 = load_year(2023)
# Returns: DataFrame with canonical columns + YEAR
```

### Data Validation

```python
import pandas as pd

# Load processed data
df = pd.read_csv("data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv")

# Verify shape and structure
print(f"Total records: {len(df):,}")
print(f"Columns: {df.columns.tolist()}")
print(f"Year distribution:\n{df.groupby('YEAR').size()}")

# Check for missing values
print(f"Missing values per column:\n{df.isna().sum()}")

# Sample decoded data
print(df[['DIABETES', 'HEALTH_STATUS', 'YEAR']].head(10))
```

## Data Schema

The output CSV contains 24 columns: 23 canonical health variables + YEAR. All values are human-readable labels (decoded from numeric codes).

| Column Name                 | Description                                              | Example Values                |
|-----------------------------|----------------------------------------------------------|-------------------------------|
| `YEAR`                      | Survey year                                              | 2015–2024                     |
| `HEALTH_STATUS`             | Self-reported general health                             | "Good", "Poor", "Refused"     |
| `PHYSICAL_HEALTH_STATUS`    | Days physical health not good (past 30 days)            | "Zero", "1-13 days", "14+ days" |
| `MENTAL_HEALTH_STATUS`      | Days mental health not good (past 30 days)              | "Zero", "1-13 days", "14+ days" |
| `EXERCISE`                  | Participation in physical activity                       | "Yes", "No", "Refused"        |
| `HEALTH_CARE_COVERAGE`      | Health insurance status                                  | "Covered", "Not covered"      |
| `SMOKER`                    | Current smoking status                                   | "Current", "Former", "Never"  |
| `DRINKER`                   | Alcohol consumption status                               | "Yes", "No", "Refused"        |
| `SOCIAL_DRINKER`            | Binge drinking status                                    | "Yes", "No", "Refused"        |
| `HEAVY_ALCOHOL_CONSUMPTION` | Heavy alcohol consumption                                | "Yes", "No", "Refused"        |
| `HEART_ATTACK`              | History of heart attack (ever told)                      | "Yes", "No", "Refused"        |
| `STROKE`                    | History of stroke (ever told)                            | "Yes", "No", "Refused"        |
| `DIABETES`                  | Diabetes status or type                                  | "Yes", "Gestational", "No"    |
| `ARTHRITIS`                 | Arthritis status                                         | "Yes", "No", "Refused"        |
| `MARITAL_STATUS`            | Marital status                                           | "Married", "Never married"    |
| `EMPLOYMENT`                | Employment status                                        | "Employed", "Unemployed"      |
| `SEX`                       | Sex of respondent                                        | "Male", "Female"              |
| `AGE_CATEGORIES`            | Age group (5-year brackets)                              | "18-24", "25-29", "65+"       |
| `BMICAT`                    | BMI category                                             | "Underweight", "Normal", "Obese" |
| `EDUCATION_LEVEL`           | Highest education attained                               | "Less than HS", "College grad" |
| `INCOME`                    | Annual household income range                            | "<$10k", "$50k-$75k", "$75k+"  |

### Mapping Configuration

Variable mappings are managed in JSON configuration files under `config/`:

- **`VAR_MAP.json`**: Maps 23 canonical variable names to year-specific raw XPT column names (handles schema drift across years)
- **`VALUE_TEXT_MAP.json`**: Universal numeric code → label mappings (e.g., 1="Yes", 2="No", 9="Refused")
- **`VALUE_MAP.json`**: Year-specific code overrides when a year's encoding differs from the universal mapping

## Configuration

### Adding a New Variable

To add a new health variable to the dataset:

1. **Locate the variable** in CDC codebooks (`data_codebooks/`):
   - Search for variable name and response codes in the HTML files
   - Note raw column names for each year (e.g., `_DIABETE4`, `_CHCVDIS1`)

2. **Update `VAR_MAP.json`**:
   ```json
   {
     "NEW_VARIABLE": {
       "2015": "RAW_COL_2015",
       "2016": "RAW_COL_2016",
       "2017": "RAW_COL_2017",
       ...
       "2024": "RAW_COL_2024"
     }
   }
   ```
   All years are required. If a variable didn't exist in a year, use `null`.

3. **Update `VALUE_TEXT_MAP.json`**:
   ```json
   {
     "NEW_VARIABLE": {
       "1": "Yes",
       "2": "No",
       "7": "Not sure",
       "9": "Refused"
     }
   }
   ```

4. **Optional: Add per-year overrides** to `VALUE_MAP.json` if coding differs by year:
   ```json
   {
     "NEW_VARIABLE": {
       "2018": {
         "1": "Yes",
         "2": "No",
         "7": "Maybe"
       }
     }
   }
   ```

5. **Test** with a single year before running the full pipeline:
   ```python
   df_test = load_year(2020)
   print(df_test[['NEW_VARIABLE']].value_counts(dropna=False))
   ```

### Modifying Normalization Rules

For day-count fields (e.g., `PHYSICAL_HEALTH_STATUS`), normalization ensures cross-year consistency:

- **2015**: Raw day counts (0–31, 88=zero days, 99=refused)
- **2016+**: Categorical (1=zero, 2=1–13 days, 3=14+ days, 9=refused)

The `normalize_days()` function converts 2015 format to 2016+ format. Modify this function in the notebook if additional normalization is needed.

## Architecture

### Data Processing Pipeline

```
1. ensure_xpt(year) → Download/Cache XPT
   ↓
2. load_year(year) → Read XPT → Map columns → Normalize → Validate → Return DataFrame
   ↓
3. load_multi_year() → Loop years → Decode values → Write CSV (append) → Free memory
   ↓
4. BRFSS_2015_2024.csv (Final unified dataset)
```

### Key Implementation Details

**Memory Management**: Multi-year processing streams sequentially (not all-in-memory) using CSV append mode. Each year is deleted after writing and garbage collected explicitly to prevent memory overflow.

**Error Resilience**: 
- Download failures retry with exponential backoff (1s, 2s, 4s delays)
- Missing columns are added as NA (doesn't halt processing)
- Decoding errors are logged; original values preserved if mapping fails

**Configuration-Driven Design**: All year-specific logic resides in JSON configs (VAR_MAP, VALUE_MAP, VALUE_TEXT_MAP), not hardcoded in functions.

## Contributing

Contributions are welcome! Areas for enhancement:

- [ ] Support for additional BRFSS years (pre-2015)
- [ ] Automated variable discovery from CDC codebooks
- [ ] Unit tests for mapping validation
- [ ] Command-line interface for notebook execution
- [ ] Documentation generation from codebooks
- [ ] Performance benchmarking and optimization

**To contribute:**

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## Related Resources

- [CDC BRFSS Documentation](https://www.cdc.gov/brfss/)
- [BRFSS Annual Data](https://www.cdc.gov/brfss/annual_data/annual_data.htm)
- [SAS Transport Format](https://en.wikipedia.org/wiki/SAS_Transport_format)
- [Pandas Documentation](https://pandas.pydata.org/)

## License

This project is licensed under the GNU General Public License v3.0. See the [LICENSE](LICENSE) file for complete details.

The BRFSS data itself is in the public domain and made available by the CDC.

---

**Author**: Callistus Mendonca  
**Repository**: [github.com/cmendonsa/brfss-diabetes-trends](https://github.com/cmendonsa/brfss-diabetes-trends)  
**Last Updated**: January 2026