# PostgreSQL: Time-Series Features, JSONB and Index Optimization

**Project:** `goit-rdb-hw-07`  
**Notebook:** `hw7.ipynb`  
**Stack:** Python, PostgreSQL, Jupyter Notebook, SQLAlchemy, `pgserver`, pandas, Parquet, SQL Magic (`%%sql`).

## Project overview

This homework covers time-series feature engineering with PostgreSQL window functions, JSONB event queries, GIN and B-tree index benchmarking, and comparison of JSONB versus normalized storage.

## Data sources and downloads

**NYC TLC Yellow Taxi trip records:** [NYC Taxi & Limousine Commission — Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page). The notebook works with Yellow Taxi trip records for **February and March 2024** in Parquet format:

- [yellow_tripdata_2024-02.parquet](https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-02.parquet)
- [yellow_tripdata_2024-03.parquet](https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-03.parquet)

Download the files from the links above, or use the data-loading cells in `hw7.ipynb` if they handle retrieval. Update local paths in the notebook as needed.

**Synthetic JSONB events:** No external download is required. SQL `generate_series(1, 50000)` generates 50,000 events in `t7_hw_events`, spanning `page_view`, `click`, `add_to_cart`, `checkout`, and `recommendation_shown` (10,000 of each type).

## How to run

1. Open the `goit-rdb-hw-07` folder in VS Code.
2. Create and activate a Python virtual environment (macOS/Linux):

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

3. Install dependencies:

   ```bash
   python -m pip install jupyter ipykernel ipython-sql sqlalchemy psycopg2-binary pgserver pandas pyarrow
   ```

   Install any additional packages imported in the notebook's setup cells.
4. Open `hw7.ipynb`, choose the `.venv` Python kernel, and execute the setup cells to initialize PostgreSQL using `pgserver` and the SQL Magic connection.
5. Download/load the TLC Parquet files and create the daily taxi table.
6. Run time-series feature queries, generate JSONB events, and complete indexing experiments **in notebook order**.
7. Create the normalized checkout table and compare the two query plans.

`%%sql` cells require a working SQL Magic database connection. The MongoDB aggregation example is **Markdown pseudocode**, not PostgreSQL SQL.

## Time-series features

Taxi trips are aggregated in `t7_hw_taxi_daily` at the **pickup zone × day** level, with `pickup_zone_id`, `ts_day`, `trip_count`, `avg_fare`, and `total_distance`. The view `v_t7_hw_taxi_features` adds:

| Feature | Calculation | Purpose |
|---|---|---|
| `trips_lag_1d` | `LAG(trip_count, 1)` | Previous observed daily trip count |
| `trips_lag_7d` | `LAG(trip_count, 7)` | Trip count seven observed rows earlier |
| `trips_lag_30d` | `LAG(trip_count, 30)` | Trip count thirty observed rows earlier |
| `rolling_avg_fare_7d` | `AVG(avg_fare)` over a 7-calendar-day `RANGE` | Rolling average of daily average fare |
| `rolling_std_fare_30d` | `STDDEV_SAMP(avg_fare)` over a 30-calendar-day `RANGE` | Rolling fare variability |
| `cumulative_trips` | Cumulative `SUM(trip_count)` | Total observed trips so far |
| `expanding_avg_trips` | Expanding `AVG(trip_count)` | Mean daily trip count so far |

Window functions partition by `pickup_zone_id`. The feature view excludes rows where `trips_lag_30d IS NULL`.

**ML caution:** `LAG(..., 7)` and `LAG(..., 30)` count preceding *rows*, not necessarily exact calendar days when dates are missing. The rolling and expanding examples include the current day; shift such features when necessary to avoid future-information leakage at prediction time.

## JSONB events and queries

The `t7_hw_events` table contains `event_id`, `user_id`, `event_ts`, `event_type`, and `payload JSONB`. The notebook demonstrates:

- **Containment:** `payload @> '{"payment_method":"card"}'::jsonb`.
- **Key existence:** `?`, `?|`, and `?&` for top-level JSONB keys.
- **JSONPath:** `jsonb_path_query` to expand recommendation arrays and `jsonb_path_exists` for conditions.
- **Field extraction:** `->` for JSONB values and `->>` for text, with numeric casts for calculations.

GIN `jsonb_path_ops` supports containment and supported JSONPath searches, but not key-existence operators `?`, `?|`, and `?&`; those can use GIN `jsonb_ops` instead.

## Index benchmarks: `EXPLAIN (ANALYZE, BUFFERS)`

A GIN index is created with:

```sql
CREATE INDEX idx_t7_events_payload_gin_pathops
ON t7_hw_events USING GIN (payload jsonb_path_ops);
```

**Observed notebook results:**

| Query | Before | After | Plan after indexing |
|---|---:|---:|---|
| `payment_method = 'card'` via `@>` | 19.788 ms | 18.003 ms | `Bitmap Index Scan` + `Bitmap Heap Scan` |
| `referrer = 'google'` via `@>` | Not measured | 5.239 ms | `Bitmap Index Scan` + `Bitmap Heap Scan` |

Before indexing, the card query used `Seq Scan`; after indexing it matched **3,334 rows** through the bitmap plan. The GIN index size was **1,144 kB**. Its modest speedup reflects, in part, that the heap scan still accessed **1,000 heap blocks**.

For text equality, an expression B-tree index was created:

```sql
CREATE INDEX idx_t7_events_device_btree
ON t7_hw_events ((payload ->> 'device'));
```

For `payload ->> 'device' = 'mobile'`, execution time decreased from **11.097 ms** (`Seq Scan`) to **6.349 ms** (`Bitmap Index Scan` + `Bitmap Heap Scan`), matching **8,333 rows**.

**Read/write trade-off:** indexes can accelerate matching reads, but use disk space and increase index-maintenance work on inserts, updates, and deletes. PostgreSQL may still select `Seq Scan` when it estimates that to be cheaper.

## JSONB versus normalized checkout storage

`t7_hw_checkouts_norm` contains typed columns for checkout events: `event_id`, `user_id`, `event_ts`, `order_id`, `total`, `items_count`, and `payment_method`, with integrity constraints and a B-tree index on `payment_method`.

Both approaches calculate the average checkout total and event count by payment method.

| Metric | JSONB events table | Normalized checkout table |
|---|---:|---:|
| Rows scanned | 50,000 | 10,000 |
| Rows aggregated | 10,000 | 10,000 |
| Execution time | 10.704 ms | 3.235 ms |
| Shared buffer hits | 1,000 | 94 |
| Plan operators | `Seq Scan`, `HashAggregate`, `Sort`, `Limit` | `Seq Scan`, `HashAggregate`, `Sort`, `Limit` |

The normalized query was approximately **3.3× faster** in this run. This is not an isolated JSONB-versus-typed-columns benchmark: the normalized table is also smaller and contains only checkout events.

### MongoDB bonus

The notebook includes Markdown-only MongoDB aggregation pseudocode using `$match`, `$group`, `$sort`, and `$limit` for the same analytical task. No MongoDB server is required for this example.

## Design decisions and future extension

- **JSONB:** suitable for raw, heterogeneous events and evolving event schemas while retaining SQL, joins, and transactions in PostgreSQL.
- **Normalized SQL:** suitable for curated, validated, regularly queried data and ML feature tables with stable types.
- **Adding a `refund` event:** in JSONB, add a new event type and payload shape, then update application validation and processing logic; a database schema migration may not be necessary. In a normalized design, create a typed refund table with appropriate constraints and indexes, and update ETL jobs and downstream queries.

## Reproducibility

Run the notebook cells **in order**: some steps drop and recreate tables or indexes. Query timing varies with cache state, database configuration, and machine resources. The timings documented above are measurements from this project's SQL outputs, not general performance guarantees.
