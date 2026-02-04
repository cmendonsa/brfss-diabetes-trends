# Project Context: BRFSS Diabetes Longitudinal Study

## 1. Project Overview
* **Goal:** Longitudinal Predictive Analysis of Diabetes Risk using Behavioral data.
* **Data Source:** CDC BRFSS (Behavioral Risk Factor Surveillance System).
* **Timeframe:** 2015–2024 (Longitudinal).
* **Current Phase:** Interim Report generation (following Synopsis submission).
* **Immediate Focus:** Comparative analysis of 2023 vs. 2024 Codebooks to ensure feature continuity.

## 2. Technical Stack & Environment
* **Language:** Python 3.x
* **Data Formats:** * Input: SAS Transport files (`.xpt`) and SAS Program files (`.sas`).
    * Output: Cleaned CSV datasets / Pandas DataFrames.
* **Key Libraries:** * `pandas` (Data manipulation)
    * `pyreadstat` (Reading .xpt files and extracting metadata)
    * `numpy` (Numerical operations)
    * `scikit-learn` (Predictive modeling - future phase)

## 3. Data Governance & Transformation Rules
* **Variable Mapping:** * Standardize variable names across years (e.g., handling changes in `_DIABETE3` vs `_DIABETE4`).
    * Extract and apply variable labels and value labels from the `.sas` format files.
* **2023 vs. 2024 Compatibility:** * Explicitly check for deprecated fields or schema changes between the 2023 and 2024 datasets.
    * Flag any "Calculated Variables" (prefixed with `_`) that have changed definitions.
* **File Handling:** * Do not hardcode file paths; use relative paths or environment variables.
    * Ensure memory efficiency when loading large annual `.xpt` files.

## 4. Coding Standards for Agent
* **Documentation:** All data cleaning functions must include docstrings referencing the specific BRFSS Codebook year and page/variable name.
* **Reproducibility:** Prefer scripts that can regenerate the dataset from raw `.xpt` files over manual edits.
* **Testing:** When writing transformation logic, generate simple assertions to verify that categorical values (e.g., 1=Yes, 2=No) are mapped correctly.

## 5. User Preferences
* **Output Style:** When generating code, prioritize readability and step-by-step comments explaining the SAS-to-Python conversion logic.
* **Context:** The user is currently drafting the Interim Report; focus analysis on insights and data validity.