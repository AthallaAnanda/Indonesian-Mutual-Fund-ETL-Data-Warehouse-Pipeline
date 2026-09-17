# Indonesian Mutual Fund ETL (pasardana.id)

**Database Assistant Selection Stage 2 2026: ETL Project**
*Data Scraping, Database Modeling, and Data Storing*

**Author:** R. Athalla Ananda Putra
**Student ID:** 18224060

---

## Table of Contents

1. [Overview](#1-overview)
2. [Progress Status](#2-progress-status)
3. [Running the Project](#3-running-the-project)
4. [Data Scraping: Workflow and Usage](#4-data-scraping-workflow-and-usage)
5. [Scraper Output JSON Structure](#5-scraper-output-json-structure)
6. [Transform and Load to OLTP](#6-transform-and-load-to-oltp)
7. [ERD and Relational Diagram](#7-erd-and-relational-diagram)
8. [Screenshots](#8-screenshots)
9. [Bonus](#9-bonus)
10. [Known Limitations](#10-known-limitations)
11. [AI Usage](#11-ai-usage)
12. [References](#12-references)

This document is the main README for the submission.

---

## 1. Overview

**Topic:** Historical NAV data, portfolios, and the Indonesian mutual fund ecosystem.
**Data source:** [pasardana.id](https://pasardana.id), accessed through internal REST endpoints using a browser session rather than a key-protected public API.
**DBMS:** PostgreSQL 16, run through Podman or Docker Compose.

### Why this topic was selected

Indonesian mutual funds have a rich relational data structure. One investment manager manages many funds, and each fund has a custodian bank, monthly portfolio data, benchmarks, and its own network of mutual fund selling agents. This topic was not used in the 2024 selection, and its data source, the internal pasardana.id endpoints, differs from other commonly used mutual fund portals. The availability of daily NAV and monthly AUM data also makes this topic suitable for building fact tables in a data warehouse.

### Summary

- 21 entities are stored in the OLTP database, with approximately 514,000 total rows.
- The data covers approximately 1,520 funds from 100 investment managers, 23 custodian banks, and 90 mutual fund selling agents.
- The bonus data warehouse uses a fact constellation model with 3 fact tables and 5 conformed dimensions.
- The bonus automated scheduling setup runs through Apache Airflow with 3 DAGs scheduled at different intervals based on the source data update frequency.

---

## 2. Progress Status

| Stage | Scope | Status |
|---|---|---|
| 0 | Environment setup, including venv, Podman, and initial DDL | Complete |
| 1 | Extract: scrape all endpoints, including master, fund, snapshot, portfolio, and quarterly data | Complete |
| 2 | Transform: preprocess 21 entities into clean JSON files | Complete |
| 3 | Load: load data into the PostgreSQL OLTP database | Complete |
| 4 | Bonus: data warehouse OLAP schema and OLTP to DW ETL | Complete |
| 5 | Bonus: automated Airflow scheduling with 3 DAGs for daily, monthly, and master data | Complete |
| 6 | Bonus: 3 query optimizations using indexes and a materialized view | Complete |
| 7 | Documentation and visual artifacts, including ERD, relational diagram, screenshots, and SQL exports | Complete |
| 8 | Final review and submission | Pending final review and pull request creation |

---

## 3. Running the Project

```bash
# 1. Prepare the Python environment
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
playwright install chromium

# 2. Start the database, including PostgreSQL and pgAdmin
podman-compose up -d
# or: docker compose up -d

# 3. Extract data. This may take some time and supports resume checks.
cd "Data Scraping/src"
python main.py

# 4. Transform the scraped data
python preprocessor.py

# 5. Load the data into OLTP
cd "../../Data Storing/src"
python load_data.py
```

Database credentials are stored in the `.env` file at the project root, including the host, port, database name, user, and password. The file is not committed to Git. PostgreSQL runs in the `reksadana_pg` container with port 5432 exposed to the host, so it can be accessed through `psql` or GUI tools such as pgAdmin and DBeaver. pgAdmin is available at `http://localhost:5050`.

### Running the bonus components: data warehouse and scheduling

```bash
# The data warehouse DDL schema is created automatically by docker-compose.
# To load or refresh the data, run:
podman exec -i reksadana_pg psql -U postgres -d reksadana < "Data Storing/Data Warehouse/src/load_dw.sql"

# The Airflow DAGs are available in airflow/dags/. Run Airflow standalone
# from a separate virtual environment, then enable reksa_dana_daily_nav,
# reksa_dana_monthly, and reksa_dana_master through the Airflow UI or CLI.
```

---

## 4. Data Scraping: Workflow and Usage

### Method

Scraping uses Playwright to open a real browser session and obtain valid session cookies. The scraper then calls the internal pasardana.id REST endpoints through `page.evaluate(fetch())` inside the browser context. No key-protected official API is used. All endpoints were identified by inspecting the browser DevTools Network tab under Fetch/XHR, then accessed in the same way as a normal browser session.

### Eight scraping phases

The main code is available at `Data Scraping/src/main.py` and runs through the following eight phases in sequence.

| Phase | Scope | Estimated number of calls |
|---|---|---:|
| 1 | Master data for investment managers, custodian banks, and mutual fund selling agents | 4 |
| 2 | Complete fund list with pagination | ~10 |
| 3 | Monthly AUM per investment manager | ~100 |
| 4 | Fund classes per investment manager for multi-class groups | ~100 |
| 5 | NAV, AUM, performance, ranking, and benchmark snapshots per fund | ~1,000 |
| 6 | Historical portfolio data per fund, with one call containing 12 monthly snapshots | ~1,000 |
| 7 | Quarterly returns per fund for the last five years | ~5,000 |
| 8 | Fund to selling agent junction data | ~10 to 20 |

### Usage

```bash
cd "Data Scraping/src"

# Run all eight phases, which is the default behavior
python main.py

# Run only selected phases and force a refetch even when a cache is available
python main.py --phases 5 --force
```

Raw scraping results are stored in `Data Scraping/raw/` as one JSON file per endpoint. These files are not committed to Git because they are large and can be reproduced. A resume check is available: if the process stops partway through, running `python main.py` again skips completed phases or files instead of starting from the beginning.

After raw scraping is complete, run `python preprocessor.py` to produce clean JSON files in `Data Scraping/data/`, as described in sections 5 and 6. Details for each endpoint, including parameters and response structures, are available in `Data Scraping/src/endpoints.py`.

---

## 5. Scraper Output JSON Structure

Preprocessed data is separated by entity rather than combined into one large file. The files are stored in `Data Scraping/data/`. The following 21 files are committed to Git.

| File | Row count | Main fields |
|---|---:|---|
| `investment_managers.json` | 100 | `manager_id, name, ojk_code, mi_permit_num, address, capital, paid_in_capital, is_active, data_last_update, ...` |
| `manager_personnel.json` | 456 | `manager_id, name, title` |
| `manager_shareholders.json` | 241 | `manager_id, shareholder_name, share_amount` |
| `manager_aum_records.json` | 949 | `manager_id, record_date, aum_value, total_units` |
| `custodian_banks.json` | 23 | `bank_id, name, ojk_code, address, ownership_status, activity_status, is_active, ...` |
| `sales_companies.json` | 90 | `company_id, name, aperd_id, npwp, address, contact_person, ...` |
| `fund_classes.json` | 182 | `class_group_id, base_name` |
| `funds.json` | 1,520 | `fund_id, manager_id, bank_id, class_group_id, name, isin_code, fund_type, currency, is_sharia, is_etf, is_index, ipo_date, official_benchmark, fee/policy fields, ...` |
| `nav_records.json` | 341,175 | `fund_id, record_date, nav_value, class_total_value` |
| `aum_records.json` | 15,462 | `fund_id, record_date, published_date, aum_value, total_units, class_total_value` |
| `asset_categories.json` | 17 | `category_id, name` |
| `securities.json` | 7,275 | `security_id, code, name, security_type, source_stock_id` |
| `portfolio_snapshots.json` | 9,625 | `fund_id, date_based, domestic_allocation_pct, foreign_allocation_pct` |
| `portfolio_allocations.json` | 23,189 | `snapshot_ref [fund_id, date_based], category_id, value_pct` |
| `portfolio_holdings.json` | 87,999 | `snapshot_ref [fund_id, date_based], security_id, value_pct` |
| `fund_performances.json` | 1,309 | `fund_id, period_code, as_of_date, return_pct, std_dev, beta, sharpe_ratio, treynor_ratio, sortino_ratio, cagr, ...` |
| `fund_rankings.json` | 6,071 | `fund_id, period_code, category_code, as_of_date, risk_rank, rating_rank, pasardana_rating, ...` |
| `fund_quarterly_returns.json` | 5,646 | `fund_id, quarter_start, return_pct` |
| `benchmarks.json` | 19 | `benchmark_id, name` |
| `benchmark_data_points.json` | 4,243 | `benchmark_id, record_date, value` |
| `fund_benchmarks.json` | 6,532 | `fund_id, benchmark_id, is_official` |
| `fund_sales_companies.json` | 3,023 | `fund_id, company_id, commission_fee` |

The following notes are important for interpreting the files above.

- The `snapshot_ref` field in `portfolio_allocations.json` and `portfolio_holdings.json` is a pair of `[fund_id, date_based]` values, not a direct foreign key to the `snapshot_id` surrogate key. The `snapshot_id` value is generated during the database load process, as described in section 6, because the JSON layer does not have a database that can provide surrogate keys.
- Derived attributes such as `daily_return` and `conservative_label` are intentionally not written to any JSON file. They are generated by the database through views and generated columns, as described in section 7.
- Field names are normalized to snake_case and their data types are parsed. Currency strings become integers, percentages become floats, and dates use the ISO `YYYY-MM-DD` format instead of the raw API string format.

Complete parsing rules are available in `Data Scraping/src/preprocessor.py`.

---

## 6. Transform and Load to OLTP

### Transform: `Data Scraping/src/preprocessor.py`

Raw JSON is transformed into 21 clean files. Important processing steps include:

- Parsing basic types, including currency strings to integers, percentages to floats, and dates to the ISO format.
- Grouping multi-class funds through `FkClassFundId` from the API instead of parsing product names, because product name formats are inconsistent.
- Combining stocks and bonds into the `security` supertype because both play the same role as instruments held in a portfolio.
- Leaving derived attributes such as `daily_return` and `conservative_label` out of this stage because they are implemented as a view or generated column in the database.

### Load: `Data Storing/src/load_data.py`

The loader reads all transformed JSON files and inserts them into PostgreSQL according to foreign key dependencies. Master tables are loaded first, followed by dependent tables, time-series tables, and finally junction tables that require `snapshot_id`.

All tables are loaded using `INSERT ... ON CONFLICT` so that the process is safe to repeat and remains idempotent. The `manager_personnel` and `manager_shareholder` tables do not have reliable natural unique columns, so they are handled differently. Existing rows for the same `manager_id` are deleted before replacement rows are inserted.

The final result is 21 populated data tables and related derived tables containing approximately 514,000 rows. The process has been verified as follows:

- No dangling foreign keys are present.
- The `conservative_label` column is generated correctly by the database. For example, the ABF Indonesia Bond Index Fund has `is_index=true`, so it automatically receives the `Indeks` label instead of `Pendapatan Tetap`, even though its `fund_type` is bond.
- The loader was run twice in succession and row counts remained unchanged across all tables, demonstrating that the process is idempotent.

### Data issues found and handling decisions

During loading, several technically invalid values were found because they violated database column limits or constraints. All of them came from the source data, not from the scraper or transformer.

1. **302 rows in `fund_ranking`** had all rank and rating columns set to 0 at the same time. This is an API sentinel meaning that no ranking was available for that period, not an actual ranking, so those rows were skipped during loading.
2. **One protection fund**, Batavia Proteksi Maxima 16, had `min_next_subscription = -1`, which is likely an "not applicable" sentinel. The value was changed to `NULL`.
3. **One fund**, MEGA ASSET MANTAP, had `red_fee_max_pct = 200`, while its other fees were in the expected 1 to 3 percent range. The value was set to `NULL`.
4. **Two pairs of fund classes** shared exactly the same ISIN code. Because it was not possible to determine which class had the correct code, only the first occurrence was retained and the others were set to `NULL`.
5. **Three funds** had an invalid combination of allocation policy percentages for bonds, equities, and money market instruments. Some values were even negative. The problematic fields were set to `NULL` per fund.
6. **Three rows in `fund_performance`** had `treynor_ratio` values in the tens of thousands, far outside a reasonable range. This happened because beta was close to zero, causing the return divided by beta ratio to become mathematically very large. The values were set to `NULL`.
7. **Some `portfolio_allocation` and `portfolio_holding` rows** appeared twice for the same snapshot and category or instrument combination, but with different percentage values. After inspection, the total for each affected snapshot was still exactly 100 percent, indicating that the rows represented two separate portions that happened to receive the same category code from the source. The handling decision was to add the values together rather than discard one row.

There was also an important finding related to `asset_category`. Category codes 7 and 9 were both labeled `Pasar Uang` in the transformed data, even though they appeared together in several snapshots and were therefore clearly different categories. After inspecting the raw JSON, category 9 was labeled `Real Estate` for most funds at the source. Therefore, category 9 is stored as `Real Estate` in the database to preserve its original meaning.

---

## 7. ERD and Relational Diagram

### ERD and relational diagram images

![ERD](Data%20Storing/design/erd.png)

![Relational Diagram](Data%20Storing/design/relational_diagram.png)

The cardinality and participation of each relationship follow the ERD above and are consistent with the constraints defined in `Data Storing/src/ddl_oltp.sql`, including `NOT NULL`, `UNIQUE`, and `ON DELETE CASCADE`.

### Structural summary

- **Strong entities:** `investment_manager`, `custodian_bank`, `sales_company`, `mutual_fund`, `fund_class`, `benchmark`, `asset_category`, and `security`.
- **Weak time-series entities:** `manager_aum_record`, `nav_record`, `aum_record`, `portfolio_snapshot`, `fund_performance`, `fund_ranking`, `fund_quarterly_return`, and `benchmark_data_point`. Each has a reliable partial key, such as `record_date`, to distinguish rows within one owner.
- **Entities with surrogate IDs instead of weak entity keys:** `manager_personnel` and `manager_shareholder`. Both depend on `investment_manager`, but their natural attributes, such as `name`, `title`, and `shareholder_name`, are not sufficiently unique to form reliable partial keys. They therefore use their own surrogate primary keys in the ERD.
- **Many-to-many relationships with attributes:** `fund_benchmark` with `is_official`, `fund_sales_company` with `commission_fee`, `portfolio_allocation` with `value_pct`, and `portfolio_holding` with `value_pct`.

### Translating the ERD into a relational diagram

The translation follows these rules consistently.

1. A **strong entity** becomes a table with its own primary key, such as `manager_id`, `fund_id`, or `bank_id`.
2. A **weak entity** becomes a table with a `NOT NULL` foreign key to its owner, enforced through `ON DELETE CASCADE`, and a composite primary key consisting of the owner foreign key and partial key. For example, `nav_record` uses `(fund_id, record_date)`.
3. A **one-to-many relationship** is represented by a foreign key on the child side, which has total participation, rather than by a separate table. For example, `mutual_fund.manager_id REFERENCES investment_manager`.
4. A **many-to-many relationship with attributes** is implemented as a separate junction table with a composite primary key made from both foreign keys, plus columns for the relationship attributes. For example, `fund_sales_company` contains `fund_id`, `company_id`, and `commission_fee`.
5. **Derived attributes** such as `daily_return` and `conservative_label` are implemented as a **VIEW**, such as `v_nav_return` and `v_benchmark_return`, when they depend on other rows, or as a **GENERATED column** when they depend on columns in the same row.
6. The translated schema was verified against 1NF through BCNF. It has no repeating groups, no partial dependencies on composite keys, and no transitive dependencies. For example, `mutual_fund` stores `manager_id` rather than the investment manager name, which is obtained through a JOIN.

### Use of surrogate keys

Most entities use natural keys taken directly from the source, such as `fund_id`, `manager_id`, and `bank_id`. Surrogate keys are introduced only where they provide a concrete benefit, either by simplifying relationships or by guaranteeing row uniqueness.

- **`portfolio_snapshot.snapshot_id`.** The natural snapshot key is the composite pair `(fund_id, date_based)`, and this pair is referenced by two child tables, `portfolio_allocation` and `portfolio_holding`. A single-column surrogate key simplifies both foreign keys so they do not need to carry two columns. The pair `(fund_id, date_based)` is still retained as a `UNIQUE` constraint to prevent duplicate snapshots.
- **`security.security_id`.** An instrument has a natural key in the form of a `code`, such as the `BBCA` stock code. The surrogate `security_id` is used as the primary key so references from `portfolio_holding` remain compact and stable, while `code` remains `UNIQUE`.
- **`manager_personnel` and `manager_shareholder`.** These entities depend on `investment_manager`, but their natural attributes are not sufficiently unique to form reliable partial keys. Director names and titles, as well as shareholder names, can repeat or be inconsistent. Therefore, these entities are not modeled as weak entities. They receive their own surrogate primary keys, while `manager_id` remains a foreign key to the owner.

The design principle is to introduce surrogate keys only when they provide a concrete benefit, whether for simplifying relationships or guaranteeing row uniqueness. They are not applied uniformly to every table.

The complete DDL for all tables, constraints, views, and generated columns is available at `Data Storing/src/ddl_oltp.sql`.

---

## 8. Screenshots

The following images show successful `SELECT ... FROM ... WHERE` queries executed against the database. The image files are stored in `Data Storing/screenshots/`.

![Screenshot query 1](Data%20Storing/screenshots/query-1.png)

![Screenshot query 2](Data%20Storing/screenshots/query-2.png)

![Screenshot query 3](Data%20Storing/screenshots/query-3.png)

---

## 9. Bonus

### 9.1 Data Warehouse

The data warehouse uses a **fact constellation** model consisting of three fact tables that share conformed dimensions.

The fact tables and their grains are:

| Fact table | Grain |
|---|---|
| `fact_nav_daily` | One fund per day |
| `fact_aum_monthly` | One fund per month |
| `fact_manager_aum_monthly` | One investment manager per month |

The conformed dimensions are `dim_date`, `dim_fund` with SCD Type 2, `dim_manager` with SCD Type 2, `dim_category`, and `dim_custodian` with SCD Type 2.

Each fact table has a different grain and they are not combined, because daily NAV, monthly fund AUM, and monthly investment manager AUM represent three different levels of observation. The `dim_date` and `dim_manager` dimensions are conformed, which enables drill-across analysis between fact tables.

The DDL file is available at `Data Storing/Data Warehouse/src/ddl_dw.sql`, while the ETL from OLTP to the data warehouse is available at `Data Storing/Data Warehouse/src/load_dw.sql`.

#### Storage concept: OLTP upsert and OLAP SCD Type 2

OLTP and the data warehouse intentionally use different approaches to historical data because they serve different purposes.

In **OLTP**, master tables such as `investment_manager`, `custodian_bank`, `sales_company`, `mutual_fund`, `security`, and `fund_class` are loaded using **upsert** with `ON CONFLICT ... DO UPDATE`. When an attribute changes, such as a fee or address, the old value is replaced by the new value and only the current state remains in the master table. This is equivalent to SCD Type 1 and is appropriate for OLTP as the normalized and consistent system of record for current data. To keep a change trail, the OLTP schema includes an `audit_log` table populated through an `AFTER UPDATE` trigger. This table answers "what changed and when", which is a different requirement from historical analysis.

In the **data warehouse OLAP layer**, the main goal is long-term trend analysis, so attribute history must be preserved. For this reason, dimensions with attributes that change over time, namely `dim_fund`, `dim_manager`, and `dim_custodian`, use **SCD Type 2**. Each historical version is stored as a separate row with `valid_from`, `valid_to`, and `is_current` markers. This allows a fact table to answer "what was the fund fee at that time" rather than only "what is the fee now".

In short, OLTP upsert maintains the current state, while SCD Type 2 in the data warehouse preserves historical changes. The two approaches complement each other, and the data warehouse can always be rebuilt from OLTP when needed.

#### SCD and dimension surrogate key decisions

- **SCD Type 2 for `dim_fund`, `dim_manager`, and `dim_custodian`.** These dimensions have attributes that genuinely change over time, including fees, addresses, and license status. If they used SCD Type 1, historical attributes would disappear whenever an ETL run overwrote a row, making historical analysis inaccurate.
- **SCD Type 1 for `dim_date` and `dim_category`.** These dimensions are static or almost never change. `dim_date` is a calendar, while `dim_category` stores `conservative_label` and `fund_type`, which are practically stable for a given fund. Applying SCD Type 2 to them would add complexity without meaningful benefit.
- **Surrogate keys as a consequence of SCD Type 2.** An SCD Type 2 dimension can contain multiple rows for the same entity, one row for each historical version, so a natural key such as `fund_id` can no longer be the primary key. These dimensions therefore use surrogate keys, namely `dim_fund_key`, `dim_manager_key`, and `dim_custodian_key`, as primary keys. The natural keys such as `fund_id` remain ordinary columns that may repeat across versions.
- **Natural fact table grain.** Fact tables retain their natural grain, such as `(date_id, fund_id)`, and add `dim_*_key` columns to identify the applicable dimension version when each fact is loaded. The grain does not change. The surrogate key only links the fact to the correct dimension version.

The SCD Type 2 design creates two important rules for analytical queries.

- Joins to SCD Type 2 dimensions must use `dim_*_key` rather than natural keys such as `fund_id`. Joining through a natural key would duplicate results when one entity has multiple historical versions.
- Aggregations of absolute values such as `SUM` or `AVG` on `aum_value` must be partitioned by `currency_code` because some funds use USD. Aggregations of ratios such as `daily_return` are safe across currencies.

Example analytical query for annual volatility by fund category:

```sql
SELECT dc.label,
       COUNT(DISTINCT f.fund_id)        AS n_fund,
       STDDEV(f.daily_return)*SQRT(252) AS ann_volatility,
       AVG(f.daily_return)*252          AS ann_return
FROM dw.fact_nav_daily f
JOIN dw.dim_category dc ON f.category_id = dc.category_id
JOIN dw.dim_date dd     ON f.date_id = dd.date_id
WHERE dd.year = 2026
GROUP BY dc.label
ORDER BY ann_volatility DESC;
```

**Data warehouse schema:**

![ERD and star schema for the data warehouse](Data%20Storing/Data%20Warehouse/design/star_schema.png)

**Analytical query evidence:**

![Data warehouse query screenshot](Data%20Storing/Data%20Warehouse/screenshots/query_1.png)

### 9.2 Automated Scheduling

Automated scheduling is implemented with Apache Airflow through three DAGs with different schedules. Each schedule is based on the actual update frequency of the source data, which can be observed through timestamp fields in the API response. For example, NAV has a `LastUpdate` field that changes on each trading day, while investment manager data has a `DataLastUpdate` field that last changed several years ago.

| DAG | Cron schedule | Scope |
|---|---|---|
| `reksa_dana_daily_nav` | `0 20 * * 1-5` on trading days at 20:00 WIB | NAV, performance, ranking, and benchmark data |
| `reksa_dana_monthly` | `0 6 2 * *` on the second day of each month | Fund AUM, investment manager AUM, portfolio, new securities, and quarterly returns |
| `reksa_dana_master` | `0 3 * * 0` early Sunday morning | Investment managers, custodian banks, selling agents, fund details, and fund classes |

DAG code is available in `airflow/dags/`. Each load uses `ON CONFLICT ... DO NOTHING` for time-series tables such as `nav_record`, `aum_record`, and `portfolio_snapshot`, because existing rows do not change and only new rows need to be added. Master tables such as `investment_manager`, `mutual_fund`, and `custodian_bank` use `ON CONFLICT ... DO UPDATE` because attributes such as fees and addresses can change. This prevents duplicate data during scheduled runs.

**Scheduling evidence.** Differences in `scraped_at` values between batches demonstrate that data is updated periodically without duplicate rows.

```sql
SELECT record_date,
       COUNT(*)        AS fund_count,
       MIN(scraped_at) AS batch_start,
       MAX(scraped_at) AS batch_end
FROM nav_record
GROUP BY record_date
ORDER BY record_date DESC
LIMIT 10;
```

The following result came from two different `reksa_dana_daily_nav` DAG runs: `scheduled__2026-07-22T13:00:00+00:00` and `manual__2026-07-23T08:54:41`. The logs are available in `airflow/logs/dag_id=reksa_dana_daily_nav/`.

```
 record_date | fund_count |          batch_start          |           batch_end
-------------+------------+-------------------------------+-------------------------------
 2026-07-22  |       1475 | 2026-07-23 09:43:00.774153+00 | 2026-07-23 09:43:00.774153+00
 2026-07-21  |       1476 | 2026-07-23 09:43:00.774153+00 | 2026-07-23 09:43:00.774153+00
 2026-07-20  |       1476 | 2026-07-23 09:43:00.774153+00 | 2026-07-23 09:43:00.774153+00
 2026-07-17  |       1476 | 2026-07-23 08:06:30.684527+00 | 2026-07-23 09:43:00.774153+00
 2026-07-16  |       1478 | 2026-07-23 08:06:30.684527+00 | 2026-07-23 08:06:30.684527+00
 2026-07-15  |       1483 | 2026-07-23 08:06:30.684527+00 | 2026-07-23 08:06:30.684527+00
```

How to read the result:

- The rows for `2026-07-20` through `2026-07-22` were created only by the second batch, where `batch_start` and `batch_end` are both `09:43:00`. These are new NAV dates that did not exist when the first batch ran, so they were added rather than overwritten.
- The `2026-07-17` row has a `batch_start` of `08:06:30` from the first batch and a `batch_end` of `09:43:00` from the second batch. This means that some funds for that date were loaded in the first batch and the second batch only added funds that were previously missing, without duplicating existing rows.
- The fund count for each date never increases by a multiple even though the DAG was run more than once, demonstrating that the process does not create redundant data.

### 9.3 Query Optimization

Three queries were optimized using `EXPLAIN ANALYZE` measurements before and after optimization. The outputs remained identical while execution became faster. As required, all optimization queries are collected in the `Query Optimasi/` folder at the repository root. The queries are stored in `Query Optimasi/query_optimasi.sql`, with SQL comments explaining the purpose of each query: a composite index for the latest NAV per fund, an index for portfolio instrument lookups, and a materialized view for the NAV and return dashboard.

**Optimization evidence 1: index for the latest NAV per fund**

![Optimization evidence 1](Query%20Optimasi/optimasi-1.png)

**Optimization evidence 2: portfolio instrument lookup index**

![Optimization evidence 2](Query%20Optimasi/optimasi-2.png)

In optimization screenshot 2, the `idx_phold_security` index happened to be installed from an earlier experiment when the before measurement was taken. Therefore, both results used an index, with execution times of 7.174 ms and 6.976 ms, and the screenshot did not show a strong contrast. The valid comparison, after dropping the index before measuring the baseline, is shown in the summary table below.

**Optimization evidence 3: materialized view for the NAV and return dashboard**

![Optimization evidence 3](Query%20Optimasi/optimasi-3.png)

**Optimization results summary:**

| # | Optimization | Before | After | Plan change |
|---|---|---:|---:|---|
| 1 | Index `idx_nav_fund_date` on `(fund_id, record_date DESC)` | 176.9 ms | 64.8 ms, approximately 2.7 times faster | `Incremental Sort` becomes a direct `Index Scan` without sorting |
| 2 | Index `idx_phold_security` on `security_id` | 9.1 ms | 4.5 ms, approximately 2 times faster | An `Index Scan` on the less suitable `(snapshot_id, security_id)` order becomes a direct `Bitmap Index Scan` on `security_id` |
| 3 | Materialized view `mv_fund_latest_nav` | 420.5 ms | 0.344 ms, approximately 1,200 times faster | `WindowAgg`, which recalculates `LAG()` for every query, becomes a direct `Index Scan` that reads stored results |

All three optimizations produce output identical to the unoptimized queries. Both row counts and values are the same, with only the execution path changing. This was verified through `SELECT COUNT(*), MD5(...)` queries at the end of `query_optimasi.sql`.

---

## 10. Known Limitations

Several items are intentionally outside the scope because this project focuses on database modeling rather than production system operations.

- **Source data revisions are not tracked.** If pasardana revises historical NAV data after scraping, the OLTP model will not capture that revision. In the data warehouse, historical dimension attribute changes are still captured through SCD Type 2.
- **Fund lifecycle is not tracked.** Analyses that only include active funds may therefore have survivorship bias, because poorly performing funds are more likely to close or merge.
- **Some portfolio snapshots do not total exactly 100 percent.** This is a source data issue and is not caused by the pipeline.

Some gaps are addressed through other mechanisms. The `audit_log` table, populated through an OLTP trigger, records changes to master data. SCD Type 2 in the data warehouse, described in section 9.1, stores the history of dimension attributes over time.

---

## 11. AI Usage

In accordance with the project specification, this section describes how AI was used during development.

### 1. Areas assisted by AI

- Writing **code scripts**, including Python scraper, preprocessor, loader, and Airflow DAG code, as well as **SQL queries and DDL** for the OLTP schema, data warehouse schema, and query optimization. AI was used to translate designs that had already been prepared into code, speed up boilerplate writing, and assist with debugging.
- Preparing documentation, including this README and technical notes in `docs/`.

### 2. Areas completed independently

- **Design decisions**, including the ERD, relational schema, normalization decisions, and other database design choices.
- **API endpoint research**, including manually inspecting pasardana.id endpoints through DevTools under the Network tab, determining parameters and response structures, and deciding which endpoints were necessary.
- **Overall database design**, including entity modeling, cardinality, participation, and the selection of stored attributes versus derived attributes. AI helped express these decisions as code and DDL, but the design decisions were made independently.

### 3. Reflection on AI usage

AI significantly accelerated implementation work, including Python boilerplate, DDL, SQL queries, and documentation. This allowed more time to be spent on endpoint research and database design decisions. However, AI output still needed to be verified against the real data and running system rather than accepted without review. For example, one query optimization proof, optimization 2, initially appeared complete but was invalid after a second review because the comparison index had already been installed from an earlier experiment. As a result, both the before and after baselines were fast and did not show the intended contrast. This issue was discovered by checking the database directly rather than relying only on the existing screenshot. The main lesson is that AI accelerates writing and documentation, but verification against real results must still be performed before a result is considered correct.

### 4. Additional details

- AI tool used: **Claude Code**.
- Estimated proportion of work involving AI: most Python and SQL script writing, including DDL, queries, and optimization queries. The entire design process, including the ERD and database schema, as well as API endpoint research, was completed manually.

---

## 12. References

- Data source: [pasardana.id](https://pasardana.id)
- [Playwright](https://playwright.dev/python/) for browser automation and session handling.
- [pandas](https://pandas.pydata.org/) for data transformation.
- [psycopg2](https://www.psycopg.org/) for PostgreSQL connectivity.
- PostgreSQL 16, run through Podman or Docker Compose.
- Apache Airflow for automated scheduling in the bonus implementation.
