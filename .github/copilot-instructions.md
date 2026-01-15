# AI Copilot Instructions for BRFSS Diabetes Trends

## Project Overview
This repository processes Behavioral Risk Factor Surveillance System (BRFSS) data from 2015–2024 to analyze diabetes trends. The core workflow is a single Jupyter notebook (`analysis/1_data_download.ipynb`) that downloads raw XPT files from CDC, harmonizes variables across years, decodes numeric codes to human-readable labels, and exports a unified CSV.

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
