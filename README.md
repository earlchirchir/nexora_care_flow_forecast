# Nexora Care Flow

Outpatient Appointment Demand Forecasting and Clinical Staffing Allocation

## Overview

Nexora Care Flow is a time-series forecasting and workforce planning system built for outpatient healthcare networks. Operating across four regional facilities (Riverside, Lakeside, Northside, and West End), the project addresses common inefficiencies in clinical scheduling:

- Overstaffing during low-demand periods, which increases operating costs.
- Understaffing during peak volume periods (such as Mondays and winter flu season), which leads to extended patient wait times and staff burnout.

The system processes 129,000+ Electronic Health Records (EHR) from 2024 through 2025, builds predictive temporal features, trains gradient boosted forecasting models, and maps weekly appointment projections to required Doctor and Nurse Full-Time Equivalents (FTEs).

## System Architecture

The end-to-end pipeline consists of three core operational layers: data ingestion and transformation, predictive modeling, and operational staffing with governance monitoring.

1. Data and Feature Layer:
   - Daily extraction and cleaning of appointment records from the central EHR database.
   - Aggregation into clinic-level daily (2,924 rows) and weekly (420 rows) time series.
   - Construction of lag terms (1, 2, and 4 weeks), 4-week rolling averages, cyclical calendar signals (sine/cosine of month), and binary flags for flu season and holidays.

2. Modeling and Operations Layer:
   - Inference via a trained Gradient Boosting Regressor predicting 4 weeks ahead.
   - Conversion of volume forecasts into Doctor FTEs (40 visits per week capacity) and Nurse FTEs (1.5 ratio per doctor).
   - Generation of weekly scheduling guidance tables with operational buffer alerts (+/-15% variance).

3. Governance and Retraining Layer:
   - Continuous comparison of weekly forecasts against verified patient volume.
   - Automated model retraining triggered when Mean Absolute Percentage Error (MAPE) exceeds 18.0% across three consecutive weeks.

## Workflow and Notebooks

The analysis is documented across six sequential notebooks in the `notebooks/` directory:

| Notebook | Focus | Primary Outputs |
| :--- | :--- | :--- |
| [01_data_cleaning.ipynb](notebooks/01_data_cleaning.ipynb) | Data audit, missing value imputation, record deduplication | `cleaned_appointments.csv`<br>`weekly_clinic_appointments.csv` |
| [02_EDA.ipynb](notebooks/02_EDA.ipynb) | Day-of-week patterns, seasonality, and volume distributions | Quantified Monday surges, flu season curves, and holiday dips |
| [03_feature_eng.ipynb](notebooks/03_feature_eng.ipynb) | Feature generation (lags, rolling stats, cyclical encoding) | `weekly_features.csv` (379 clean observations) |
| [04_model_development.ipynb](notebooks/04_model_development.ipynb) | Chronological train/test split, model benchmarking, evaluation | Trained model artifact (`weekly_forecasting_model.pkl`)<br>Evaluation scorecard (`evaluation_summary.json`) |
| [05_staffing_guidance.ipynb](notebooks/05_staffing_guidance.ipynb) | Translation of demand forecasts to Doctor and Nurse FTEs | `staffing_guidance_holdout.csv`<br>Capacity alert classifications |
| [06_model_refresh_documentation.ipynb](notebooks/06_model_refresh_documentation.ipynb) | Governance protocols, retraining triggers, architecture mapping | Production runbook and drift detection implementation |

## Key Findings from Exploratory Analysis

Analysis of appointment patterns over the 2024–2025 period highlighted three operational factors:

1. Monday Demand Spike: Mondays average 32% higher appointment volume than Fridays. Scheduling practices should allocate additional clinical hours early in the week rather than maintaining equal daily coverage.
2. Winter Seasonality: Total appointments rise by approximately 24.5% between October and February each year due to seasonal respiratory infections.
3. Holiday Reductions: Weeks containing Thanksgiving (Week 47) and Christmas (Week 52) show volume reductions of 40% to 50%, followed by an immediate rebound in early January.

## Model Benchmarking and Evaluation

Models were evaluated using a chronological holdout validation set covering Q4 2025 (64 clinic-weeks across all four facilities). Shuffling was deliberately avoided to eliminate temporal data leakage.

| Model | Approach | MAPE (%) | RMSE (Visits) | Assessment |
| :--- | :--- | :---: | :---: | :--- |
| Naive Baseline (`lag_4w`) | Same-week-last-month heuristic | 20.73% | 89.02 | Benchmark reflecting manual scheduling practices |
| Random Forest | Bagging ensemble (100 trees) | 19.68% | 77.24 | Reduced variance, moderate fit on turning points |
| Gradient Boosting | Sequential boosting (100 trees) | 18.55% | 72.65 | Selected champion; lowest overall error on seasonal swings |

Feature importance analysis indicates that the immediate prior week volume (`lag_1w`) and the 4-week smoothed moving average (`rolling_mean_4w`) contribute over 80% of total predictive signal, with the seasonal flu indicator providing necessary adjustments during Q4 spikes.

## Staffing Allocation Logic

Predicted appointment numbers are mapped to staffing requirements using standard primary care operational ratios:

- Doctor Staffing:
  $$\text{Doctor FTE} = \left\lceil \frac{\text{Forecasted Weekly Volume}}{40} \right\rceil$$
  Each full-time physician is budgeted for approximately 40 appointments per week (8 appointments per day across a 5-day schedule).
- Nurse Staffing:
  $$\text{Nurse FTE} = \left\lceil \text{Doctor FTE} \times 1.5 \right\rceil$$
  Assigned at 1.5 nurses per practicing physician for intake, triage, vaccination support, and post-visit documentation.
- Rounding: The ceiling function is applied to allocate whole provider shifts, avoiding fractional scheduling that could compromise clinic floor coverage.

### Capacity Status Classification

Weekly clinic schedules are classified based on the percentage deviation between forecasted and actual volume:

- Optimal Capacity (+/-15% variance): Achieved in 70.3% of holdout clinic-weeks (45 of 64). Demand and staffing remain balanced within safe operational tolerances.
- Overstaffed Alert (> +15% forecast error): Observed in 20.3% of holdout weeks, predominantly during holiday dips. Provides an advance signal to adjust temporary provider hours.
- Understaffed Alert (< -15% forecast error): Observed in 9.4% of holdout weeks, concentrated during rapid flu surge weeks. Serves as a prompt to mobilize float pool personnel.

## Production Governance and Model Monitoring

The system operates under a defined maintenance protocol:

- Scoring Schedule: The forecasting pipeline executes every Sunday at 23:00 to generate volume and staffing projections for the upcoming 4-week planning window.
- Retraining Schedule: Scheduled on the first calendar day of each month using all verified historical data up to that date.
- Drift Detection Rule: Model errors are tracked on a rolling basis. If holdout MAPE exceeds 18.0% for three consecutive weeks, an automated retraining workflow is triggered. Single-week anomalies (such as weather disruptions) are ignored to avoid unnecessary model churn.

## Repository Structure

```
nexora_care_flow/
├── README.md
├── STORYLINE.md
├── data/
│   ├── raw/
│   │   └── AppointmentRecords.csv
│   └── processed/
│       ├── cleaned_appointments.csv
│       ├── daily_clinic_appointments.csv
│       ├── weekly_clinic_appointments.csv
│       ├── weekly_features.csv
│       ├── weekly_holdout_predictions.csv
│       └── staffing_guidance_holdout.csv
├── model/
│   ├── weekly_forecasting_model.pkl
│   └── evaluation_summary.json
└── notebooks/
    ├── 01_data_cleaning.ipynb
    ├── 02_EDA.ipynb
    ├── 03_feature_eng.ipynb
    ├── 04_model_development.ipynb
    ├── 05_staffing_guidance.ipynb
    ├── 06_model_refresh_documentation.ipynb
    ├── production_architecture_flowchart.svg
    └── production_architecture_flowchart.png
```

## Setup and Dependencies

### Environment Requirements

- Python 3.10 or higher
- Required packages: `pandas`, `numpy`, `scikit-learn`, `matplotlib`, `seaborn`, `joblib`

### Installation

```bash
git clone https://github.com/earlchirchir/nexora_care_flow_forecast.git
cd nexora_care_flow_forecast
pip install pandas numpy scikit-learn matplotlib seaborn joblib
```

### Execution

Notebooks can be run sequentially via Jupyter:

```bash
jupyter notebook notebooks/01_data_cleaning.ipynb
```
