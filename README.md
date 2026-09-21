# 🏥 Nexora Care Flow: Patient Demand Forecasting & Operational Staffing

> An end-to-end healthcare data science solution that predicts outpatient clinic appointment demand and dynamically optimizes Doctor and Nurse staffing allocations across regional medical centers.

---

## 📌 Executive Summary

**Nexora Health** operates 4 regional outpatient clinics:
- **Riverside Medical Center** (Clinic 1, Large)
- **Lakeside Health Hub** (Clinic 2, Medium)
- **Northside Family Clinic** (Clinic 3, Medium)
- **West End Wellness Center** (Clinic 4, Small)

Clinic operations historically struggled with a persistent resource mismatch:
1. **Overstaffing on slow days** wasted clinical payroll and left healthcare professionals idle.
2. **Understaffing during peak demand** (Monday surges and winter flu season) resulted in excessive patient wait times, clinician burnout, and compromised care quality.

**Nexora Care Flow** replaces static scheduling with an automated, data-driven machine learning pipeline. The system audits 129,000+ Electronic Health Records, engineers temporal and seasonal features, forecasts weekly patient demand using a champion Gradient Boosting model (18.55% MAPE), and converts volume predictions into actionable Doctor and Nurse Full-Time Equivalent (FTE) rosters.

---

## 🏗️ System Architecture & Production Workflow

The diagram below illustrates the end-to-end data pipeline, machine learning engine, operational staffing translation, and governance monitoring loop:

![Nexora Care Flow Production Architecture](notebooks/production_architecture_flowchart.svg)

The production architecture is organized into three distinct layers:
1. **Layer 1: Data & Feature Pipeline**:
   - Ingests raw appointment logs nightly from the Electronic Health Records (EHR) database.
   - Cleans records, standardizes dates, and aggregates data into daily (2,924 rows) and weekly (420 rows) time series.
   - Computes temporal lag features (`lag_1w`, `lag_2w`, `lag_4w`), moving averages (`rolling_mean_4w`), cyclical calendar coordinates, and seasonal event flags.
2. **Layer 2: Clinical Operations & Staffing**:
   - The champion Gradient Boosting Regressor generates a 4-week rolling volume forecast per clinic.
   - The staffing engine applies clinical capacity formulas to compute required Doctor and Nurse headcounts.
   - A weekly staffing bulletin (`staffing_guidance_holdout.csv`) is published for clinic practice managers and shift schedulers.
3. **Layer 3: Governance & Automated Drift Monitoring**:
   - Weekly error telemetry monitors forecast accuracy against actual patient visits.
   - A drift detection rule triggers an automated retraining pipeline whenever the Mean Absolute Percentage Error (MAPE) exceeds 18.0% for 3 consecutive weeks.

---

## 📖 Project Chapters & Notebook Roadmap

The repository is structured into 6 sequential, fully executed Jupyter Notebooks located in the [`notebooks/`](notebooks/) directory:

| Chapter / Notebook | Core Objective | Key Output / Metric |
| :--- | :--- | :--- |
| **[01_data_cleaning.ipynb](notebooks/01_data_cleaning.ipynb)** | Audits raw EHR records, handles missing data, and standardizes datetimes. | `cleaned_appointments.csv` (129,353 rows)<br>`weekly_clinic_appointments.csv` (420 rows) |
| **[02_EDA.ipynb](notebooks/02_EDA.ipynb)** | Discovers demand patterns, day-of-week surges, and annual flu seasonality. | Identifies +32% Monday surge, +24.5% flu spike, -45% holiday volume drop |
| **[03_feature_eng.ipynb](notebooks/03_feature_eng.ipynb)** | Engineers time-series lags, rolling trends, cyclical months, and flu flags. | `weekly_features.csv` (379 clean feature rows) |
| **[04_model_development.ipynb](notebooks/04_model_development.ipynb)** | Trains and benchmarks Naive Baseline, Random Forest, and Gradient Boosting. | **Champion GBR**: **18.55% MAPE**, **72.65 RMSE**<br>`model/weekly_forecasting_model.pkl` |
| **[05_staffing_guidance.ipynb](notebooks/05_staffing_guidance.ipynb)** | Converts volume forecasts into Doctor and Nurse FTE staffing schedules. | **70.3% Optimal Capacity** weeks<br>`data/processed/staffing_guidance_holdout.csv` |
| **[06_model_refresh_documentation.ipynb](notebooks/06_model_refresh_documentation.ipynb)** | Defines MLOps governance, retraining cadences, and drift detection rules. | Production SLAs, drift prototype, and deliverables catalog |

---

## 📊 Key Operational Discoveries (Exploratory Analysis)

Exploratory Data Analysis across 2 full operating years (Jan 2024 – Dec 2025) revealed three primary operational patterns:

1. **The Monday Surge (+32% Volume)**:
   - Mondays consistently experience 32% higher appointment volume than Fridays due to symptom buildup over the weekend.
   - *Operational Action*: Shift staffing allocations toward early-week coverage rather than flat daily staffing.
2. **Q4 Winter Flu Season (+24.5% Volume)**:
   - Weekly demand rises steadily from October through February, driven by respiratory illnesses.
   - *Operational Action*: Schedule seasonal float pool nurses and extend temporary physician hours from October to February.
3. **Holiday Dips & Sudden Rebounds (-45% to -58%)**:
   - Thanksgiving (ISO Week 47) and Christmas/New Year (ISO Week 52) experience severe drops in elective appointments, followed by sharp Week 1 rebounds.
   - *Operational Action*: Avoid over-scheduling clinicians during major holiday weeks while preparing for immediate post-holiday volume surges.

---

## 🤖 Model Development & Benchmarking Results

Models were evaluated chronologically on an unseen holdout validation set covering Q4 2025 (64 clinic-weeks across all 4 clinics during the demanding flu season):

| Model / Approach | Category | MAPE (%) | RMSE (Appointments) | Performance vs Baseline |
| :--- | :--- | :---: | :---: | :--- |
| **Naive Baseline (`lag_4w`)** | Rule-Based Heuristic | **20.73%** | **89.02** | *Legacy Status Quo* (lags seasonal transitions by 4 weeks) |
| **Random Forest Regressor** | Bagging Ensemble (100 Trees) | **19.68%** | **77.24** | **-1.05% MAPE**, **-11.78 RMSE** (-13.2% error reduction) |
| **Gradient Boosting Regressor (Champion) 🏆** | Boosting Ensemble (100 Trees) | **18.55%** | **72.65** | **-2.18% MAPE**, **-16.37 RMSE** (-18.4% error reduction) |

### Predictive Drivers (Feature Importance)
Feature importance audits of the champion Gradient Boosting model demonstrated that:
- **`lag_1w`** (immediate prior-week volume) and **`rolling_mean_4w`** (smoothed 4-week demand baseline) provide over 80% of predictive power.
- **`Is_Flu_Season`** and calendar signals (`Month_Cos`) calibrate the model for winter volume influxes.

---

## 🏥 Operational Staffing Framework

To translate volume forecasts into actionable staffing rotas, the solution applies standardized healthcare workforce capacity formulas:

### Staffing Formulas
- **Doctor FTE Requirement**:
  $$\text{Doctor FTE} = \left\lceil \frac{\text{Predicted Weekly Volume}}{40} \right\rceil$$
  *Rationale*: A full-time primary care physician comfortably handles ~40 appointments per week (8 patient visits per day across a 5-day work week).
- **Nurse FTE Requirement**:
  $$\text{Nurse FTE} = \left\lceil \text{Doctor FTE} \times 1.5 \right\rceil$$
  *Rationale*: Outpatient clinical standards require 1.5 nurses per physician for intake, triage, vitals, vaccinations, and care coordination.
- **Integer Ceiling Rounding (`np.ceil`)**:
  Clinics schedule whole clinician shifts. Rounding up prevents provider shortages and prioritizes patient safety.

### Capacity Alert Thresholds ($\pm 15\%$)
- **Optimal Capacity ($\pm 15\%$)**: Forecasted demand matches scheduled capacity within safe operational margins. Achieved in **70.3% of clinic-weeks (45 out of 64)**.
- **Overstaffed Alert ($> +15\%$)**: Forecast significantly exceeds patient arrivals (**20.3% of weeks**), indicating opportunities to reduce payroll waste or schedule administrative tasks.
- **Understaffed Alert ($< -15\%$)**: Patient demand exceeds scheduled capacity (**9.4% of weeks**), providing an advance warning to activate float pool nurses and locum physicians.

---

## 📜 Production Governance & Model Refresh Protocol

To mitigate model drift caused by demographic changes, new service lines, or epidemiological shifts, the following production governance rules are established:

1. **Weekly Scoring Cadence**: Automated forecast generation runs every Sunday at 23:00 for the upcoming 4 calendar weeks.
2. **Monthly Retraining Cadence**: Model parameters are re-fit on the 1st of every month incorporating the latest verified appointment logs.
3. **Automated Drift Detection Rule**:
   - An alert triggers immediate model retraining if the rolling forecast error exceeds an **18.0% MAPE ceiling for 3 consecutive weeks**.
   - Requiring 3 consecutive breaches ensures that temporary operational anomalies (such as blizzard-related clinic closures) do not cause false retraining alarms.

---

## 📁 Repository Structure

```
nexora_care_flow/
├── README.md                                  # Project overview and executive summary (3rd person)
├── STORYLINE.md                               # Complete data science narrative and business arc
├── data/
│   ├── raw/
│   │   └── AppointmentRecords.csv             # Raw EHR appointment logs (129,353 records)
│   └── processed/
│       ├── cleaned_appointments.csv           # Deduplicated, cleaned record-level data
│       ├── daily_clinic_appointments.csv      # Daily appointment counts per clinic (2,924 rows)
│       ├── weekly_clinic_appointments.csv     # Weekly aggregated appointment counts (420 rows)
│       ├── weekly_features.csv                # 8 engineered time-series features (379 rows)
│       ├── weekly_holdout_predictions.csv     # Model forecasts vs actuals on Q4 holdout (64 rows)
│       └── staffing_guidance_holdout.csv      # Doctor/Nurse FTE rosters and capacity alerts
├── model/
│   ├── weekly_forecasting_model.pkl           # Serialized champion GradientBoostingRegressor
│   └── evaluation_summary.json                # Benchmark metrics and feature contract
└── notebooks/
    ├── 01_data_cleaning.ipynb                 # Chapter 1: Ingestion, audit, and data hygiene
    ├── 02_EDA.ipynb                           # Chapter 2: Exploratory analysis and demand discovery
    ├── 03_feature_eng.ipynb                   # Chapter 3: Time-series feature engineering
    ├── 04_model_development.ipynb            # Chapter 4: Model training, evaluation, and benchmarking
    ├── 05_staffing_guidance.ipynb             # Chapter 5: Operational Doctor & Nurse FTE guidance
    ├── 06_model_refresh_documentation.ipynb   # Chapter 6: Governance, drift monitoring, and handover
    ├── production_architecture_flowchart.svg  # Vector architecture and governance flowchart
    └── production_architecture_flowchart.png  # High-resolution rasterized flowchart
```

---

## 🚀 Getting Started & Execution Guide

### Prerequisites
- Python 3.10+ (tested on Python 3.14)
- Standard data science libraries: `pandas`, `numpy`, `scikit-learn`, `matplotlib`, `seaborn`, `joblib`

### Installation
```bash
# Clone the repository
git clone https://github.com/earlchirchir/nexora_care_flow_forecast.git
cd nexora_care_flow_forecast

# Install required dependencies
pip install pandas numpy scikit-learn matplotlib seaborn joblib
```

### Running the Notebooks
Execute the notebooks in sequential order (`01` through `06`):
```bash
jupyter notebook notebooks/01_data_cleaning.ipynb
```
Each notebook is self-contained and pre-executed with rendered outputs, statistical tables, and visualization charts.

---

## 👥 Project Information & Acknowledgements
- **Author**: Earl Chirchir
- **Curriculum**: Amdari Data Science Practical Syllabus (Weeks 2 – 4)
- **Domain**: Healthcare Operations Management & Outpatient Clinical Workforce Analytics
- **License**: MIT License
