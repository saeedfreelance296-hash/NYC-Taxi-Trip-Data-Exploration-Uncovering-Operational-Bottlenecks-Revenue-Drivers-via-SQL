# NYC Yellow Taxi Q1 2022 — Exploratory Data Analysis (EDA) & Data Pipeline Log

## 📌 Executive Summary
This project performs an end-to-end Exploratory Data Analysis (EDA) and data quality audit on **9+ million records** of New York City Yellow Taxi trip data covering Q1 2022 (January – March). 

The goal of this analysis is to evaluate data integrity, audit schema definitions, identify operational glitches, optimize relational engine performance for big data workloads, and establish a foundational data layer for business intelligence dashboarding.

---

## ⚡ Performance Optimization & Engine Strategy

Running analytical aggregations (`SUM`, `AVG`, `GROUP BY`, window functions) across 9+ million rows on raw, unindexed row-store tables causes severe I/O bottlenecks and high query latency. 

To transform this database from transactional storage to an analytical engine, a two-tier indexing strategy was executed:

### 1. Clustered Columnstore Index (CCI) on Fact Table
Applied a **Clustered Columnstore Index** to `dbo.yellow_tripdata_2022`. 

Columnstore reorganizes physical data storage from row-by-row into column-based rowgroups (~1 million rows each), enabling **Batch Execution Mode** and high-ratio data compression.

```sql
-- Convert fact table from row-store layout to columnar analytical storage
CREATE CLUSTERED COLUMNSTORE INDEX CCI_yellow_tripdata_2022
ON dbo.yellow_tripdata_2022;
GO
```

#### Optimization Benefits & Key Metrics:
* **Query Latency:** Multi-column aggregation execution time dropped from **10+ seconds** down to **sub-second / millisecond response times**.
* **Disk Footprint Compression:** Reduced storage requirements significantly via columnar dictionary compression.
* **Segment Elimination & Column Pruning:** SQL Server skips non-relevant compressed row groups when filtering date ranges, reading only the specific columns queried.

### 2. Clustered Primary Key (B-Tree) on Dimension Table
Established a clustered primary key on `dbo.taxi_zone_lookup` to optimize dimension hash/merge joins.

```sql
-- Enforce non-nullability and apply primary key constraint
ALTER TABLE dbo.taxi_zone_lookup
ALTER COLUMN location_id INT NOT NULL;
GO

ALTER TABLE dbo.taxi_zone_lookup
ADD CONSTRAINT PK_taxi_zone_lookup PRIMARY KEY CLUSTERED (location_id);
GO
```

#### Engineering & Schema Resolution:
* **Schema Fix:** During CSV/Parquet import, SQL Server assigned default nullable status (`IS_NULLABLE = YES`) to all lookup columns, blocking primary key creation (`cannot create on nullable columns`).
* **Resolution:** Explicitly altered `location_id` to `NOT NULL` prior to applying `PRIMARY KEY CLUSTERED`, establishing fast hash join mechanics with fact table keys (`pu_location_id` / `do_location_id`).

---

## 1. Database & Schema Exploration

### Q1.1: Record Footprint & Geographic Coverage
* **Total Trip Records Loaded:** `9,000,000+` (Jan – Mar 2022)
* **Active Dispatch Vendors:** `4` vendors (Vendor 1 = Creative Mobile Technologies, Vendor 2 = VeriFone Inc.)
* **Unique Active Pickup Zones (`pu_location_id`):** `260` out of 265 official NYC Taxi Zones
* **Unique Active Drop-off Zones (`do_location_id`):** `261` out of 265 official NYC Taxi Zones

#### Analytical Takeaways:
1. **Asymmetric Flow:** Drop-off coverage (261 zones) slightly exceeds pickup coverage (260 zones). This reflects NYC Taxi and Limousine Commission (TLC) street-hail regulations—Yellow Taxis can drop off passengers anywhere across all 5 boroughs, but pickups remain heavily concentrated in Manhattan and major airport terminals (JFK / LaGuardia).
2. **Inactive Outer Zones:** The 4 to 5 missing zones in Q1 2022 correspond to outer service regions or administrative codes (e.g., `location_id` 264 and 265 for "Unknown/NV") where roadside street-hail demand is non-existent.

### Q1.2: Schema Architecture & Data Type Audit

| Table Name | Column Name | Data Type | Nullable | Role / Business Context |
| :--- | :--- | :--- | :--- | :--- |
| **`taxi_zone_lookup`** | `location_id` | `INT` | **NO** | **Primary Key** |
| `taxi_zone_lookup` | `borough` | `NVARCHAR` | YES | Geographical borough name |
| `taxi_zone_lookup` | `zone` | `NVARCHAR` | YES | TLC Taxi Zone neighborhood |
| `taxi_zone_lookup` | `service_zone` | `NVARCHAR` | YES | Sub-region service grouping |
| **`yellow_tripdata_2022`** | `vendor_id` | `INT` | YES | Technology provider ID |
| `yellow_tripdata_2022` | `tpep_pickup_datetime` | `DATETIME2` | YES | Meter start timestamp |
| `yellow_tripdata_2022` | `tpep_dropoff_datetime` | `DATETIME2` | YES | Meter stop timestamp |
| `yellow_tripdata_2022` | `passenger_count` | `FLOAT` | YES | Passenger count (stored as float due to NULL handling) |
| `yellow_tripdata_2022` | `trip_distance` | `FLOAT` | YES | Odometer trip distance in miles |
| `yellow_tripdata_2022` | `rate_code_id` | `FLOAT` | YES | Fare rate tier ID (stored as float) |
| `yellow_tripdata_2022` | `store_and_fwd_flag` | `VARCHAR` | YES | In-vehicle memory flag (`Y`/`N`) |
| `yellow_tripdata_2022` | `pu_location_id` | `INT` | YES | **Foreign Key** → `taxi_zone_lookup.location_id` |
| `yellow_tripdata_2022` | `do_location_id` | `INT` | YES | **Foreign Key** → `taxi_zone_lookup.location_id` |
| `yellow_tripdata_2022` | `payment_type` | `INT` | YES | Payment method code (1 = CC, 2 = Cash, etc.) |
| `yellow_tripdata_2022` | `fare_amount` | `DECIMAL` | YES | Base time/distance meter fare |
| `yellow_tripdata_2022` | `extra` | `DECIMAL` | YES | Peak hour / night rush surcharges |
| `yellow_tripdata_2022` | `mta_tax` | `DECIMAL` | YES | Mandatory MTA tax ($0.50) |
| `yellow_tripdata_2022` | `tip_amount` | `DECIMAL` | YES | Credit card tip amount |
| `yellow_tripdata_2022` | `tolls_amount` | `DECIMAL` | YES | Bridge and tunnel tolls |
| `yellow_tripdata_2022` | `improvement_surcharge`| `DECIMAL` | YES | TLC technology surcharge ($0.30) |
| `yellow_tripdata_2022` | `total_amount` | `DECIMAL` | YES | Total billed amount charged to passenger |
| `yellow_tripdata_2022` | `congestion_surcharge` | `DECIMAL` | YES | NYC congestion fee |
| `yellow_tripdata_2022` | `airport_fee` | `DECIMAL` | YES | Airport pickup fee |

### Q1.3: Referential Integrity Audit (Orphan Check)
* **Orphan Pickup Trips (`pu_location_id` unmatched):** `0` records
* **Orphan Drop-off Trips (`do_location_id` unmatched):** `0` records
* **Integrity Score:** **100.0%** clean link between fact trip records and dimension zones.

---

## 2. Dimension & Data Quality Audit

### Q2.1: Rate Code Distribution & Pricing Tier Analysis

| Rate Code | Pricing Description | Trip Count | Share (%) | Business & Operational Context |
| :--- | :--- | :--- | :--- | :--- |
| **1.0** | Standard Rate | 8,407,017 | **92%** | Core city street-hail revenue baseline across NYC |
| **NULL** | Unassigned / System Glitch | 291,055 | **3%** | Meter transmission failures or unmapped third-party dispatches |
| **2.0** | JFK Airport | 264,023 | **2%** | Flat-rate airport transit runs |
| **5.0** | Negotiated Fare | 48,974 | **<1%** | Agreed-upon long-distance or driver-passenger special rates |
| **99.0** | Vendor Test / System Override | 32,477 | **<1%** | Vendor meter diagnostic logging and hardware testing |
| **3.0** | Newark Airport | 16,999 | **<1%** | Interstate airport transit (out-of-state fee structures) |
| **4.0** | Nassau / Westchester | 10,628 | **<1%** | Regional suburban transport outside NYC limits |
| **6.0** | Group Ride | 71 | **<1%** | Shared passenger trips |

#### Anomaly Log & Data Cleaning Protocol:
1. **Unassigned / NULL Rate Codes (291,055 trips / 3%):** Represents meter communication failures or automated third-party dispatches. In production reporting, these trips should be labeled as `"Unassigned Meter"` rather than discarded, as they hold valid monetary and location data.
2. **Vendor Diagnostic Code 99 (32,477 trips):** Confirmed as internal hardware diagnostic or manual vendor testing logs. These need to be flagged during rate-tier analysis to prevent skewing average standard trip metrics.
### Q2.2: Data Hygiene & Anomaly Audit (Dirty Data Log)

#### Part 1: Timestamp & Trip Duration Integrity

| Metric Audited | Audit Condition | Record Count | % of Dataset | Business & Analytical Context |
| :--- | :--- | :--- | :--- | :--- |
| **Missing Timestamps** | `pickup IS NULL OR dropoff IS NULL` | **0** | **0.00%** | **Perfect Ingestion:** 100% of trip records contain valid meter timestamps. |
| **Invalid Trip Durations** | `dropoff <= pickup` | **8,412** | **~0.09%** | **Meter Glitches / Cancellations:** Driver instantly canceled or clicked start/stop simultaneously, resulting in zero or negative trip duration. |

#### Data Cleaning Protocol:
* **Production Filter:** Exclude records where `tpep_dropoff_datetime <= tpep_pickup_datetime` when computing trip duration metrics (e.g., Average Speed, Trip Duration Distribution, Hourly Utilization) to prevent skewed duration baselines.
#### Part 2: Financial & Fare Anomalies

| Metric Audited | Audit Condition | Record Count | % of Dataset | Business & Analytical Context |
| :--- | :--- | :--- | :--- | :--- |
| **Zero / Negative Total Fares** | `total_amount <= 0` | **51,914** | **~0.58%** | **Reversals & Disputes:** Customer chargebacks, disputed rides, driver cancellations, or voided transactions. |
| **Unbilled Distance** | `fare_amount <= 0 AND trip_distance > 0` | **43,643** | **~0.48%** | **Meter Errors / Overrides:** Vehicle traveled logged miles, but base meter fare registered zero or negative. |

#### Data Cleaning Protocol:
* **Financial Analysis:** Filter out `total_amount <= 0` when computing revenue totals, average fare per trip, or tipping percentages to avoid understating actual revenue.
* **Operational Auditing:** Keep unbilled distance records flagged separately for operational review to evaluate vendor meter hardware reliability.

### Q2.3: Payment Method Distribution & Channel Breakdown

| Payment Type | Method Description | Trip Count | Share (%) | Business & Operational Context |
| :--- | :--- | :--- | :--- | :--- |
| **1** | Credit Card | 6,956,059 | **76.68%** | Dominant payment method across NYC. Provides automated, verifiable tipping records for driver earning analysis. |
| **2** | Cash | 1,745,160 | **19.24%** | Secondary payment method (~1 out of 5 rides). Tip amounts are rarely logged at the meter, understating true cash tips. |
| **0** | Unmapped / NULL | 291,055 | **3.21%** | Corresponds directly to the 3% unassigned rate codes (meter sync errors / third-party dispatch disconnections). |
| **3** | No Charge | 40,417 | **0.45%** | Promotional rides, administrative/courtesy trips, or driver error overrides. |
| **4** | Dispute | 38,550 | **0.42%** | Contested charges or rider payment refusal at drop-off. High correlation with negative fare records. |
| **5** | Unknown | 3 | **<0.01%** | Edge case system anomaly. |

#### Analytical Insights & Pipeline Impact:
1. **Tip Analysis Separation:** Because **19.24% of trips are cash** where tips are rarely entered into the meter, tipping metrics (such as Average Tip % or Tip-to-Fare Ratio) **must be calculated exclusively on Credit Card transactions (`payment_type = 1`)**. Blending cash rides into tip calculations artificially depresses average tip figures.
2. **Revenue Loss via Disputes & No-Charge:** Over **78,000 trips (~0.87%)** ended as `Dispute` or `No Charge`, providing a clear target for operational fraud/loss auditing.

### Q3.1: Temporal Boundaries & Outlier Detection
* **Target Date Range:** January 1, 2022 – March 31, 2022 (Q1 2022)
* **Out-of-Range Records Detected:** `110` trips (<0.01% of dataset)
* **Outlier Context:** Erroneous timestamps spanning outside Q1 2022 caused by meter hardware clock resets or uncalibrated terminal dates.
* **Cleaning Action:** Filter strictly for `tpep_pickup_datetime >= '2022-01-01' AND tpep_pickup_datetime < '2022-04-01'` in all downstream temporal queries.

  ### Q3.2 & Q3.3: Hourly Demand Distribution & Trip Duration Profile

| Pickup Hour | Total Trips | Daily Share (%) | Avg Duration (Mins) | Operational Phase & Demand Context |
| :---: | :---: | :---: | :---: | :--- |
| **00:00 (12 AM)** | 231,637 | 2.56% | 14.64 | Late Night Transit / Bar & Event Off-peak |
| **01:00 (1 AM)** | 155,806 | 1.72% | 14.28 | Overnight Low Demand |
| **02:00 (2 AM)** | 104,379 | 1.15% | 14.04 | Overnight Low Demand |
| **03:00 (3 AM)** | 72,498 | 0.80% | 14.28 | System Trough (Lowest daily volume) |
| **04:00 (4 AM)** | 47,521 | 0.52% | 15.49 | Early Airport & Inter-Borough Runs |
| **05:00 (5 AM)** | 52,346 | 0.58% | 15.27 | Early Morning Shift Commute |
| **06:00 (6 AM)** | 137,835 | 1.52% | 15.35 | Morning Commute Kickoff |
| **07:00 (7 AM)** | 272,480 | 3.01% | 15.87 | AM Peak Surge |
| **08:00 (8 AM)** | 364,418 | 4.02% | 16.01 | Core Morning Business Rush |
| **09:00 (9 AM)** | 393,606 | 4.34% | 14.97 | Mid-Morning Corporate Transit |
| **10:00 (10 AM)** | 427,306 | 4.72% | 15.16 | Mid-Day Commercial Demand |
| **11:00 (11 AM)** | 459,700 | 5.07% | 15.26 | Mid-Day Business Inter-Zone Travel |
| **12:00 (12 PM)** | 501,767 | 5.54% | 15.48 | Lunch Hour Volume Build |
| **13:00 (1 PM)** | 519,299 | 5.73% | 16.00 | Afternoon Business Travel |
| **14:00 (2 PM)** | 565,465 | 6.24% | 17.33 | Early Afternoon Traffic Congestion |
| **15:00 (3 PM)** | 593,309 | 6.55% | 17.95 | **Peak Trip Duration (17.95 min)** / Driver Shift Change |
| **16:00 (4 PM)** | 588,318 | 6.49% | 17.57 | Early PM Rush Hour |
| **17:00 (5 PM)** | 637,778 | 7.04% | 16.23 | Evening Rush Surge |
| **18:00 (6 PM)** | 667,030 | **7.36%** | 15.04 | **Daily Demand Peak (667k trips)** / Evening Commute & Dining |
| **19:00 (7 PM)** | 586,438 | 6.47% | 14.26 | Evening Entertainment Transit |
| **20:00 (8 PM)** | 477,789 | 5.27% | 14.05 | Nighttime Leisure & Dining |
| **21:00 (9 PM)** | 450,507 | 4.97% | 13.74 | **Fastest Traffic Flow (13.74 min)** / Evening Off-Peak |
| **22:00 (10 PM)** | 422,643 | 4.66% | 14.21 | Late Night Hospitality Demand |
| **23:00 (11 PM)** | 332,808 | 3.67% | 14.52 | Late Night Transit |

#### Key Operational Insights:
1. **Daily Volume Peak (18:00 / 6 PM):** Highest ride concentration of the day with **667,030 trips (7.36% of total volume)** as corporate office departures merge with evening leisure traffic.
2. **Congestion Bottleneck Peak (15:00 / 3 PM):** Reaches the longest average duration of **17.95 minutes per trip**. This reflects the daily TLC shift changeover window combined with mid-town afternoon street traffic.
3. **Traffic Speed Efficiency Window (21:00 / 9 PM):** Average trip duration drops to a daily low of **13.74 minutes**, indicating minimal street friction and faster transit times across Midtown/Downtown.
