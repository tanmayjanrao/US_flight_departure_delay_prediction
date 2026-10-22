# Flight Departure Delay Prediction: Project Context

## Goal
Regression problem: predict flight **departure delay** (minutes, `DEP_DELAY`) for US flights.

## Project Structure
Local folder: `D:\US_delay_departure`
GitHub: https://github.com/tanmayjanrao/Flight-Departure-Delay-Prediction

```
US_delay_departure/
├── notebooks/
│   ├── 01_basic_checks.ipynb
│   ├── 02_eda.ipynb
│   ├── 03_feature_engineering.ipynb
│   └── 04_modeling.ipynb
├── PROJECT_CONTEXT.md
├── README.md
└── .gitignore
```
- One notebook per stage. Commit to GitHub once per Kaggle session.
- Data is never committed (too large; lives on Kaggle).

## Data
- Source: US BTS on-time performance, monthly ZIPs (Jan 2022 to Aug 2026).
- Raw parquet (all columns stored as strings): `/kaggle/input/datasets/tjaycuz/us-flight-data/US_flight_data.parquet`
- Size: **32,957,871 rows x 119 columns**, 195 row groups.
- **Working `df`: 32,957,871 rows x 68 columns, 4.5 GB** (51 columns with >=99% nulls left out at load time).
- Saved typed copy: `/kaggle/working/flights_df_68cols.parquet` (compact dtypes preserved).
  - Status: **confirm the save cell ran and that the file was kept as a Kaggle Dataset / saved version.**
  - Load next session with: `df = pd.read_parquet("<path from Add Input panel>/flights_df_68cols.parquet")`

## Working Rules
- Dataframe is always named `df`.
- Do not drop columns until the data shows they are not worth keeping.
- Short answers, code in cells. One cell at a time when the output decides the next step.
- Never load the raw parquet with default pandas (all-string columns, slow and RAM heavy). Use the typed parquet.
- Always check for hidden nulls before casting to plain `int` (nulls show as 0.00% when rounded).

## Dtype Conversion (DONE)
Raw all-string columns converted to compact types; memory 6.6 GB -> 4.5 GB.
- `int8`: date parts, `CANCELLED`, `DIVERTED`, `DISTANCE_GROUP`, state FIPS, WAC
- `int16` / `uint16` / `int32`: times, distance, carrier/airport IDs, city market IDs, airport seq IDs
- Nullable `Int8` / `Int16` / `UInt16`: columns with nulls (delay, time, `*_GROUP`, `*_DEL15`, `MKT_*`) plus three with hidden nulls: `DIV_AIRPORT_LANDINGS`, `CRS_ELAPSED_TIME`, `OP_CARRIER_FL_NUM`
- `category`: all text columns
- `DEP_DELAY` (target) stays `float32` so nulls remain NaN.

## Findings So Far
- **Confirmed:** all 566,961 null `DEP_DELAY` rows are cancelled flights (`CANCELLED=1`, `DIVERTED=0`).
- 19,447 cancelled flights still have a `DEP_DELAY`; 86,850 diverted flights all have one. Revisit in EDA.
- 51 columns dropped at load: `SCH_OP_*`, `DIV1-5_*`, `DIV_*` (except `DIV_AIRPORT_LANDINGS`), `FIRST_DEP_TIME`, `TOTAL_ADD_GTIME`, `LONGEST_ADD_GTIME`.
- `MKT_*` and `DUP`: 64% null (schema difference across months). Use `OP_UNIQUE_CARRIER` for carrier.
- Delay cause columns: 78.94% null; `CANCELLATION_CODE` 98.22% null.
- `FLIGHTS` is always 1 and `DUP` has one unique value: constant, useless (drop candidates, not dropped yet).
- `DEP_DELAY` range: -115 to 7223 (outliers present, not handled yet).
- **`CRS_ELAPSED_TIME` min is -272: impossible, needs checking.**
- Core pre-departure columns (date parts, origin, dest, scheduled times, distance, carrier) have 0% nulls. `TAIL_NUM` 0.33% null.

## Leakage Rule
- Post-departure columns of the **same flight** (`ARR_*`, `TAXI_*`, `WHEELS_*`, `AIR_TIME`, `ACTUAL_ELAPSED_TIME`, delay causes, `DEP_TIME`, `DEP_DELAY_NEW`, `DEP_DEL15`, `DEP_DELAY_GROUP`) must **not** be model inputs as-is.
- They are kept as raw material for feature engineering: use the **previous flight of the same `TAIL_NUM`** (prev arrival delay, prev taxi-out, prev delay causes).
- Check that the previous flight landed before the current scheduled departure, otherwise the lag feature leaks too.
- Do **not** drop null-`DEP_DELAY` rows until lag features are built (cancelled flights break the tail-number chain).

## Decisions Log
| # | Decision | Status |
|---|----------|--------|
| 1 | Drop the 16 fully-null `DIV4_*` / `DIV5_*` columns | Done |
| 2 | Drop all columns with >=99% nulls (51 total, kept 68) | Done |
| 3 | Use `OP_UNIQUE_CARRIER` instead of `MKT_UNIQUE_CARRIER` | Proposed |
| 4 | Null `DEP_DELAY` = cancelled flights | Confirmed; rows dropped only after lag features |
| 5 | Keep all 68 columns for now (no further drops) | Decided |
| 6 | Convert to compact dtypes | Done |
| 7 | Outlier handling for `DEP_DELAY` | Pending |
| 8 | Time-based train/test split (data is chronological) | Planned |

## Session Log
| Session | Done |
|---------|------|
| 1 | Merged monthly ZIPs into one CSV; converted to Parquet; verified integrity |
| 2 | Moved to Kaggle; null counts from Parquet stats for all 119 columns; identified 16 fully-null columns |
| 3 | Dropped 51 columns with >=99% nulls (68 kept); converted dtypes (6.6 -> 4.5 GB); confirmed null `DEP_DELAY` = cancelled; saved `flights_df_68cols.parquet` |

## Next Steps
- [ ] Confirm saved parquet persisted and loads in a fresh session
- [ ] Check `CRS_ELAPSED_TIME` negative values
- [ ] EDA (target distribution, outliers, patterns by hour/carrier/airport/month)
- [ ] Feature engineering (previous flight per `TAIL_NUM`, rolling airport/carrier delays, time features)
- [ ] Drop cancelled rows after lag features
- [ ] Time-based split, baseline model, tuned models
- [ ] Evaluate (MAE / RMSE / R2)
