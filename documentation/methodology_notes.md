# Research Methodology Notes

**Project:** Longitudinal Predictive Analysis of Diabetes Risk (2015-2024)
**Author:** Callistus Mendonca
**Source Data:** CDC Behavioral Risk Factor Surveillance System (BRFSS)

---

## 1. Data Source and Acquisition
* **Primary Source:** The study utilizes data from the Centers for Disease Control and Prevention's (CDC) Behavioral Risk Factor Surveillance System (BRFSS), the largest continuously running health survey in the world.
* **Data Access:** Core survey files were acquired via the CDC BRFSS Annual Data Portal for the years 2015 through 2024.
* **Format:** Original files were downloaded in `.ZIP` and `.XPT` (SAS Transport) formats.
* **Reproducibility:** To bypass GitHub storage limits, the project repository includes scripts to automate the download and decompression of source files directly from the CDC rather than hosting the raw bulk data.

## 2. Schema Harmonization Strategy
A primary challenge in this longitudinal analysis is the variability in variable naming conventions and question phrasing across different survey years (e.g., alcohol consumption or e-cigarette use coding).
* **Cross-Walk Map:** A "Cross-Walk Map" was constructed to normalize field definitions across the distinct codebooks from 2015 to 2024.
* **Common Data Model (CDM):** Variables were mapped to a Common Data Model to ensure longitudinal consistency before being ingested into the analytical pipeline.

## 3. Study Scope and Design
* **Population:** The study analyzes a nationally representative sample of U.S. adults (18 years or older).
* **Timeline:** The analysis spans a ten-year period (2015–2024), capturing pre-pandemic, pandemic, and post-pandemic intervals.
* **Sample Size:** The cumulative dataset exceeds 4 million records, ensuring statistical saturation.
* **Inclusion Criteria:** Participants providing full core diabetes status information along with relevant demographic (e.g., Age, Sex, Race) and behavioral data.

## 4. Statistical Power and Sample Size
Formal power calculations were conducted to ensure sufficiency for specific research questions (RQ) despite the large dataset size.

| Metric | RQ1 (Generational) | RQ2 (Pandemic) | RQ3 (Predictor Stability) | RQ4 (Compounding Risk) |
| :--- | :--- | :--- | :--- | :--- |
| **Power Calculation (80% Power)** | 714 | 8,280 | N/A | 280 |
| **Confidence Interval (Precision)** | 1,067 (±3%) | 9,604 (±1%) | 9,604 (±1%) | 601 (±4%) |

## 5. Analytic Approach by Research Question
The study employs a mix of traditional statistical hypothesis testing and machine learning techniques.

### RQ1: Generational Shift
* **Objective:** Compare diabetes prevalence between young adults (18-29) and older adults (30+).
* **Statistical Methods:** Descriptive prevalence comparisons; Two-proportion z-test.
* **Machine Learning:** Binary Logistic Regression to quantify odds ratios after controlling for confounders.

### RQ2: Pandemic Impact
* **Objective:** Assess if COVID-19 materially altered diabetes prevalence or risk distributions.
* **Statistical Methods:** Difference-in-proportion testing comparing three periods: Pre-Pandemic (2015-2019), Pandemic (2020-2022), and Post-Pandemic (2023-2024).
* **Machine Learning:** Temporal model generalization tests. Models trained on pre-pandemic data are tested on pandemic-era data to measure "performance drift" (proxy for structural population changes).

### RQ3: Predictor Stability
* **Objective:** Determine if risk factors (BMI, smoking, income) retain predictive power over the decade.
* **Statistical Methods:** Year-by-year logistic regressions to compare coefficient direction and magnitude.
* **Machine Learning:**
    * **Gradient Boosting (XGBoost/CatBoost) & Random Forest:** Used to capture non-linear feature interactions.
    * **SHAP (SHapley Additive exPlanations):** Used to quantify the annual marginal contribution of features to visualize stability.

### RQ4: Compounding Risk
* **Objective:** Evaluate if multiple risk factors interact synergistically rather than additively.
* **Statistical Methods:** Risk index construction (0, 1, 2, 3+ factors) and trend analysis.
* **Machine Learning:** Decision-tree-based models (XGBoost) to natively handle non-linear interaction effects.

## 6. Machine Learning Evaluation Framework
Given the class imbalance (diabetes prevalence is ~10-15%), the analysis prioritizes metrics beyond simple accuracy.

* **Imbalance Handling:** SMOTE (Synthetic Minority Over-sampling Technique) is used during training.
* **Key Metrics:**
    * **Recall (Sensitivity):** Prioritized to minimize false negatives.
    * **F1-Score:** Harmonic mean of precision and recall.
    * **AUPRC (Area Under the Precision-Recall Curve):** Primary performance metric for imbalanced classes.

## 7. Tools and Infrastructure
* **Language:** Python.
* **Environment:** Jupyter Notebooks for EDA, ingestion, and modeling.
* **Version Control:** GitHub.
    * *Repository Structure:* Organized into `data_raw` (script-generated), `data_processed` (cleaned CSVs), `analysis` (notebooks), and `reports` (figures).