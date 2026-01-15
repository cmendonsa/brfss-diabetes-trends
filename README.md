# BRFSS Diabetes Trends Analysis (2015-2024)

This project analyzes data from the Behavioral Risk Factor Surveillance System (BRFSS) to explore trends related to diabetes and other health indicators from 2015 to 2024. It includes a data processing pipeline to download, clean, and normalize the raw BRFSS data into a unified dataset suitable for longitudinal analysis.

## Data Source

The data is sourced from the Centers for Disease Control and Prevention (CDC) BRFSS annual surveys. The raw data is downloaded in SAS Transport format (.XPT) from the official CDC website.

- **Source:** [CDC BRFSS Annual Data](https://www.cdc.gov/brfss/annual_data/annual_data.htm)

## Project Structure

The project is organized into the following directories:

- `analysis/`: Contains the Jupyter notebook used for data downloading and processing.
- `config/`: Stores configuration files that map variable names and values across different years.
- `data_codebooks/`: Includes the official BRFSS codebooks for each year, which provide detailed information about the survey questions and variables.
- `data_raw/`: A temporary directory where the raw, compressed data is downloaded.
- `data_processed/`: Contains the processed and cleaned data, with one CSV file per year and a final merged dataset.
- `analysis/1_data_download.ipynb`: The primary notebook for data processing.

## Getting Started

To reproduce the analysis, you will need Python 3 and the required packages listed in the `analysis/1_data_download.ipynb` notebook.

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/callistus-neo/brfss-diabetes-trends.git
    cd brfss-diabetes-trends
    ```

2.  **Install the required packages:**
    The notebook uses `pandas`, `numpy`, `requests`, and `tqdm`. You can install them using pip:
    ```bash
    pip install pandas numpy requests tqdm
    ```

3.  **Run the data processing notebook:**
    Open and run the `analysis/1_data_download.ipynb` notebook. This will:
    - Download the raw data for each year (2015-2024) into the `data_raw` directory.
    - Process each year's data, normalize the variables, and save them as CSV files in the `data_processed` directory.
    - Merge the yearly CSV files into a single dataset: `BRFSS_2015_2024.csv`.
    - Create a compressed version of the final dataset: `BRFSS_2015_2024.zip`.

## Variables

The following canonical variables are used in this project. The `config/VAR_MAP.json` file defines the mapping from the original BRFSS variable names to these canonical names.

| Canonical Name              | Description                                                                 |
| --------------------------- | --------------------------------------------------------------------------- |
| `HEALTH_STATUS`             | General health status.                                                      |
| `PHYSICAL_HEALTH_STATUS`    | Number of days physical health was not good in the past 30 days.            |
| `MENTAL_HEALTH_STATUS`      | Number of days mental health was not good in the past 30 days.              |
| `EXERCISE`                  | Participation in physical activity.                                         |
| `HEALTH_CARE_COVERAGE`      | Health care coverage status.                                                |
| `SMOKER`                    | Smoking status.                                                             |
| `DRINKER`                   | Alcohol consumption status.                                                 |
| `SOCIAL_DRINKER`            | Binge drinking status.                                                      |
| `HEAVY_ALCOHOL_CONSUMPTION` | Heavy alcohol consumption status.                                           |
| `HEART_ATTACK`              | History of heart attack.                                                    |
| `STROKE`                    | History of stroke.                                                          |
| `DIABETES`                  | Diabetes status.                                                            |
| `ARTHRITIS`                 | Arthritis status.                                                           |
| `MARITAL_STATUS`            | Marital status.                                                             |
| `EMPLOYMENT`                | Employment status.                                                          |
| `SEX`                       | Sex of the respondent.                                                      |
| `AGE_CATEGORIES`            | Age categories (5-year groups).                                             |
| `BMICAT`                    | Body Mass Index (BMI) categories.                                           |
| `EDUCATION_LEVEL`           | Education level.                                                            |
| `INCOME`                    | Income level.                                                               |

## Codebooks

The `data_codebooks/` directory contains the official BRFSS codebooks for each year of the study. These documents are essential for understanding the details of each variable, including the survey questions, coding, and any changes over time.

## License

This project is licensed under the GNU General Public License v3.0 - see the [LICENSE](LICENSE) file for details.