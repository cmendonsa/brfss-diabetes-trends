# BRFSS Diabetes Risk Analysis Data Dictionary

**Dataset:** Longitudinal Predictive Analysis of Diabetes Risk (2015-2024)
**Source:** Centers for Disease Control and Prevention (CDC) Behavioral Risk Factor Surveillance System (BRFSS)
**Harmonization Strategy:** Variables have been normalized to a Common Data Model (CDM) across the 2015-2024 distinct codebooks.

## Variable Summary

| Variable Name | Classification | Data Type | Original BRFSS Variable(s) | Description |
| :--- | :--- | :--- | :--- | :--- |
| **YEAR** | Temporal | Int64 | N/A | BRFSS Survey Year (2015-2024) |
| **DIABETES** | Outcome | float64 | DIABETE3, DIABETE4 | Respondent diabetes classification status |
| **AGE_CATEGORIES** | Demographics | float64 | AGEG5YR | Reported age in five-year age categories |
| **SEX** | Demographics | float64 | SEX, SEX1, BIRTHSEX, SEXVAR | Sex at birth |
| **RACE** | Demographics | float64 | _RACE, _RACE1 | Race-Ethnicity grouping |
| **INCOME** | Demographics | float64 | _INCOMG, _INCOMG1 | Annual household income categories |
| **EDUCATION_LEVEL** | Demographics | float64 | _EDUCAG | Highest level of education completed |
| **EMPLOYMENT** | Demographics | float64 | EMPLOY1 | Current employment status |
| **MARITAL_STATUS** | Demographics | Float64 | MARITAL | Current marital status |
| **HEALTH_CARE_COVERAGE**| Demographics | float64 | _HCVU651, _HCVU652, _HCVU653, _HCVU654 | Whether respondent has health care coverage |
| **BMICAT** | Biometrics | Float64 | _BMI5CAT | Body Mass Index (BMI) category |
| **HEART_ATTACK** | Chronic Conditions | float64 | _MICHD | History of Coronary Heart Disease (CHD) or Myocardial Infarction (MI) |
| **STROKE** | Chronic Conditions | Float64 | CVDSTRK3 | History of Stroke |
| **EXERCISE** | Behaviors | float64 | EXERANY2 | Participation in leisure-time physical activity |
| **SMOKER** | Behaviors | Float64 | _SMOKER3 | Smoking status |
| **HEAVY_ALCOHOL_CONSUMPTION** | Behaviors | float64 | _RFDRHV5, _RFDRHV6, _RFDRHV7, _RFDRHV8, _RFDRHV9 | Heavy alcohol consumption status |
| **HEALTH_STATUS** | Health Status | Float64 | _RFHLTH | Overall self-rated health status |
| **POOR_PHYSICAL_HEALTH_DAYS** | Health Status | float64 | PHYSHLTH, _PHYS14D | Number of days physical health was not good |
| **POOR_MENTAL_HEALTH_DAYS** | Health Status | float64 | MENTHLTH, _MENT14D | Number of days mental health was not good |

---

## Variable Details and Value Encodings

### 1. YEAR
* **Classification:** Temporal
* **Type:** Int64
* **Description:** The annual cycle of the BRFSS survey.
* **Range:** 2015 to 2024

### 2. DIABETES
* **Classification:** Outcome
* **Type:** float64
* **Original Variable:** `DIABETE3`, `DIABETE4`
* **Values:**
    * `1`: Diabetes
    * `2`: Gestational Diabetes
    * `3`: No Diabetes
    * `4`: Pre-diabetes
    * `7`: Not sure
    * `9`: Refused

### 3. AGE_CATEGORIES
* **Classification:** Demographics
* **Type:** float64
* **Original Variable:** `AGEG5YR`
* **Values:**
    * `1`: 18-24
    * `2`: 25-29
    * `3`: 30-34
    * `4`: 35-39
    * `5`: 40-44
    * `6`: 45-49
    * `7`: 50-54
    * `8`: 55-59
    * `9`: 60-64
    * `10`: 65-69
    * `11`: 70-74
    * `12`: 75-79
    * `13`: 80+
    * `14`: Refused

### 4. SEX
* **Classification:** Demographics
* **Type:** float64
* **Original Variable:** `SEX`, `SEX1`, `BIRTHSEX`, `SEXVAR`
* **Values:**
    * `1`: Male
    * `2`: Female
    * `7`: Not Sure
    * `9`: Refused

### 5. RACE
* **Classification:** Demographics
* **Type:** float64
* **Original Variable:** `_RACE`, `_RACE1`
* **Values:**
    * `1`: White
    * `2`: Black
    * `3`: American Indian
    * `4`: Asian
    * `5`: Hawaiian
    * `6`: Other
    * `7`: Non-Hispanic
    * `8`: Hispanic
    * `9`: Refused

### 6. INCOME
* **Classification:** Demographics
* **Type:** float64
* **Original Variable:** `_INCOMG`, `_INCOMG1`
* **Values:**
    * `1`: Less than $15,000
    * `2`: $15,000 to < $25,000
    * `3`: $25,000 to < $35,000
    * `4`: $35,000 to < $50,000
    * `5`: More than $50,000
    * `9`: Not sure

### 7. EDUCATION_LEVEL
* **Classification:** Demographics
* **Type:** float64
* **Original Variable:** `_EDUCAG`
* **Values:**
    * `1`: Dropout
    * `2`: High School Graduate
    * `3`: Undergraduate
    * `4`: Graduate
    * `9`: Not sure

### 8. EMPLOYMENT
* **Classification:** Demographics
* **Type:** float64
* **Original Variable:** `EMPLOY1`
* **Values:**
    * `1`: Wages
    * `2`: Self-employed
    * `3`: Long-term unemployed
    * `4`: Short-term unemployed
    * `5`: Homemaker
    * `6`: Student
    * `7`: Retired
    * `8`: Unable to work
    * `9`: Refused

### 9. MARITAL_STATUS
* **Classification:** Demographics
* **Type:** Float64
* **Original Variable:** `MARITAL`
* **Values:**
    * `1`: Married
    * `2`: Divorced
    * `3`: Widowed
    * `4`: Separated
    * `5`: Single
    * `6`: Partner
    * `9`: Refused

### 10. HEALTH_CARE_COVERAGE
* **Classification:** Demographics
* **Type:** float64
* **Original Variable:** `_HCVU651`, `_HCVU652`, `_HCVU653`, `_HCVU654`
* **Values:**
    * `1`: Yes
    * `2`: No
    * `9`: Refused

### 11. BMICAT
* **Classification:** Biometrics
* **Type:** Float64
* **Original Variable:** `BMI5CAT`
* **Values:**
    * `1`: Underweight
    * `2`: Normal
    * `3`: Overweight
    * `4`: Obese

### 12. HEART_ATTACK
* **Classification:** Chronic Conditions
* **Type:** float64
* **Original Variable:** `_MICHD`
* **Values:**
    * `1`: Yes
    * `2`: No

### 13. STROKE
* **Classification:** Chronic Conditions
* **Type:** Float64
* **Original Variable:** `CVDSTRK3`
* **Values:**
    * `1`: Yes
    * `2`: No
    * `7`: Not Sure
    * `9`: Refused

### 14. EXERCISE
* **Classification:** Behaviors
* **Type:** float64
* **Original Variable:** `EXERANY2`
* **Values:**
    * `1`: Yes
    * `2`: No
    * `9`: Refused

### 15. SMOKER
* **Classification:** Behaviors
* **Type:** Float64
* **Original Variable:** `_SMOKER3`
* **Values:**
    * `1`: Everyday
    * `2`: Somedays
    * `3`: Former
    * `4`: Never
    * `9`: Refused

### 16. HEAVY_ALCOHOL_CONSUMPTION
* **Classification:** Behaviors
* **Type:** float64
* **Original Variable:** `_RFDRHV5`, `_RFDRHV6`, `_RFDRHV7`, `_RFDRHV8`, `_RFDRHV9`
* **Values:**
    * `1`: No
    * `2`: Yes
    * `9`: Refused

### 17. HEALTH_STATUS
* **Classification:** Health Status
* **Type:** Float64
* **Original Variable:** `RFHLTH`
* **Values:**
    * `1`: Good
    * `2`: Poor
    * `9`: Refused

### 18. POOR_PHYSICAL_HEALTH_DAYS
* **Classification:** Health Status
* **Type:** float64
* **Original Variable:** `PHYSHLTH`, `_PHYS14D`
* **Values:**
    * `1`: Zero days when physical health not good
    * `2`: 1-13 days when physical health not good
    * `3`: 14+ days when physical health not good
    * `9`: Refused

### 19. POOR_MENTAL_HEALTH_DAYS
* **Classification:** Health Status
* **Type:** float64
* **Original Variable:** `MENTHLTH`, `_MENT14D`
* **Values:**
    * `1`: Zero days when mental health not good
    * `2`: 1-13 days when mental health not good
    * `3`: 14+ days when mental health not good
    * `9`: Refused

---

## Variable Details and Value Harmonizing Encodings (Recoded)

### 1. YEAR
* **Classification:** Temporal
* **Type:** Int64
* **Description:** The annual cycle of the BRFSS survey.
* **Range:** 2015 to 2024

### 2. DIABETES
* **Classification:** Outcome
* **Type:** float64
* **Values:**
    * `0`: No Diabetes
    * `1`: Gestational
    * `2`: PreDiabetes
    * `3`: Diabetes

### 3. AGE_CATEGORIES
* **Classification:** Demographics
* **Type:** float64
* **Values:**
    * `0`: 18-24
    * `1`: 25-29
    * `2`: 30-34
    * `3`: 35-39
    * `4`: 40-44
    * `5`: 45-49
    * `6`: 50-54
    * `7`: 55-59
    * `8`: 60-64
    * `9`: 65-69
    * `10`: 70-74
    * `11`: 75-79
    * `12`: 80+

### 4. SEX
* **Classification:** Demographics
* **Type:** float64
* **Values:**
    * `0`: Female
    * `1`: Male

### 5. RACE
* **Classification:** Demographics
* **Type:** float64
* **Values:**
    * `0`: White
    * `1`: Black
    * `2`: American Indian
    * `3`: Asian
    * `4`: Hawaiian
    * `5`: Other
    * `6`: Multi-racial
    * `7`: Hispanic

### 6. INCOME
* **Classification:** Demographics
* **Type:** float64
* **Values:**
    * `0`: Less than $15,000
    * `1`: $15,000 to < $25,000
    * `2`: $25,000 to < $35,000
    * `3`: $35,000 to < $50,000
    * `4`: More than $50,000

### 7. EDUCATION_LEVEL
* **Classification:** Demographics
* **Type:** float64
* **Values:**
    * `0`: Dropout
    * `1`: High School Graduate
    * `2`: Undergraduate
    * `3`: Graduate

### 8. EMPLOYMENT
* **Classification:** Demographics
* **Type:** float64
* **Values:**
    * `0`: Long-term unemployed
    * `1`: Short-term unemployed
    * `2`: Unable to work
    * `3`: Homemaker
    * `4`: Student
    * `5`: Retired
    * `6`: Self-employed
    * `7`: Wages

### 9. MARITAL_STATUS
* **Classification:** Demographics
* **Type:** Float64
* **Values:**
    * `0`: Single
    * `1`: Separated
    * `2`: Divorced
    * `3`: Widowed
    * `4`: Partner
    * `5`: Married

### 10. HEALTH_CARE_COVERAGE
* **Classification:** Demographics
* **Type:** float64
* **Values:**
    * `0`: No
    * `1`: Yes

### 11. BMICAT
* **Classification:** Biometrics
* **Type:** Float64
* **Values:**
    * `0`: Underweight
    * `1`: Normal
    * `2`: Overweight
    * `3`: Obese

### 12. HEART_ATTACK
* **Classification:** Chronic Conditions
* **Type:** float64
* **Values:**
    * `0`: No
    * `1`: Yes

### 13. STROKE
* **Classification:** Chronic Conditions
* **Type:** Float64
* **Values:**
    * `0`: No
    * `1`: Yes

### 14. EXERCISE
* **Classification:** Behaviors
* **Type:** float64
* **Values:**
    * `0`: No
    * `1`: Yes

### 15. SMOKER
* **Classification:** Behaviors
* **Type:** Float64
* **Values:**
    * `0`: Never
    * `1`: Former
    * `2`: Somedays
    * `3`: Everyday

### 16. HEAVY_ALCOHOL_CONSUMPTION
* **Classification:** Behaviors
* **Type:** float64
* **Values:**
    * `0`: No
    * `1`: Yes

### 17. HEALTH_STATUS
* **Classification:** Health Status
* **Type:** Float64
* **Values:**
    * `0`: Poor
    * `1`: Good

### 18. POOR_PHYSICAL_HEALTH_DAYS
* **Classification:** Health Status
* **Type:** float64
* **Values:**
    * `0`: Zero days when physical health not good
    * `1`: 1-13 days when physical health not good
    * `2`: 14+ days when physical health not good

### 19. POOR_MENTAL_HEALTH_DAYS
* **Classification:** Health Status
* **Type:** float64
* **Values:**
    * `0`: Zero days when mental health not good
    * `1`: 1-13 days when mental health not good
    * `2`: 14+ days when mental health not good
