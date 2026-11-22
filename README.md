# ⚡ German Day-Ahead Power Price Forecast

Forecasting hourly **day-ahead electricity prices for the Germany/Luxembourg bidding zone** using only free public data from [SMARD.de](https://www.smard.de) (Bundesnetzagentur).

The goal of this project is not just a model, but to understand **what drives German power prices** and to build a forecast that beats a strong naive benchmark, using only information a trader actually has before the day-ahead auction closes.

---

## 📊 Results

Test period: **August 2026 – October 2026** (≈3 months, hourly, out-of-sample)

| Model | Approach | MAE (€/MWh) |
|---|---|---|
| Baseline | Price of the same hour yesterday | 40.32 |
| v1 | LightGBM predicting the price directly | 53.82 ❌ |
| v2 | LightGBM predicting deviation from 7-day average | 46.45 ❌ |
| **v3** | **LightGBM predicting the change vs. yesterday** | **26.98 ✅** |

➡️ **v3 reduces the error by ~33 % compared to the naive baseline.**

---

## 🔍 The Story: Why v1 and v2 Failed

**v1 – Overfitting and price-level shift**
- Train MAE was only 7.6 €/MWh, but test MAE was 53.8 €/MWh, which means the model clearly overfit.
- Plotting predictions vs. actuals showed the model captured the daily *shape* correctly but was **systematically too low**.
- Reason: the overall price level in 2026 was higher than in the training period (driven by fuel and CO₂ costs, which are not in the data). Tree-based models cannot extrapolate beyond the value range they were trained on. The `month` feature also anchored the model to last year's price levels.

**v2 – Removing the level**
- Predicting the deviation from a rolling 7-day average and simplifying the model helped (53.8 → 46.5), but it still did not beat the baseline.

**v3 – Learning the *correction* to yesterday's price**
- In power markets, yesterday's price is a very strong predictor. Instead of predicting the price from scratch, v3 starts from yesterday's price and learns **how much it should change**.
- Key features are the **day-over-day changes** in forecasted wind, solar, load and **residual load** (load minus wind and solar).
- Intuition: *"Tomorrow there is 10 GW more wind than today → residual load drops → price drops."*

---

## 🧠 Key Insight: Residual Load

**Residual load = grid load − wind − solar**

It describes how much demand must be covered by conventional (mostly gas and coal) power plants, which set the marginal price according to the **merit order**.
In this dataset, the correlation between residual load and price is **0.75**.

---

## ✅ No Look-Ahead Bias

The German day-ahead auction (EPEX Spot) closes at **12:00 on D-1**. At that time, a trader knows:
- day-ahead prices for day D-1 (set in the previous auction)
- day-ahead **forecasts** of wind, solar and load for day D

The model uses **only forecasted** generation and consumption (not actual values) plus lagged prices. Train/test are split **by time**, not randomly.

---

## 🗂️ Data

Source: SMARD.de – Download center, hourly resolution, 01.11.2024 – 01.11.2026 (~17,500 hours)

| File | Used columns |
|---|---|
| Day-ahead prices | Germany/Luxembourg [€/MWh] |
| Forecasted generation (day-ahead) | Wind onshore, Wind offshore, Photovoltaics [MWh] |
| Forecasted consumption | Grid load, Residual load [MWh] |

---

## ⚙️ Features (v3)

| Feature | Description |
|---|---|
| `d_wind_on`, `d_wind_off`, `d_solar` | Change in forecasted generation vs. same hour yesterday |
| `d_load`, `d_residual_load` | Change in forecasted load / residual load vs. yesterday |
| `residual_load` | Absolute residual load level |
| `hour`, `weekday` | Calendar effects |
| `price_lag_24` | Price of the same hour yesterday |

Target: `price − price_lag_24` → prediction = `price_lag_24 + model output`

---

## 🚀 How to Run

```bash
pip install pandas matplotlib lightgbm scikit-learn ipykernel
```

1. Download the three CSV files from [SMARD Download Center](https://www.smard.de/en/downloadcenter/download-market-data) (Country: Germany, Resolution: Hour, File type: CSV, English format)
2. Put them into the `data/` folder
3. Open `explore.ipynb` and run all cells

```
power-price-forecast/
├── data/
│   ├── Day-ahead_prices_....csv
│   ├── Forecasted_generation_Day-Ahead_....csv
│   └── Forecasted_consumption_....csv
├── explore.ipynb
└── README.md
```

---

## ⚠️ Limitations & Next Steps

- **Single test window** (3 months). Next: rolling / walk-forward validation across all seasons.
- **No fuel prices.** Adding TTF gas and EU ETS CO₂ prices should explain the price level shift seen in v1.
- **No cross-border flows** or neighbouring market prices.
- **Negative prices and extreme spikes** are still hard to predict. Next: separate evaluation for these hours.
- **15-minute products:** the day-ahead market moved to 15-minute resolution, and the model could be extended accordingly.
- Hyperparameters were not tuned.
- **Automation:** daily data pull and next-day forecast via an n8n workflow.

---

## 🛠️ Tech Stack

Python · pandas · LightGBM · scikit-learn · matplotlib · Jupyter

---

## 👤 Author

**Reza Barakati**
M.Sc. Digital Technologies (Ostfalia University of Applied Sciences)
Interested in energy trading, data analysis and forecasting.
