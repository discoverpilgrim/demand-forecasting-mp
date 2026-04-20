# 📊 Sales Forecasting Dashboard v2

A Streamlit-based sales forecasting tool using XGBoost with adstocked marketing spend, recursive multi-day forecasting, and scenario planning across products and channels.

---

## Features

- **Adstocked Spend Modeling** — Applies exponential decay (EWM) to marketing spend to capture carry-over effects
- **Recursive Forecasting** — Predicts day-by-day using rolling lag features, anchored to recent historical baselines to prevent drift
- **Baseline Auto-Suggest** — Automatically recommends starting discount levels per SKU × Channel using the last 30 days of available data
- **Scenario vs Baseline** — Run a "what-if" scenario with custom sale events and Shopify offers, then compare incremental impact against a clean baseline
- **Calibration** — Applies a last-30-day correction factor to align model outputs with recent actuals
- **Log-Space Confidence Bands** — 90% or 95% uncertainty bands computed in log-space for better coverage at low volumes
- **Training Metrics Export** — Saves MAE, WAPE, bias, sigma, and train/test split info to `training_metrics.txt` and `training_metrics.json`

---

## Requirements

```
streamlit
xgboost==3.1.3
pandas
numpy
plotly
openpyxl
pyarrow
```

Install all dependencies:

```bash
pip install streamlit xgboost==3.1.3 pandas numpy plotly openpyxl pyarrow
```

---

## Data Files

Place the following files in a `data/` folder in the same directory as the script:

| File | Format | Required Columns |
|---|---|---|
| `Sales_Data_2025.parquet` | Parquet | `Order_date`, `Master_Title`, `Channel`, `Units_Sold`, `Gross_Sales`, `GMV`, `source_channel` |
| `Spends_Data_2025.parquet` | Parquet | `date_start`, `Master_Title`, `Spend` |
| `offer.xlsx` | Excel | `Title`, `Offer`, `Discount` (header row auto-detected) |

> **Note:** Rows where `source_channel == "Offline"` are automatically excluded.

---

## Running the App

```bash
streamlit run forecast_app_v2.py
```

---

## Workflow

### 1. Train the Model
On first run, no model exists. Click **🚀 Train Model** in the sidebar. This trains an XGBoost regressor on log-transformed units and saves the following files:

```
units_model.pkl
feature_columns.pkl
channel_mapping.pkl
title_mapping.pkl
model_sigmas.pkl
training_metrics.json
training_metrics.txt
```

Click **🔄 Retrain Model** any time to update with fresh data or different hyperparameters.

### 2. Configure the Forecast
- Select a **Product** and **Channel**
- Set the **Forecast Period** (up to 90 days from last sales date)
- Review or override the **auto-suggested baseline discounts** for both the selected channel and Shopify
- Set a **Monthly Marketing Spend** budget

### 3. Add Scenario Events (optional)
- **Shopify Offers** — Select from the offer list in `offer.xlsx` (or enter "Other" with a manual discount) and apply to specific dates
- **Channel Sale Events** — Define a discount percentage and apply to specific dates

### 4. Generate Forecast
Click **🔮 Generate Forecast** to produce:
- Summary metrics: Total Units, Revenue, Avg Daily Units, ROI
- Incremental impact vs baseline (Δ Units, Δ Revenue)
- Interactive Plotly chart with historical actuals, 30D median trend, baseline forecast, scenario forecast, and confidence bands
- A detailed day-by-day forecast table
- A downloadable **CSV export**

---

## Sidebar Controls

| Control | Description |
|---|---|
| Adstock alpha | Decay rate for spend carry-over (higher = faster decay) |
| Max day-to-day change cap | Limits how sharply daily predictions can swing |
| Show 30D median baseline | Overlays rolling 30-day median on historical chart |
| Show confidence bands | Toggles forecast uncertainty shading |
| Band level | 90% (z=1.64) or 95% (z=1.96) |

---

## Model Details

**Algorithm:** XGBoost (`tree_method=hist`)

**Target:** `log1p(Units_Sold)`

**Key Features:**
- Discount percentage (channel + Shopify cross-channel proxy)
- Adstocked and log-saturated spend
- Lag features: 1-day and 7-day units
- Rolling averages: 7-day and 30-day
- Discount gap (channel vs Shopify)
- Calendar: cyclic sin/cos encodings for day-of-week, month, week-of-year
- Regime flags: weekend, month start/end, payday window
- Encoded channel and product IDs

**Train/Test Split:** 80/20 chronological

**Stabilization (Recursive):** Predictions are blended with lag values and anchored to a 30-day historical baseline, with a ceiling derived from the 95th percentile of recent history and a hard floor at 30% of the baseline median.

---

## Output Files

| File | Description |
|---|---|
| `training_metrics.json` | Structured metrics: MAE, WAPE, bias, sigma, dates |
| `training_metrics.txt` | Human-readable version of the same |
| `forecast_<product>_<channel>_<start>_<end>.csv` | Downloaded forecast table |
