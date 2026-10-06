# Base Station Traffic Spike Prediction — Shanghai Telecom

Predicting, one hour ahead, whether a mobile base station will experience a traffic spike, using the Shanghai Telecom base station access dataset (June – November 2014).

## Results (test set: 3 Oct – 30 Nov 2014)

| Model | PR-AUC | ROC-AUC | Precision | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Persistence (spike now → spike next hour) | 0.136 | 0.667 | 0.349 | 0.349 | 0.349 |
| Logistic regression | 0.402 | 0.940 | 0.407 | 0.425 | 0.416 |
| **LightGBM** | **0.467** | **0.949** | **0.460** | **0.434** | **0.447** |

Spikes are 2.1% of test station-hours, so PR-AUC is the primary metric (random = 0.021). Accuracy is not used: predicting "no spike" everywhere already scores 97.9%.

## Dataset

Each record is one user session at a base station.

| Column | Description |
|---|---|
| `month` | Logging month (`YYYYMM`) |
| `start_time` | Session start (date-time) |
| `end_time` | Session end (date-time) |
| `latitude`, `longitude` | Base station coordinates |
| `district` | Shanghai district of the base station (added during cleaning) |
| `user_id` | Anonymised user identifier |

The raw data consists of 12 half-month Excel files (`data_<month>.<start>~<month>.<end>.xlsx`), 6,952,921 records in total.

District boundaries (`Data/Reference/shanghai_districts.geojson`) are the 16 current Shanghai districts from [DataV.GeoAtlas](https://datav.aliyun.com/portal/school/atlas/area_selector) (Alibaba Cloud).

## Method

1. **Cleaning:** duplicates, missing coordinates and invalid records removed; each station assigned to a district (6,150,955 clean records).
2. **Hourly load:** number of different users connected to each station in each hour (1,943 regularly active stations × 4,392 hours).
3. **Spike label:** an hour is a spike when load is above the station's average for the previous 7 days plus two standard deviations, with at least 3 users (about 2% of hours).
4. **Features:** 83 candidate features (load history, activity, spike history, spatial, calendar, real-time state, hourly profile), reduced to **15** by LightGBM importance ranking, choosing the smallest set within 1% of the best validation PR-AUC.
5. **Models:** persistence baseline, logistic regression and LightGBM (hyperparameters tuned with Optuna), trained on a chronological 70 / 30 split with a validation period for early stopping and threshold selection.
6. **Analysis:** feature importance (gain and SHAP), performance on holidays and special events, and an event study.

## Project structure

```
.
├── main.ipynb                      Full pipeline: cleaning, features, models, results
├── requirements.txt                Pinned Python dependencies
├── Data/
│   ├── Raw/                        12 original Excel files (read-only, not tracked in git)
│   ├── Reference/
│   │   └── shanghai_districts.geojson
│   └── Clean/
│       └── sessions_clean.parquet  Clean dataset (generated, not tracked in git)
└── README.md
```

## Reproducing the results

1. Install Python 3.12 and the dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Place the 12 raw Excel files in `Data/Raw/`.
3. Open `main.ipynb` and run all cells (**Restart → Run All**). A full run takes about 30 minutes. The notebook writes `Data/Clean/sessions_clean.parquet`.

Hyperparameter tuning is switched off by default (`RUN_TUNING = False`, tuned values are in the configuration cell). Set it to `True` to repeat the Optuna search (requires `optuna`, about 12 minutes extra).

## Cleaning pipeline

| Step | Rule | Removed | Records after |
|---|---|---:|---:|
| Raw data | — | — | 6,952,921 |
| Duplicates | Record identical to another in every column | 2,055 | 6,950,866 |
| Missing values | Missing latitude / longitude | 715,984 | 6,234,882 |
| Invalid records | `end_time` < `start_time`; start outside 1 Jun – 30 Nov 2014; coordinates outside the Shanghai bounding box | 83,399 | 6,151,483 |
| District assignment | Station more than 1 km outside every district boundary | 528 | **6,150,955** |

Result: 6,150,955 clean records from 9,616 users at 3,006 base stations in 15 districts.

## Known limitations

- **Missing coordinates are not random.** The missing rate rises from about 8% in June to about 13% in November, and the top 10% of users account for about 79% of missing records (compared with about 49% of all records). Removing these records slightly under-counts station load, more so in later months.
- **Zero-length sessions are kept.** 86,231 sessions (1.4%) have `start_time == end_time`. They are treated as valid connection events. About 95% of them come from the top 10% of users.
- **Data collection outage on 12 October 2014.** Almost no sessions were logged network-wide between 00:00 and 22:59 (1,287 sessions that day against about 32,500 on a normal day). These hours are treated as missing in the hourly load series and excluded from spike labels and baselines.
- **Post-holiday rebound.** Spikes are measured against the previous 7 days, so working days right after a long holiday (e.g. 8–11 October, after National Day) show more spikes because the baseline week was quiet.
- **District boundaries are current, not historical.** The former Zhabei district, which merged into Jing'an in 2015, is labelled Jing'an.
