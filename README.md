# Base Station Traffic Spike Prediction — Shanghai Telecom

Data preparation for predicting traffic spikes at mobile base stations, using the Shanghai Telecom base station access dataset (June – November 2014).

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

## Project structure

```
.
├── main.ipynb                      Data loading, overview, cleaning and export
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
3. Open `main.ipynb` and run all cells (**Restart → Run All**). This takes about 2 minutes. The notebook writes `Data/Clean/sessions_clean.parquet`.

Later stages load the clean data directly:

```python
import pandas as pd
df = pd.read_parquet('Data/Clean/sessions_clean.parquet')
```

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
- **District boundaries are current, not historical.** The former Zhabei district, which merged into Jing'an in 2015, is labelled Jing'an.
