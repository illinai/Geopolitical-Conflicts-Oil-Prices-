# Geopolitical Conflict and Oil Stock Performance

**Project — Group 26**  
Sana Shah (94945664), Maya Knutsvig (17950502), Illina Islam (58189903)

---

## Overview

This project analyzes whether patterns in oil prices and stock market behavior observed during the **First Gulf War (1990–1991)** can be used to predict market responses during the **Second Gulf War (2003)** for 10 oil and gas companies. Based on these findings, the project explores whether similar patterns can provide insight into potential market reactions during current **U.S.–Iran geopolitical tensions**.

[Take a look!](https://gulf-war-oil-stocks.streamlit.app) — hosted for free on Streamlit Community Cloud (the app sleeps after about 12 hours without visitors; the first load afterwards takes under a minute).

---

## Companies Tracked

| Region | Ticker | Company |
| --- | --- | --- |
| Canada | SU | Suncor Energy |
| Canada | CNQ | Canadian Natural Resources |
| Canada | CVE* | Cenovus Energy |
| US | XOM | ExxonMobil |
| US | CVX | Chevron |
| US | COP | ConocoPhillips |
| International | SHEL | Shell |
| International | TTE | TotalEnergies |
| International | BP | BP |
| International | 2222.SR* | Saudi Aramco |

*CVE (founded 2009) and 2222.SR (IPO 2019) lack data for Gulf War periods and are included only in the current U.S.–Iran analysis. CNQ has no First Gulf War data. The remaining 7 companies (BP, COP, CVX, SHEL, SU, TTE, XOM) cover all three conflict periods and are used for the full historical comparison.

---

## Research Questions

1. To what extent can oil price and oil & gas stock patterns observed during the First Gulf War predict market behavior during the Second Gulf War?
2. How do oil prices and stock markets respond before, during, and after major geopolitical conflict events?
3. Can patterns learned from historical conflicts provide insight into potential market behavior during the current U.S.–Iran tensions?

---

## Datasets

| Source | Description |
| --- | --- |
| [yfinance](https://pypi.org/project/yfinance/) | Historical stock prices for all 10 companies |
| [EIA](https://www.eia.gov/dnav/pet/pet_pri_spt_s1_d.htm) | Daily spot oil prices |
| [GDELT](https://www.gdeltproject.org) | Geopolitical event data |

---

## Pipeline

```text
01_data_acquisition.py   →   Fetch stock + oil price data via yfinance & EIA API
02_etl_cleaning.py       →   Clean, standardize, engineer features (returns, volatility, MAs)
03_eda.py                →   Exploratory analysis across conflict periods
04_modeling.py           →   Train on Gulf War I, test on Gulf War II
05_live_predictions.py   →   Apply model to live U.S.–Iran data
06_export_tableau.py     →   Export data for dashboards
app/app.py               →   Streamlit dashboard
```

### Stage Details

**Data Acquisition** — Collect historical stock and oil price data using Python (yfinance). Define conflict periods: First Gulf War (1990–1991), Second Gulf War (2003). Store raw data locally in `data/raw/` (re-create it any time by running the script).

**ETL / Cleaning** — Handle missing values and inconsistent dates. Standardize time-series data. Calculate derived features: daily returns, moving averages, volatility. Document all variables in `data_dictionary.md`. Load cleaned datasets into PostgreSQL (Neon) with tables: `stock_data`, `oil_prices`, `conflict_events`, `model_results`. Cleaned CSVs are also kept in `data/cleaned/` as the app's fallback data source.

**EDA** — Analyze oil price trends during conflict periods. Compare stock performance before, during, and after conflicts. Identify volatility spikes and recovery patterns.

**Modeling** — Train a Logistic Regression baseline and a Random Forest Classifier on First Gulf War data. Test on Second Gulf War data, predicting oil price movement direction and stock direction. Evaluate generalizability across conflict periods.

**Live Predictions (Extension)** — Pull live oil price and stock data via EIA API and yfinance. Apply the trained model to current U.S.–Iran geopolitical conditions to generate real-time directional predictions for all 10 companies.

---

## Tools & Technologies

| Tool | Purpose |
| --- | --- |
| Python (pandas, numpy, scikit-learn, yfinance) | Data acquisition, ETL, EDA, modeling |
| PostgreSQL (Neon, free tier) | Structured storage of cleaned datasets |
| Git + CSV files | Raw and processed dataset storage (cleaned CSVs are kept in `data/cleaned/` as the app's fallback) |
| Streamlit | Interactive web dashboard |
| Tableau | Reporting dashboard — conflict period comparisons and model predictions vs. actuals |
| Streamlit Community Cloud | Showcase/Deployment (free) |

---

## Hosting & Data (free setup)

The dashboard runs on **Streamlit Community Cloud** and reads its data from a **Neon** Postgres database. Both are free tiers, so keeping the project online costs nothing.

| Layer | Original deployment (AWS) | Current (free) |
| --- | --- | --- |
| App hosting | EC2 instance running Streamlit | Streamlit Community Cloud, deployed from GitHub |
| Database | RDS PostgreSQL | Neon Postgres (free plan) |
| File storage | S3 bucket | Files in git (`data/cleaned/`); raw data is regenerated by `01_data_acquisition.py` |
| Credentials | `.env` with `DB_HOST`, `DB_USER`, AWS keys, SSL certificate | one `DATABASE_URL` (Streamlit secret, or `.env` locally) |

### Why we migrated

AWS's free plan is time-limited, and our RDS instance had stopped accepting connections (its status showed "Inaccessible-encryption-credentials"). We moved to services with permanent free tiers so the project stays available without a paid account.

### What changed

- `app/app.py` queries Neon through `DATABASE_URL` using a short-lived connection per query (Neon suspends idle databases, so a long-lived connection would go stale). If the database is unreachable or not configured, it falls back to the CSVs in `data/cleaned/`. The sidebar shows which source is in use.
- All pipeline scripts now share one connection helper, `scripts/db_config.py`. S3 uploads in scripts 01 and 02 are optional and off unless AWS keys are set.
- `requirements.txt` is pinned to the versions the app was built with (`requirements-pipeline.txt` adds the packages only the scripts need).
- The cleaned CSVs the app needs are now committed to the repo (about 4.5 MB).

### How the data was moved

Because the RDS instance could not be reached, the Neon tables were rebuilt from the cleaned CSVs: `stock_data` (29,759 rows), `oil_prices` (9,366), `conflict_events` (16). The `model_results` table was regenerated by re-running `scripts/04_modeling.py` on that data.

One data fix came out of this: the cleaned oil file had been cut off at exactly 5,000 rows (ending 2008-10-30) even though the raw EIA file runs to March 2026. It was rebuilt from the raw file using the same logic as `scripts/fix_oil_data.py`. The first 5,000 rows are unchanged, so earlier headline oil statistics (for example the roughly +48% First Gulf War and +3.2% Second Gulf War changes in WTI) are unaffected. With the full series, the Gulf Wars to U.S.–Iran experiment runs again.

| Experiment | Model | Accuracy | F1 |
| --- | --- | --- | --- |
| Gulf War 1 → Gulf War 2 | Logistic Regression | 0.583 | 0.534 |
| Gulf War 1 → Gulf War 2 | Random Forest | 0.578 | 0.524 |
| Gulf Wars → U.S.–Iran | Logistic Regression | 0.637 | 0.614 |
| Gulf Wars → U.S.–Iran | Random Forest | 0.630 | 0.604 |

### Running it yourself

```bash
# run the dashboard locally (uses the CSVs unless DATABASE_URL is set)
pip install -r requirements.txt
streamlit run app/app.py

# optional: use a database
cp .streamlit/secrets.toml.example .streamlit/secrets.toml   # then set DATABASE_URL
# or put DATABASE_URL=postgresql://user:password@host/dbname?sslmode=require in a .env file

# run the pipeline scripts
pip install -r requirements-pipeline.txt
python3 scripts/01_data_acquisition.py   # ...through 06_export_tableau.py
```

To deploy your own copy: fork the repo, create a free Neon project, then on [share.streamlit.io](https://share.streamlit.io) deploy `app/app.py` (Python 3.12) and add `DATABASE_URL` under Advanced settings, Secrets. Never commit `.env` or `secrets.toml`.

---

## Project Structure

```text
.
├── scripts/
│   ├── 01_data_acquisition.py
│   ├── 02_etl_cleaning.py
│   ├── 03_eda.py
│   ├── 04_modeling.py
│   ├── 05_live_predictions.py
│   ├── 06_export_tableau.py
│   ├── fix_oil_data.py
│   └── db_config.py              # shared Neon/Postgres connection (DATABASE_URL)
├── app/
│   └── app.py                    # Streamlit dashboard
├── data/
│   └── cleaned/                  # small CSVs the app falls back to (stocks, oil, events, model results)
├── models/
├── reports/
├── data_dictionary.md
├── requirements.txt              # app dependencies (used by Streamlit Community Cloud)
├── requirements-pipeline.txt     # extra packages for running the scripts
└── .streamlit/
    ├── config.toml
    └── secrets.toml.example      # copy to secrets.toml (git-ignored) for local runs
```

---

## Risks & Limitations

- **Survivorship/availability bias** — CVE and 2222.SR did not exist during the First Gulf War and are excluded from that portion of the analysis.
- **Causation vs. correlation** — Market movements during conflict periods may be driven by many simultaneous factors beyond geopolitical events (e.g., recessions, interest rates).
- **Generalizability** — Patterns from 1990–2003 may not reliably predict modern markets due to algorithmic trading, ETFs, and structural changes in energy markets.
- **Data licensing** — yfinance is for educational/personal use. EIA data is public domain. Datasets used under their respective licenses.
- **No financial advice** — This project is purely academic and should not be interpreted as investment guidance.
