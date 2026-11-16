# Sewage level prediction in Brussels

**Techniques of Artificial Intelligence**

Mohammadmahdi (Sina) Khaji · Jana Heitkemper

## Introduction

Brussels operates a combined sewer system. During intense rainfall, excess water (a mixture of rain- and wastewater) is discharged untreated into the Senne at combined sewer overflow (CSO) locations. This project focuses on the **Lion overflow location (DO08)**, where overflow events occur when water levels in the collector exceed **3000 mm**.

**Research question:** *How can future water level in the sewer best be modeled using sewage and rainfall data?*

## Data

| Source | Sensor(s) | Resolution | Period | State |
|---|---|---|---|---|
| Rainfall (`Pluviometer data/`) | Avant-Port, Flagey | 5 min | 2022–2024 | cleaned |
| Sewer level (`Sewage data/`) | Lion | 1 min | 2022–2024 | raw |

## Data cleaning & preprocessing

- Shifted the sewage index by one hour to align with **UTC+1**
- Set a 5-day gap of negative water level values to NaN
- Filled small missing values by linear interpolation
- Downsampled sewage data from 1 min to 5 min using **max aggregation**, to preserve spikes important for sewer overflow
- Created targets by shifting the water level by -12 (t+60) and -3 (t+15) and removing rows with NaNs

## Feature engineering

Input features are based only on current (t) and past (<t) measurements:

- `hour`: encoded with sin & cos
- `month`: encoded with sin & cos
- `waterlevel_delta`: water level change from the previous datapoint
- Rainfall lags: 15, 30, 60 and 90 min
- Accumulated rainfall windows: 60 to 540 min

## Train–validation–test split

Time series, so `shuffle=False`:

| Set | Share | Used for |
|---|---|---|
| Train | 60% | feature importance & selection, internal cross-validation, hyperparameter tuning |
| Validation | 20% | comparing different models and selecting the best |
| Test | 20% | performance of the chosen model (t+60 and t+15) |

Feature selection was done on the training set only, to avoid data leakage.

## Models (t+60)

Three models from different families were compared against a persistence baseline:

| Model | NSE | RMSE | MAE |
|---|---|---|---|
| Baseline (Persistence) | 0.918 | 169.9 mm | 102.6 mm |
| **ANN (MLPRegressor) ✓ Selected** | **0.946** | **137.8 mm** | **75.5 mm** |
| Random Forest | 0.942 | 143.2 mm | 81.4 mm |
| Linear Regression | 0.936 | 150.3 mm | 86.4 mm |

All three models beat the baseline. The ANN was then retrained and re-tuned for **t+15**.

## Overflow classification

Overflow is not modelled separately: if the predicted water level exceeds **3000 mm**, it counts as an overflow. The dataset is heavily imbalanced (only **2.6%** of samples are overflow events), so precision, recall and F1-score are used instead of accuracy.

| Horizon | Precision (overflow) | Recall (overflow) | F1-score (overflow) |
|---|---|---|---|
| t+60 | 0.97 | 0.71 | 0.82 |
| t+15 | 0.98 | 0.95 | 0.96 |

**Key takeaway:** precision stays high in both models, while recall on overflow jumps from 0.71 to 0.95.

## References

- Barani Design. (2020). *Rain rate intensity classification.* https://www.baranidesign.com/faq-articles/2020/1/19/rain-rate-intensity-classification
- Shakil, A., Khalighi, M. A., Pudlo, P., Leclerc, C., Laplace, D., Hamon, F., & Boudonne, A. (2023). Outlier detection in non-stationary time series applied to sewer network monitoring. *Internet of Things, 21*, 100654.
- Statista. (2025). *Annual rainfall in Belgium from 2006 to 2016.* https://www.statista.com/statistics/526023/annual-rainfall-in-belgium/
