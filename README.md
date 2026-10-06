# Base Station Traffic Spike Prediction (Shanghai Telecom)

This project predicts if a mobile base station will have a traffic spike in the next hour. It uses the Shanghai Telecom dataset from June to November 2014.

## Results

The models were tested on data they never saw during training (3 October to 30 November 2014).

| Model | PR AUC | ROC AUC | Precision | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Persistence (spike now, so spike next hour) | 0.136 | 0.667 | 0.349 | 0.349 | 0.349 |
| Logistic regression | 0.402 | 0.940 | 0.407 | 0.425 | 0.416 |
| **LightGBM** | **0.467** | **0.949** | **0.460** | **0.434** | **0.447** |

Only 2.1% of the test hours are spikes. For this reason PR AUC is the main measure. A random guess scores 0.021. Accuracy is not used, because saying "no spike" every time already gives 97.9%.

## Dataset

**Source:** [Shanghai Telecom Dataset](http://sguangwang.com/TelecomDataset.html), shared by Shanghai Telecom and published by Prof. Shangguang Wang. It has more than 7.2 million records of 9,481 mobile phones using 3,233 base stations over six months. The dataset page lists the papers to cite when you use the data. A copy is also on [Kaggle](https://www.kaggle.com/datasets/mexwell/telecom-shanghai-dataset).

The raw files are not in this repository. Download them from the source above and put them in `Data/Raw/`.

Each record is one user session at a base station.

| Column | Meaning |
|---|---|
| `month` | Month of the record |
| `start_time` | When the session started |
| `end_time` | When the session ended |
| `latitude`, `longitude` | Location of the base station |
| `district` | Shanghai district of the base station (added during cleaning) |
| `user_id` | Hidden user ID |

The raw data comes in 12 Excel files, each covering half a month. Together they have 6,952,921 records.

The district borders (`Data/Reference/shanghai_districts.geojson`) are the 16 current Shanghai districts from [DataV.GeoAtlas](https://datav.aliyun.com/portal/school/atlas/area_selector) by Alibaba Cloud.

## How it works

1. **Cleaning:** we remove copies, records with no location and wrong records. Each base station is then given its district. This leaves 6,150,955 clean records.
2. **Hourly load:** for each station and each hour, we count how many different users were connected. We keep 1,943 stations that are active on at least half of the days.
3. **Spike:** an hour is a spike when the station is much busier than normal. This means more users than its average of the past 7 days plus two standard deviations, and at least 3 users. About 2% of hours are spikes.
4. **Features:** we build 83 candidate features from the data. Then we keep the 15 most useful ones. We pick the smallest set that scores within 1% of the best.
5. **Models:** we compare a simple rule (persistence), logistic regression and LightGBM. The data is split by time into 70% for training and 30% for testing. LightGBM settings were tuned with Optuna.
6. **Analysis:** we show which features matter most and how the model does on holidays and special event days.

## Project files

```
.
├── main.ipynb                      Full project: cleaning, features, models and results
├── requirements.txt                Python packages and versions
├── Data/
│   ├── Raw/                        12 original Excel files (not stored in git)
│   ├── Reference/
│   │   └── shanghai_districts.geojson
│   └── Clean/
│       └── sessions_clean.parquet  Clean data made by the notebook (not stored in git)
└── README.md
```

## How to run it

1. Install Python 3.12 and the packages:
   ```bash
   pip install -r requirements.txt
   ```
2. Download the 12 raw Excel files from the [Shanghai Telecom Dataset](http://sguangwang.com/TelecomDataset.html) page and put them in `Data/Raw/`.
3. Open `main.ipynb` and run all cells (**Restart, then Run All**). A full run takes about 30 minutes. The notebook saves the clean data to `Data/Clean/sessions_clean.parquet`.

Tuning is turned off by default (`RUN_TUNING = False`) because the tuned settings are already saved in the notebook. Set it to `True` to run the tuning again. This needs the `optuna` package and takes about 12 more minutes.

## Data cleaning steps

| Step | Rule | Removed | Records left |
|---|---|---:|---:|
| Raw data | | | 6,952,921 |
| Copies | The record is exactly the same as another record | 2,055 | 6,950,866 |
| No location | The latitude or longitude is missing | 715,984 | 6,234,882 |
| Wrong records | End time is before start time, or the date is outside June to November 2014, or the location is outside Shanghai | 83,399 | 6,151,483 |
| No district | The station is more than 1 km outside every district border | 528 | **6,150,955** |

In the end there are 6,150,955 clean records from 9,616 users at 3,006 base stations in 15 districts.

## Known limits

* **Missing locations are not random.** The share of records with no location goes up from about 8% in June to about 13% in November. Also, 10% of users make up about 79% of these records, while they make up only about 49% of all records. Removing them makes station load look a little lower, more so in the later months.
* **Very short sessions are kept.** 86,231 sessions (1.4%) have the same start and end time. We treat them as real connections. About 95% of them come from 10% of users.
* **Data is missing on 12 October 2014.** Almost nothing was recorded from 00:00 to 22:59 that day (1,287 sessions compared with about 32,500 on a normal day). These hours are marked as missing and are not used for spikes.
* **Busy days after holidays.** Spikes are measured against the past 7 days. Right after a long holiday (for example 8 to 11 October, after National Day) the past week was quiet, so normal working days show more spikes.
* **District borders are from today, not 2014.** The old Zhabei district joined Jing'an in 2015, so its stations are counted as Jing'an.
