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
