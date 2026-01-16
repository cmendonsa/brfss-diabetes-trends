# AI Copilot Instructions for BRFSS Diabetes Trends

**Document Version**: 1.1  
**Last Updated**: January 2026  
**Audience**: AI Assistants, Developers, Contributors

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture & Data Flow](#architecture--data-flow)
3. [Critical Patterns & Conventions](#critical-patterns--conventions)
4. [Common Tasks & Workflows](#common-tasks--workflows)
5. [Debugging Guide](#debugging-guide)
6. [Configuration Reference](#configuration-reference)
7. [Key Function Signatures](#key-function-signatures)
8. [Performance & Optimization](#performance--optimization)
9. [Guidelines for AI Agents](#guidelines-for-ai-agents)

---

## Project Overview

This repository processes Behavioral Risk Factor Surveillance System (BRFSS) data spanning 2015–2024 to analyze diabetes trends and related health indicators. The core workflow is a single, comprehensive Jupyter notebook (`analysis/1_data_download.ipynb`) that:

1. **Downloads** raw XPT files from the CDC with intelligent caching
2. **Harmonizes** variables across years despite schema drift
3. **Normalizes** inconsistent coding schemes (e.g., day-count fields)
4. **Decodes** numeric values to human-readable labels
5. **Exports** a unified CSV dataset for analysis

### Scope & Constraints

- **Data Years**: 2015–2024 (10 years of continuous data)
- **File Format**: SAS Transport (.XPT) format from CDC
- **Output**: Single CSV file (`BRFSS_2015_2024_HARMONIZED_DECODED.csv`) with 24 columns
- **Memory Constraint**: ~500 MB – 1 GB per year (sequential processing required)
- **Processing Time**: ~30–60 minutes for full pipeline

---

## Architecture & Data Flow

### Input Sources

| Source | Format | Location | Purpose |
|--------|--------|----------|---------|
| CDC BRFSS Data | SAS XPT | `https://www.cdc.gov/brfss/annual_data/{year}/files/LLCP{year}XPT.zip` | Raw survey responses |
| Codebooks | HTML | `data_codebooks/` | Variable names and code mappings |
| Config Files | JSON | `config/*.json` | Variable mappings and transformations |

### Processing Pipeline (Notebook Cells)

```
1. [Load Configuration]
   └─ Load VAR_MAP.json, VALUE_TEXT_MAP.json, VALUE_MAP.json
   └─ Verify all config files valid JSON and contain required keys

2. [Ensure XPT Files Exist]
   ├─ ensure_xpt(year) for each year 2015–2024
   ├─ Download from CDC if missing (with exponential backoff)
   └─ Cache locally in data_raw/{year}/LLCP{year}.XPT

3. [Process Each Year Sequentially]
   ├─ load_year(year)
   │  ├─ Read XPT file using pd.read_sas()
   │  ├─ Validate required columns exist via VAR_MAP
   │  ├─ Rename columns to canonical names
   │  ├─ Normalize day-count fields (2015 only)
   │  └─ Return DataFrame with canonical columns + YEAR
   │
   └─ load_multi_year()
      ├─ Loop: for each year in [2015, ..., 2024]
      ├─ Call load_year(year)
      ├─ Decode numeric codes to labels via VALUE_TEXT_MAP
      ├─ Append to CSV file (mode="a" except first year mode="w")
      ├─ Delete df_year and gc.collect() to free memory
      └─ Return path to final CSV

4. [Validation]
   ├─ Verify output CSV shape matches logged row counts
   ├─ Check all 24 columns present (23 canonical + YEAR)
   └─ Validate year distribution and data types
```

### Output

- **Primary**: `data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv`
- **Size**: ~2–3 GB (typically 4+ million rows)
- **Encoding**: UTF-8 with comma delimiters
- **Schema**: 24 columns (see [Data Schema](#data-schema) in README)

---

## Critical Patterns & Conventions

### 1. Variable Mapping Strategy

A two-tier mapping system handles year-to-year schema drift:

#### VAR_MAP: Canonical → Raw Column Names

Maps canonical variable names (e.g., `HEALTH_STATUS`) to year-specific raw XPT column names (e.g., `_RFHLTH`).

**Structure**:
```json
{
  "CANONICAL_NAME": {
    "2015": "RAW_COL_2015",
    "2016": "RAW_COL_2016",
    ...
    "2024": "RAW_COL_2024"
  }
}
```

**Key Points**:
- All years (2015–2024) are required keys
- Some variables stable across years (e.g., `_RFHLTH` for `HEALTH_STATUS`)
- Some variables change names year-to-year (e.g., `INCOME`: `_INCOMG` → `_INCOMG1` in 2021)
- Value should be `null` if variable didn't exist in a particular year

**Example: INCOME Schema Drift**
```json
"INCOME": {
  "2015": "_INCOMG",   // 5 income categories
  "2016": "_INCOMG",
  "2017": "_INCOMG",
  "2018": "_INCOMG",
  "2019": "_INCOMG",
  "2020": "_INCOMG",
  "2021": "_INCOMG1",  // Schema changed to 7 categories
  "2022": "_INCOMG1",
  "2023": "_INCOMG1",
  "2024": "_INCOMG1"
}
```

#### VALUE_TEXT_MAP: Universal Code Mappings

Constant mappings from numeric codes to human-readable labels. Used as the default for all years.

**Structure**:
```json
{
  "CANONICAL_NAME": {
    "1": "Label for code 1",
    "2": "Label for code 2",
    "9": "Refused"
  }
}
```

**Convention**:
- Code `9` typically means "Refused"
- Code `7` typically means "Not sure" or "Uncertain"
- Codes are strings in JSON but cast to integers during load

**Example: DIABETES Codes**
```json
"DIABETES": {
  "1": "Diabetes",
  "2": "Gestational",
  "3": "No",
  "4": "PreDiabetes",
  "7": "Not sure",
  "9": "Refused"
}
```

#### VALUE_MAP: Per-Year Code Overrides (Rare)

Used when a specific year's codes differ significantly from the universal mapping.

**Structure**:
```json
{
  "FIELD_NAME": {
    "YEAR": {
      "1": "Label for this year's code 1",
      "2": "Label for this year's code 2"
    }
  }
}
```

**Decoding Priority**:
1. Check `VALUE_MAP[canonical][year]` if exists
2. Otherwise fall back to `VALUE_TEXT_MAP[canonical]`

**Example: SEX in 2018 with Extra Code**
```json
"SEX": {
  "2018": {
    "1": "Male",
    "2": "Female",
    "7": "Not Sure",  // 2018-specific code
    "9": "Refused"
  }
  // Other years use VALUE_TEXT_MAP["SEX"]
}
```

### 2. Normalization Rules

**Day-Count Fields** have inconsistent coding across years:

| Field | Applies To |
|-------|-----------|
| `PHYSICAL_HEALTH_STATUS` | Days physical health not good (past 30 days) |
| `MENTAL_HEALTH_STATUS` | Days mental health not good (past 30 days) |

**Coding Differences**:

| Year | Coding | Example Values |
|------|--------|-----------------|
| **2015** | Raw day counts | `0`, `1`, `15`, `31`, `88` (zero), `99` (refused) |
| **2016+** | Categorical | `1` (zero), `2` (1–13 days), `3` (14+ days), `9` (refused) |

**Normalization Logic** (in `normalize_days()` function):

```
Input (2015): 0–31 → Output (2016+): 1 (zero)
Input (2015): 1–13 → Output (2016+): 2 (1–13 days)
Input (2015): 14–31 → Output (2016+): 3 (14+ days)
Input (2015): 88 → Output (2016+): 1 (zero days)
Input (2015): 99 → Output (2016+): 9 (refused)
```

**Critical**: Always normalize 2015 data to 2016 format before decoding to ensure cross-year comparisons are valid.

### 3. Memory Management

The notebook processes multiple years while maintaining low memory footprint:

**Strategy: Sequential Processing with Explicit Cleanup**

```python
for year in YEARS:
    df_year = load_year(year)           # ~500 MB–1 GB
    df_year_decoded = decode(df_year)   # Transform in-place or create new
    df_year_decoded.to_csv(output_path, mode="a", header=(year==first_year))
    
    del df_year                         # Explicit deletion
    del df_year_decoded
    gc.collect()                        # Force garbage collection
    # Memory now released; ready for next year
```

**Key Points**:
- Never materialize all years in memory simultaneously
- Use CSV `mode="a"` (append) to avoid re-reading previous data
- Call `gc.collect()` after each year's write
- Track memory deltas in logs (`_get_memory_bytes()` function)
- Prefer `psutil.Process().memory_info().rss` if available (accurate)
- Fall back to `tracemalloc` if psutil not installed

### 4. Error Handling & Resilience

| Error Type | Handling | Outcome |
|------------|----------|---------|
| Download fails | Exponential backoff: 1s, 2s, 4s delays (max 3 retries) | Retry or raise RuntimeError |
| ZIP is corrupted | Check magic bytes and attempt re-download | RuntimeError if repeated |
| XPT read fails | Log error, return empty DataFrame | Year skipped gracefully |
| Missing columns | Log warning, add as NA via `reindex()` | Data quality impacted but process continues |
| Decoding fails | Log per-column, preserve original value | Some fields remain numeric |

**Design Philosophy**: Fail gracefully rather than halt. Log all errors for post-run investigation.

---

## Common Tasks & Workflows

### Adding a New Variable

**Goal**: Include a new health variable in the output dataset.

**Steps**:

1. **Find Raw Column Names** in CDC codebooks (`data_codebooks/`):
   - Search HTML files for variable concept (e.g., "diabetes", "physical activity")
   - Locate raw column name (e.g., `_DIABETE4`, `_CHCVDIS1`)
   - Note any year-to-year changes in variable names or codes

2. **Update `VAR_MAP.json`**:
   ```json
   {
     "NEW_VARIABLE": {
       "2015": "RAW_COL_2015",
       "2016": "RAW_COL_2016",
       ...
       "2024": "RAW_COL_2024"
     }
   }
   ```
   - All 10 years are required
   - Use `null` if variable didn't exist in a year

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
   - Include all possible numeric codes
   - Use standardized labels (e.g., "Refused", "Not sure")

4. **Optional: Add Per-Year Overrides** to `VALUE_MAP.json` if coding differs:
   ```json
   {
     "NEW_VARIABLE": {
       "2018": {
         "1": "Yes",
         "2": "No",
         "7": "Maybe",
         "9": "Refused"
       }
     }
   }
   ```

5. **Optional: Add Normalization** if variable requires special handling (like day-count fields):
   - Create a normalization function in `load_year()`
   - Apply before column rename to canonical name
   - Document normalization logic in notebook comments

6. **Test with Single Year**:
   ```python
   df_test = load_year(2020)
   print(df_test[['NEW_VARIABLE']].value_counts(dropna=False))
   # Verify output has human-readable labels
   ```

7. **Run Full Pipeline**:
   ```python
   load_multi_year(output_path="data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv")
   ```

**Example: Adding DIET_FRUIT (hypothetical)**

Codebook shows: `_FRUTSUM` in 2016+, `FRUITJUC` in 2015 with codes 1=Daily, 2=Not daily, 9=Refused

1. **VAR_MAP entry**:
   ```json
   "DIET_FRUIT": {
     "2015": "FRUITJUC",
     "2016": "_FRUTSUM",
     ...
     "2024": "_FRUTSUM"
   }
   ```

2. **VALUE_TEXT_MAP entry**:
   ```json
   "DIET_FRUIT": {
     "1": "Daily",
     "2": "Not daily",
     "9": "Refused"
   }
   ```

3. **Test** to confirm decoding works for all years

### Extending the Processing Pipeline

**Goal**: Add custom transformations or validations.

**Design Principle**: Use configuration-driven approach. Avoid hardcoded year conditionals.

**Bad (Hardcoded)**:
```python
if year == 2021:
    df['INCOME'] = df['_INCOMG1']  # ❌ Don't do this
else:
    df['INCOME'] = df['_INCOMG']
```

**Good (Configuration-Driven)**:
```python
# In VAR_MAP.json:
"INCOME": {
  "2020": "_INCOMG",
  "2021": "_INCOMG1"
}

# In load_year():
raw_col = VAR_MAP[canonical][year]
df[canonical] = df[raw_col]
```

**Where to Add Logic**:
- **Mappings**: Add to `VAR_MAP`, `VALUE_TEXT_MAP`, `VALUE_MAP` (JSON config files)
- **Per-Year Transformations**: Add to `load_year()` function (e.g., normalization)
- **Per-Year Decoding**: Add to `VALUE_MAP` (per-year code overrides)
- **Validation**: Add to `validate_year_mappings()` function

### Validating Output CSV

**Goal**: Confirm data quality after `load_multi_year()` completes.

**Quick Validation Script**:
```python
import pandas as pd

df = pd.read_csv("data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv")

# 1. Shape & Structure
print(f"Shape: {df.shape}")  # Expected: (~4M rows, 24 columns)
print(f"Columns: {list(df.columns)}")

# 2. Year Distribution
print(f"\nRows per year:")
print(df.groupby('YEAR').size())

# 3. Data Types
print(f"\nData types:")
print(df.dtypes)

# 4. Missing Values
print(f"\nMissing values per column:")
print(df.isna().sum())

# 5. Decoded Values (spot check)
print(f"\nSample decoded data:")
print(df[['DIABETES', 'HEALTH_STATUS', 'YEAR']].head(10))

# 6. Value Uniqueness
print(f"\nUnique values (sample):")
for col in ['DIABETES', 'HEALTH_STATUS', 'SMOKER']:
    print(f"  {col}: {df[col].unique()}")
```

**Validation Checklist**:
- [ ] Shape matches total rows logged in `[WROTE]` lines
- [ ] 24 columns present (23 canonical + YEAR)
- [ ] Year distribution: ~350k–500k rows per year
- [ ] All canonical columns are object/string type (decoded)
- [ ] YEAR column is integer type
- [ ] Missing values are rare (expected < 1% per column)
- [ ] Sample values are human-readable (e.g., "Diabetes", "Yes", "Good")
- [ ] No numeric codes remain in decoded columns

---

## Debugging Guide

### Issue: Column Not Found in Raw XPT

**Symptoms**: `KeyError: 'RAW_COL_NAME'` or missing columns in output

**Diagnosis**:
```python
# 1. Check VAR_MAP configuration
from config import VAR_MAP
print(VAR_MAP.get('CANONICAL_NAME', {}).get(2020))  # Should return raw column name

# 2. Inspect actual raw XPT columns
df_raw = pd.read_sas("data_raw/2020/LLCP2020.XPT")
print(df_raw.columns.tolist())

# 3. Cross-reference codebook HTML
# Search: LLCP 2019_ Codebook Report.html for variable names

# 4. Verify VAR_MAP entry
# Ensure key matches year and value matches actual column name (case-sensitive)
```

**Solution**:
1. Search CDC codebook for correct raw column name
2. Update `VAR_MAP.json` with correct name for affected years
3. Re-run `load_year()` to verify fix

### Issue: Unexpected NAs in Output CSV

**Symptoms**: Missing values where data should exist

**Diagnosis**:
```python
# 1. Check raw vs. normalized data
df_raw = pd.read_sas("data_raw/2020/LLCP2020.XPT")
print(f"Raw column NAs: {df_raw['_RFHLTH'].isna().sum()}")

# 2. Check after mapping
df_year = load_year(2020)
print(f"Canonical column NAs: {df_year['HEALTH_STATUS'].isna().sum()}")

# 3. Check distribution before decoding
print(df_year['HEALTH_STATUS'].value_counts(dropna=False))

# 4. Verify VALUE_TEXT_MAP has all codes
print(VALUE_TEXT_MAP.get('HEALTH_STATUS'))
```

**Common Causes**:
- Missing codes in `VALUE_TEXT_MAP` → Add missing code mappings
- Normalization error (day-count fields) → Review `normalize_days()` logic
- Column mapping error → Verify `VAR_MAP` entry
- Data actually missing → Expected behavior (log and continue)

### Issue: Memory Spike During Multi-Year Run

**Symptoms**: Process slows or crashes during `load_multi_year()`

**Diagnosis**:
```python
# Monitor memory in notebook:
# Run with smaller year range first
YEARS = [2020, 2021, 2022]
load_multi_year(...)

# Check log output for memory deltas
# Look for: "[MEMORY] Before: X MB, After: Y MB, Delta: Z MB"
```

**Solutions**:
1. Verify `del df_year; gc.collect()` called after each year
2. Check for DataFrame copies in `load_year()` (use inplace operations)
3. Profile with memory_profiler:
   ```bash
   pip install memory-profiler
   python -m memory_profiler analysis/1_data_download.ipynb
   ```
4. Reduce processing batch or add intermediate cleanup

### Issue: CSV Rows Mismatch (Fewer Rows Than Expected)

**Symptoms**: Final CSV has significantly fewer rows than raw XPT total

**Diagnosis**:
```python
# Count raw rows
df_raw = pd.read_sas("data_raw/2020/LLCP2020.XPT")
print(f"Raw XPT rows: {len(df_raw):,}")

# Count in processed CSV
df_csv = pd.read_csv("data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv")
print(f"Processed CSV rows: {len(df_csv[df_csv['YEAR']==2020]):,}")

# Check for early `continue` statements
# Search notebook for: if len(df) == 0: continue
```

**Common Causes**:
- Years skipped due to read failures → Check logs for `Failed to read SAS XPT`
- Rows dropped during `reindex()` → Should only add columns, not remove rows
- Rows deleted during normalization → Review `normalize_days()` logic
- Write failures mid-pipeline → Check disk space and permissions

---

## Configuration Reference

### VAR_MAP.json Structure

```json
{
  "CANONICAL_NAME_1": {
    "2015": "RAW_COL_A",
    "2016": "RAW_COL_A",
    "2017": "RAW_COL_A",
    "2018": "RAW_COL_B",  // Changed names
    "2019": "RAW_COL_B",
    "2020": "RAW_COL_B",
    "2021": "RAW_COL_C",  // Changed again
    "2022": "RAW_COL_C",
    "2023": "RAW_COL_C",
    "2024": "RAW_COL_C"
  },
  "CANONICAL_NAME_2": { ... }
}
```

**Requirements**:
- All 10 years (2015–2024) must be present as keys
- Values are exact raw column names from XPT files
- Use `null` if variable missing in a year (not empty string)

### VALUE_TEXT_MAP.json Structure

```json
{
  "CANONICAL_NAME_1": {
    "1": "First option",
    "2": "Second option",
    "9": "Refused"
  },
  "CANONICAL_NAME_2": { ... }
}
```

**Requirements**:
- Keys are numeric codes as strings (will be cast to int)
- Values are human-readable labels
- Include all possible codes across all years
- Use consistent labels across years (except in VALUE_MAP overrides)

### VALUE_MAP.json Structure (Optional)

```json
{
  "CANONICAL_NAME": {
    "2018": {
      "1": "Year-specific label 1",
      "2": "Year-specific label 2"
    }
    // Other years use VALUE_TEXT_MAP
  }
}
```

**Use When**: A single year has different encoding than other years

---

## Key Function Signatures

### Core Pipeline Functions

#### `ensure_xpt(year: int, retries: int = 3, timeout: int = 30) → str`

Downloads and caches CDC XPT file for a specific year.

**Behavior**:
- Returns local path if file already exists (idempotent)
- Downloads ZIP from CDC if missing
- Extracts .XPT file and moves to `data_raw/{year}/LLCP{year}.XPT`
- Retries with exponential backoff (2^attempt seconds)

**Returns**: Full file path to cached XPT file

**Raises**:
- `RuntimeError`: ZIP archive doesn't contain .xpt files
- `requests.exceptions.RequestException`: Download fails after retries

#### `load_year(year: int) → pd.DataFrame`

Loads, processes, and harmonizes data for a single year.

**Process**:
1. Ensure XPT file cached via `ensure_xpt(year)`
2. Read XPT using `pd.read_sas()`
3. Validate expected columns exist (via `VAR_MAP`)
4. Rename columns to canonical names
5. Normalize special fields (e.g., day-count fields for 2015)
6. Add YEAR column
7. Return DataFrame with canonical columns

**Returns**: DataFrame with canonical columns + YEAR, or empty DataFrame on error

**Shape**: (num_respondents, 24) where columns are canonical names + YEAR

#### `load_multi_year(output_path: str = "...") → str`

Main pipeline function. Processes all years sequentially and writes to CSV.

**Process**:
```
for year in [2015, ..., 2024]:
    df_year = load_year(year)
    decode_values(df_year)
    df_year.to_csv(output_path, mode="a" or "w", header=first_year)
    del df_year; gc.collect()
```

**Returns**: Full path to output CSV file

**Side Effects**:
- Creates `data_processed/BRFSS_2015_2024_HARMONIZED_DECODED.csv`
- Logs progress and memory usage
- Caches XPT files in `data_raw/{year}/`

#### `validate_year_mappings(df_raw: pd.DataFrame, year: int) → list`

Validates that all canonical columns can be found in raw XPT.

**Returns**: List of missing canonical column names (empty if all valid)

**Side Effects**: Logs summary to stdout

#### `normalize_days(series: pd.Series) → pd.Series`

Converts 2015 raw day-count values to 2016+ categorical format.

**Input**: Values like 0–31, 88 (zero), 99 (refused)

**Output**: Values mapped to 1 (zero), 2 (1–13 days), 3 (14+ days), 9 (refused)

**Used On**:
- `PHYSICAL_HEALTH_STATUS`
- `MENTAL_HEALTH_STATUS`

---

## Performance & Optimization

### Benchmarks (Typical System: 8 GB RAM, SSD)

| Operation | Duration | RAM Usage |
|-----------|----------|-----------|
| Download all years (first run) | 20–40 min | <100 MB |
| Process single year (2020) | 2–3 min | 600–800 MB |
| Full multi-year pipeline | 25–45 min | 600–900 MB (per year) |

### Optimization Tips

1. **Skip Downloads**: Manually download years from CDC once, then set `retries=0`
2. **Parallel Processing**: Modify to process non-dependent tasks in parallel (with caution)
3. **Subset Years**: Test on small year ranges (e.g., `YEARS = [2020, 2021]`)
4. **Use SSD**: CSV writes are faster on solid-state drives
5. **Pre-allocate Memory**: Estimate RAM needs; add swap if necessary

### Bottlenecks

| Bottleneck | Impact | Solution |
|-----------|--------|----------|
| XPT read (`pd.read_sas`) | I/O bound (slow network) | Cache locally (via `ensure_xpt`) |
| CSV write (`to_csv`) | I/O bound (slow disk) | Use SSD or increase write buffer |
| Decoding loop | CPU bound | Vectorize if possible (reduce Python loops) |
| Memory fragmentation | Memory leak risk | Call `gc.collect()` explicitly per year |

---

## Guidelines for AI Agents

### DO ✅

- **Use configuration files** for all mappings (VAR_MAP, VALUE_TEXT_MAP, VALUE_MAP)
- **Respect the processing order**: read → validate → map → normalize → decode → write
- **Test changes** on single year (`load_year(2020)`) before full pipeline
- **Preserve missing values** as `pd.NA`; never drop rows for missing codes
- **Log errors** with context (year, column, raw value) before continuing
- **Memory is a constraint**: Stream years sequentially, delete explicitly, gc.collect()

### DON'T ❌

- **Hardcode column names or mappings** in function logic
- **Use year conditionals** outside config (e.g., `if year == 2021`)
- **Materialize all years** in memory simultaneously
- **Drop rows** with missing/invalid codes; preserve and log
- **Skip gc.collect()** calls after CSV writes
- **Modify raw data** in place; always copy or be explicit about mutations

### When Adding Features

1. Check if feature requires year-specific handling
2. If yes → Add to configuration JSON file
3. If no → Add to core function logic (with tests)
4. Always test on subset before full pipeline
5. Update documentation (this file + README)

### When Debugging

1. Check logs first (look for errors, warnings, deltas)
2. Isolate to specific year via `load_year(year)`
3. Inspect intermediate DataFrames (shape, dtypes, nulls)
4. Cross-reference codebook documentation
5. Verify configuration (JSON syntax, required keys)



## Architecture & Data Flow

### Input
- Raw SAS XPT files from CDC (one per year, 2015–2024) stored in `data_raw/{year}/LLCP{year}.XPT`
- Data codebooks in `data_codebooks/` (HTML format documenting variable names and codes)

### Processing Pipeline (Notebook)
1. **Download & Cache** (`ensure_xpt`): Downloads ZIP from CDC with exponential backoff retry logic; caches XPT locally to avoid re-downloading
2. **Load & Map** (`load_year`): Reads XPT using `pd.read_sas()`, validates that expected columns exist, maps year-specific column names to canonical names
3. **Normalize** (`load_year` + `normalize_days`): Converts health status fields to standardized 2016-format codes for cross-year consistency
4. **Decode** (`load_multi_year`): Maps numeric codes to human-readable labels using `VALUE_TEXT_MAP` before CSV export
5. **Aggregate** (`load_multi_year`): Streams years sequentially to CSV with memory tracking to avoid OOM

### Output
- Single CSV: `BRFSS_2015_2024_HARMONIZED_DECODED.csv` with canonical columns + YEAR

## Critical Patterns & Conventions

### Variable Mapping Strategy
Two-tier mapping system to handle year-to-year schema drift:
- **`VAR_MAP`**: Maps canonical names (e.g., `HEALTH_STATUS`) → year-specific raw column names (e.g., `_RFHLTH` in 2015–2024)
  - Structure: `{"CANONICAL": {year: "RAW_COL_NAME", ...}}`
  - Some variables remain stable across years; others change (e.g., `INCOME` splits into 7 categories in 2021+)
- **`VALUE_TEXT_MAP`**: Constant mappings from numeric codes → labels (e.g., `{1: "Good", 2: "Poor", 9: "Refused"}`)
  - Used as fallback; per-year overrides stored in `VALUE_MAP` but rarely needed

### Normalization Rules
**Day-count fields** (`PHYSICAL_HEALTH_STATUS`, `MENTAL_HEALTH_STATUS`) have inconsistent coding across years:
- 2015: Raw day counts (0–31, 88=zero, 99=refused)
- 2016+: Categorical (1=zero, 2=1–13 days, 3=14+ days, 9=refused)

**Always normalize to 2016 format** via `normalize_days()` to ensure cross-year comparisons are valid.

### Memory Management
- Streams years sequentially (not all-in-memory) using CSV `mode="a"` (append)
- Deletes `df_year` and calls `gc.collect()` after write to free memory
- Tracks memory before/after each year with `_get_memory_bytes()` and logs deltas
- Prefers `psutil` if available; falls back to `tracemalloc`

### Error Handling
- Download failures use exponential backoff (2^attempt seconds between retries)
- XPT read failures log to stdout and return empty DataFrame (skipped gracefully)
- Missing columns logged but don't halt processing; missing columns added as NA via `reindex()`
- Enum decoding errors caught and logged per-column per-year; original values preserved on exception

## Common Tasks & Workflows

### Adding a New Variable
1. Find raw column name(s) for each year from codebooks in `data_codebooks/` (search HTML for variable name and response codes)
2. Add entry to `VAR_MAP`: `"NEW_VAR": {2015: "RAW_2015", 2016: "RAW_2016", ...}` (all years required)
3. Add numeric→label mappings to `VALUE_TEXT_MAP`: `"NEW_VAR": {1: "Label1", 2: "Label2", ...}`
4. If coding differs drastically in 2015 vs others, add per-year override in `VALUE_MAP["NEW_VAR"]`
5. If day-count field (like `PHYSICAL_HEALTH_STATUS`), create normalization logic similar to `normalize_days()` and apply in `load_year()`
6. Test with single year: `df = load_year(2020)` to verify mappings before running full pipeline

**Example: Adding DIET_FRUIT (hypothetical fruit consumption)**
- Codebook shows `_FRUTSUM` in 2016+, `FRUITJUC` in 2015
- `VAR_MAP["DIET_FRUIT"] = {2015: "FRUITJUC", 2016: "_FRUTSUM", ..., 2024: "_FRUTSUM"}`
- `VALUE_TEXT_MAP["DIET_FRUIT"] = {1: "Daily", 2: "Not daily", 9: "Refused"}`
- If 2015 has different codes: add `VALUE_MAP["DIET_FRUIT"][2015] = {1: "Yes", 2: "No", 9: "Refused"}`

### Debugging Data Issues

**Column not found in raw XPT:**
- Check `validate_year_mappings()` output: prints missing fields per year
- Inspect raw XPT columns: `df_raw.columns.tolist()` after `pd.read_sas()`
- Cross-reference codebook HTML for actual column name (case-sensitive)
- Verify `VAR_MAP` entry has correct year key and raw column name

**Unexpected NAs in output CSV:**
- Run `load_year(year)` in isolation and check `df_year.isna().sum()` per column
- Verify normalization didn't drop valid codes (check `normalize_days()` logic)
- Check decoding step: does `VALUE_TEXT_MAP[col]` have all input codes as keys?
- Use `df_year[col].value_counts(dropna=False)` to see raw distribution before write

**Memory spike during multi-year run:**
- Check memory deltas in log output: `delta: +X MB` indicates memory not freed
- Verify `del df_year; gc.collect()` is called after each year's CSV write
- Profile with smaller year range first: `YEARS = [2020, 2021, 2022]` to isolate
- If still high, reduce year batch or add intermediate garbage collection in loop

**CSV rows mismatch (fewer rows than expected):**
- Check for early `continue` in loop (skipped years due to empty DataFrame)
- Verify all `load_year()` calls succeeded: check for `Failed to read SAS XPT` warnings
- Confirm no rows dropped during `reindex()` (should only add columns, not remove rows)
- Count raw XPT rows: `df_raw.shape[0]` after `pd.read_sas()` to compare

### Extending Processing
- All year-specific logic goes in `VAR_MAP`, `VALUE_MAP`, or `VALUE_TEXT_MAP` (not hard-coded in functions)
- Transformations applied in order: load → validate → map columns → normalize → decode → reindex → write
- To add validation: add logic in `validate_year_mappings()` or `load_year()` before `out` is finalized
- **Never hardcode year conditionals** (e.g., `if year == 2021`) outside config dicts—use config-driven approach instead

### Working with CDC Codebooks

CDC provides HTML codebooks in `data_codebooks/` with variable documentation:
- **Find variable**: Search HTML for concept name (e.g., "diabetes", "physical activity")
- **Extract raw column name**: Located near variable description (e.g., `_DIABETE4`)
- **Map response codes**: Look for "Value Label" or "Frequency" sections listing code→label pairs
- **Note year changes**: Some variables rename year-to-year; cross-reference multiple codebook files
- **Enum codes 7, 9**: Often mean "Not sure" or "Refused"; always include in mappings

**Example codebook excerpt:**
```
Variable Name: _RFHLTH (Reported Health Status)
Value Label:
  1 = Good
  2 = Poor
  9 = Refused
```
→ `"HEALTH_STATUS": {2015: "_RFHLTH", ..., 2024: "_RFHLTH"}`
→ `VALUE_TEXT_MAP["HEALTH_STATUS"] = {1: "Good", 2: "Poor", 9: "Refused"}`

### Validating the Output CSV

After `load_multi_year()` completes, verify:
- **Shape**: `df.shape` should match total rows logged in `[WROTE]` lines
- **Columns**: 23 canonical columns + YEAR = 24 total
- **Year distribution**: `df.groupby('YEAR').size()` shows data per year
- **Data types**: All canonical columns are object/string (decoded labels), YEAR is int64
- **Missing values**: `df.isna().sum()` shows expected NAs (rare for complete responses)
- **Sample decoding**: `df.head()` shows human-readable labels like "Diabetes", "Yes", "1-13 days"
- **CSV integrity**: Re-read CSV: `df2 = pd.read_csv('output.csv')` should match shape

**Quick validation script:**
```python
df = pd.read_csv("BRFSS_2015_2024_HARMONIZED_DECODED.csv")
print(f"Shape: {df.shape}")
print(f"Columns: {list(df.columns)}")
print(f"Rows per year:\n{df.groupby('YEAR').size()}")
print(f"Missing values:\n{df.isna().sum()}")
print(f"Sample row:\n{df.iloc[0]}")
```

## File Organization
```
analysis/1_data_download.ipynb     # Single notebook; all code here
data_raw/{year}/                   # Cached XPT + ZIP files (*.XPT/*.zip in .gitignore)
data_codebooks/                    # HTML reference docs from CDC
.github/copilot-instructions.md    # This file
```

## Key Function Signatures & Patterns

### Core Pipeline Functions
```python
ensure_xpt(year, retries=3, timeout=30) → str
  # Downloads & caches CDC XPT; returns local path
  # Idempotent: skips download if ZIP/XPT already exist
  # Raises RuntimeError if ZIP is bad or contains no .xpt

load_year(year) → pd.DataFrame
  # Single year pipeline: download → read → map → normalize → validate
  # Returns DataFrame with canonical columns + YEAR
  # Empty DataFrame on read failure (graceful skip)

load_multi_year(output_path="BRFSS_2015_2024_HARMONIZED_DECODED.csv") → str
  # Main entry point; processes all years sequentially
  # Streams to CSV with mode="a" (append) to manage memory
  # Returns output_path; prints row counts and memory deltas

validate_year_mappings(df_raw, year) → list
  # Returns list of missing canonical fields
  # Logs summary to stdout for debugging

normalize_days(series) → pd.Series
  # Converts 2015 raw day counts to 2016+ categorical codes
  # Input: 0–31 (days) → Output: 1 (zero), 2 (1-13), 3 (14+), 9 (refused)
  # Always call on PHYSICAL_HEALTH_STATUS, MENTAL_HEALTH_STATUS before decoding
```

### Decoding Functions (utilities, not in main pipeline but useful for testing)
```python
decode_value(canonical, year, val) → str | pd.NA
  # Single-value decoder; combines VALUE_MAP[canonical][year] + VALUE_TEXT_MAP fallback

decode_series(canonical, year, series) → pd.Series
  # Vectorized decoder for Series; preserves NAs
```

### Data Transformation Order (in `load_multi_year`):
```
1. load_year(year)              → DataFrame with raw numeric codes, canonical columns
2. reindex(canonical_cols)      → Ensure column order; add NAs for missing columns
3. Decode via VALUE_TEXT_MAP    → Convert numeric codes to human-readable labels
4. to_csv(mode="a"|"w")        → Write to output file
5. del + gc.collect()           → Free memory explicitly
```

## Mapping Dictionary Reference

### VAR_MAP Structure
Maps canonical names to year-specific raw XPT column names. All 10 years (2015–2024) required as keys.
```python
VAR_MAP = {
    "CANONICAL_NAME": {
        2015: "RAW_COL_2015",
        2016: "RAW_COL_2016",
        # ... up to 2024
    },
    # ...
}
```
**Year-to-year example: INCOME changes from 5 to 7 categories in 2021**
```python
"INCOME": {
    2015: "_INCOMG",   # 5 categories
    ...,
    2020: "_INCOMG",
    2021: "_INCOMG1",  # 7 categories
    ...,
    2024: "_INCOMG1"
}
```

### VALUE_TEXT_MAP Structure
Constant mappings (fallback for all years) from numeric code → human-readable label.
```python
VALUE_TEXT_MAP = {
    "CANONICAL_NAME": {
        1: "Label for code 1",
        2: "Label for code 2",
        9: "Refused",
        # ... all possible codes
    },
}
```
**Example:**
```python
"DIABETES": {
    1: "Diabetes",
    2: "Gestational",
    3: "No",
    4: "PreDiabetes",
    7: "Not sure",
    9: "Refused"
}
```

### VALUE_MAP Structure (rare, per-year overrides)
Used when a year's codes differ significantly from the constant mapping in VALUE_TEXT_MAP.
```python
VALUE_MAP = {
    "FIELD_NAME": {
        year: {code: label, ...},  # Override for specific year
        # ...
    },
}
```
**Example: SEX in 2018 includes extra code**
```python
"SEX": {
    2018: {
        1: "Male",
        2: "Female",
        7: "Not Sure",    # 2018 only
        9: "Refused"
    },
    # Other years fall back to VALUE_TEXT_MAP
}
```

## Performance & Memory Management Insights

- **XPT read bottleneck**: `pd.read_sas()` is I/O bound; caching (via `ensure_xpt`) is critical
- **Memory growth**: Each year's DataFrame held in memory until CSV write + `gc.collect()`. Typical per-year RAM: 500 MB – 1 GB
- **CSV append**: `to_csv(mode="a")` avoids re-reading previously written data
- **Memory tracking**: `_get_memory_bytes()` prefers psutil (accurate RSS) but falls back to tracemalloc (approximate)
- **Exponential backoff**: Download retries use `2^attempt` seconds (1s, 2s, 4s) to avoid overwhelming CDC servers

## Key Dependencies
- **pandas**: XPT read (`pd.read_sas`), data manipulation, nullable Int64 dtypes
- **numpy**: NaN/NA handling
- **requests**: HTTP download with streaming
- **zipfile**: Extract XPT from CDC ZIPs
- **psutil** (optional): More accurate memory profiling than tracemalloc

## Notes for AI Agents
- **Do not hardcode column names or mappings**—use `VAR_MAP` and value maps
- **Respect the encoding chain**: read (XPT) → validate → map → normalize → decode → write (CSV)
- **Preserve missing values as `pd.NA`** during all transformations; never drop rows with missing codes
- **Test mapping changes** on a single year first before running full multi-year pipeline
- **Memory is a constraint**; avoid materializing all years in memory simultaneously
