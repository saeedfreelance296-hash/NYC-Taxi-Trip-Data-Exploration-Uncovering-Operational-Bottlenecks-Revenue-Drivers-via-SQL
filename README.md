# NYC Taxi Trip Data Exploration: Uncovering Operational Bottlenecks & Revenue Drivers via SQL

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

### Q4.1: Summary Statistics & Operational Distribution

| Variable | Min | Avg | Max | Std Dev | Data Quality & Operational Takeaway |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Trip Distance (Miles)** | 0.01 | **5.77** | 348,798.53 | 593.91 | **Extreme Outlier Alert:** The average distance is ~5.77 miles, but extreme maximum outliers (~348k miles) indicate odometer hardware glitches or corrupted meter log transmissions. |
| **Base Fare Amount ($)** | $0.01 | **$13.39** | $401,092.32 | $134.83 | **Core Revenue Baseline:** The average base fare across valid non-negative trips is $13.39, with extreme multi-hundred-thousand dollar outliers skewing standard deviation. |
| **Total Amount ($)** | $0.31 | **$19.90** | $401,095.62 | $135.13 | **Gross Ticket Size:** Including surcharges, tolls, fees, and credit card tips, the true average total cost per trip is $19.90. |

#### Data Cleaning & Dashboard Filtering Protocol:
* **Metric Truncation / Winsorization:** To calculate true operational averages (e.g., Average Trip Distance or Revenue per Mile) without skewing business decisions, extreme outliers (e.g., `trip_distance > 500 miles` or `total_amount > $1,000`) must be trimmed using percentile boundaries (e.g., 99.9th percentile) or operational capping filters in downstream SQL views.

### Q4.2: Credit Card vs. Cash Tipping Behavior Analysis

| Payment Description | Total Tips ($) | Avg Tip ($) | Total Base Fare ($) | Effective Tip Ratio (%) | Analytical & Operational Context |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Credit Card** | **$22,005,628.87** | **$3.16** | **$93,129,144.35** | **23.63%** | **In-Cab Terminal Prompts:** Digital payment readers offer preset tip options (15%, 20%, 25%), capturing an effective 23.63% gratuity relative to base fare. |
| **Cash** | $1,235.14 | $0.00 | $22,644,580.57 | **0.01%** | **Unrecorded Cash Gratuities:** Drivers receive cash tips directly without logging them into the meter terminal, resulting in an artificial near-zero tip record ($1,235 total across 1.7M trips). |

#### Analytical Takeaway & Pipeline Rule:
* **Metric Isolation:** Any KPI calculation assessing driver tip earnings, gratuity percentages, or total compensation **must filter exclusively for `payment_type = 1` (Credit Card)**. Blending cash records into overall tipping averages falsely depresses overall tips and misleads revenue reporting.

### Q4.3: Total Revenue Stream & Fee Breakdown Analysis

| Revenue Component | Aggregate Value ($) | Share of Total Revenue (%) | Strategic & Operational Context |
| :--- | :--- | :--- | :--- |
| **Base Fare Amount** | ~$122,200,560.77 | **67.58%** | **Core Service Value:** Metered distance/time charges form the foundational 2/3 of all TLC gross receipts. |
| **Tips (Gratuities)** | ~$22,946,509.56 | **12.69%** | **Primary Driver Earnings Driver:** Digital credit card tipping accounts for over an eighth of total collected revenue. |
| **Congestion Surcharge** | ~$20,270,320.90 | **11.21%** | **Regulatory Surcharge:** NYC MTA congestion fee levied on trips traversing Manhattan below 96th Street. |
| **Tolls Amount** | ~$3,851,541.80 | **2.13%** | **Infrastructure Reimbursables:** Pass-through toll collections primarily for inter-borough bridges, tunnels, and airport access. |
| **Gross Total Revenue** | **$180,823,558.40** | **100.00%** | **Q1 2022 Gross Benchmark:** Total top-line gross dollar intake across all valid trips. |

#### Financial Takeaway:
* Non-fare components (tips, congestion fees, tolls, and mandatory surcharges) make up **over 32% of total passenger expenditure**.

  ## 5. Magnitude & Spatial Revenue Analysis

### Q5.1: Volume and Gross Revenue by Pickup Borough

| Pickup Borough | Total Trips | Share of Trips (%) | Gross Revenue ($) | Share of Revenue (%) | Operational & Spatial Context |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Manhattan** | 8,202,816 | 89.96% | $139,599,459.05 | 77.54% | **Dominant Core Market:** Manhattan generates 9 out of 10 taxi pickups and over three-quarters of total gross revenue. |
| **Queens** | 684,345 | 7.50% | $34,617,929.42 | 19.23% | **Airport Revenue Surge:** Home to JFK and LaGuardia airports. High average fare per trip inflates its revenue share relative to volume. |
| **Unknown** | 87,491 | 0.96% | $2,376,823.22 | 1.32% | **Unmapped Terminals:** Pickups in unassigned or corrupted location IDs. |
| **Brooklyn** | 57,348 | 0.63% | $1,568,347.65 | 0.87% | Secondary outer-borough market heavily served by green taxis and rideshares rather than yellow cabs. |
| **N/A** | 24,777 | 0.27% | $1,294,173.81 | 0.72% | Uncategorized location zones. |
| **Bronx** | 12,552 | 0.14% | $411,647.19 | 0.23% | Low yellow taxi penetration zone. |
| **EWR (Newark)** | 1,337 | 0.01% | $127,010.01 | 0.07% | Out-of-state airport pickups. |
| **Staten Island**| 578 | 0.01% | $40,124.52 | 0.02% | Lowest demand volume in NYC. |
| **Total** | **9,118,244** | **100.00%** | **$180,035,514.87** | **100.00%** | **Q1 2022 Spatial Revenue Baseline** |

#### Spatial Takeaway:
* **Manhattan & Queens duopoly:** Together, Manhattan and Queens represent **97.46% of total NYC yellow cab revenue** ($174.2M out of $180M), confirming that yellow taxi fleet deployment should be concentrated almost exclusively in Manhattan corridors and airport transit routes.

  ### Q5.3: Borough Trip Efficiency & Revenue Density Analysis

| Pickup Borough | Gross Revenue ($) | Total Miles Driven | Avg Revenue / Mile ($) | Base Fare ($) | Total Rides | Avg Base Fare / Ride ($) | Operational Efficiency Context |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Manhattan** | $138,694,843.46 | 37,830,640.96 | **$3.67** | $90,362,611.62 | 8,107,786 | **$11.15** | **Dense Congestion Core:** High turnover, short trips (avg ~4.6 miles), and constant base fee pickups maximize revenue density per mile. |
| **Queens** | $34,108,882.38 | 9,499,979.43 | **$3.59** | $25,344,135.04 | 653,244 | **$38.80** | **High-Yield Airport Runs:** Long-haul trips to JFK/LaGuardia generate over 3x the average base fare per ride compared to Manhattan. |
| **Unknown** | $2,207,486.77 | 439,246.80 | **$5.03** | $1,564,439.43 | 81,586 | **$19.18** | **Unmapped Terminal Outliers:** Unassigned location IDs representing specialized transit zones. |
| **Brooklyn** | $1,343,695.60 | 2,858,626.25 | **$0.47** | $1,082,133.88 | 50,800 | **$21.30** | **Long Deadhead Outer Mileage:** High mileage driven relative to metered fares indicates significant non-metered transit between drop-offs and pickups. |
| **N/A** | $989,185.34 | 182,280.44 | **$5.43** | $953,975.09 | 21,831 | **$43.70** | Uncategorized zone logs showing high average ticket size. |
| **Bronx** | $290,689.86 | 655,322.76 | **$0.44** | $258,917.17 | 9,410 | **$27.52** | **Low Yellow Taxi Penetration:** Long distances between dispatched fares result in low overall revenue per mile driven ($0.44/mi). |
| **EWR (Newark)** | $31,217.33 | 1,059.86 | **$29.45** | $25,426.62 | 321 | **$79.21** | **Out-of-State Premium:** Out-of-state airport trips dominated by mandatory surcharges and premium flat rates. |
| **Staten Island**| $27,264.21 | 6,734.17 | **$4.05** | $23,099.63 | 396 | **$58.33** | **Low-Volume Long Distance:** Infrequent, high-ticket long-distance trips. |

#### Financial & Operational Takeaway:
* **Volume vs. Ticket Size Trade-off:** **Manhattan** drives total revenue through volume and density ($3.67/mile over 8.1M trips), whereas **Queens** provides the highest per-trip yield ($38.80 base fare/ride) driven by airport transit. 
* **Outer Borough Efficiency Drag:** Brooklyn and the Bronx exhibit low revenue per mile ($0.47 and $0.44 per mile), demonstrating that yellow cabs lose significant operational efficiency when operating outside Manhattan and major airport corridors.
  
### Step 6: Advanced Ranking & Window Functions

#### Q6.1: Top 5 Taxi Zones by Revenue per Borough (DENSE_RANK)

| Borough | Zone Rank | Taxi Zone | Total Trips | Gross Revenue ($) | Operational & Strategic Context |
| :--- | :---: | :--- | :---: | :---: | :--- |
| **Manhattan** | **#1** | **Upper East Side South** | 428,745 | **$6,479,343.35** | **Residential Core:** Dense residential corridor with constant daily commuter demand and short, high-turnover local rides. |
| | **#2** | Upper East Side North | 404,597 | $6,425,574.83 | High-density residential turnover adjacent to medical centers (NewYork-Presbyterian / Mount Sinai). |
| | **#3** | Midtown Center | 351,630 | $6,058,208.30 | Commercial business district driving prime weekday corporate transit. |
| | **#4** | Penn Station / Madison Sq West | 293,049 | $5,080,925.59 | Major multi-modal transit hub; heavy rail passenger connections. |
| | **#5** | Times Sq / Theatre District | 268,088 | $5,073,772.00 | Entertainment and tourism hub generating late afternoon and evening volume. |
| **Queens** | **#1** | **JFK Airport** | 352,035 | **$21,344,994.12** | **Top NYC Single Zone Revenue:** Flat-rate airport tariffs, high tolls, and heavy tipping generate over $21.3M in gross revenue. |
| | **#2** | LaGuardia Airport | 225,429 | $9,978,637.83 | Secondary airport hub driving high per-trip yields across short-to-mid distance routes. |
| | **#3** | East Elmhurst | 32,279 | $1,388,731.40 | Zone adjacent to LGA capturing overflow airport transit and staging activity. |
| | **#4** | Sunnyside | 7,056 | $178,217.30 | Queens-Manhattan bridge access corridor. |
| | **#5** | Baisley Park | 1,854 | $123,260.30 | Industrial/cargo staging corridor near JFK Airport. |
| **Brooklyn** | **#1** | **Downtown Brooklyn / MetroTech** | 5,228 | **$123,571.02** | Commercial and civic center anchoring outer-borough yellow cab demand. |
| | **#2** | Brooklyn Heights | 4,521 | $112,206.11 | High-income residential district with frequent Manhattan-bound trips. |
| | **#3** | Williamsburg (North Side) | 3,149 | $77,509.46 | Nightlife and dining hub generating late-night commuter traffic. |
| | **#4** | Boerum Hill | 3,427 | $77,166.84 | Residential commuter zone adjacent to Downtown Brooklyn. |
| | **#5** | Fort Greene | 3,399 | $75,155.13 | Cultural district (BAM) anchoring local intra-borough transit. |
| **Bronx** | **#1** | **Mott Haven / Port Morris** | 919 | **$21,662.18** | Industrial/commercial hub near South Bronx Manhattan bridge crossings. |
| | **#2** | Co-Op City | 475 | $18,839.28 | Massive residential complex generating long-distance borough-to-borough trips. |
| | **#3** | West Concourse | 660 | $15,142.91 | Yankee Stadium and civic center corridor. |
| | **#4** | Soundview / Castle Hill | 429 | $14,872.48 | Long-distance residential commutes. |
| | **#5** | Williamsbridge / Olinville | 360 | $14,633.84 | Outer Bronx residential corridor. |
| **EWR** | **#1** | **Newark Airport** | 321 | **$31,217.33** | Out-of-state airport destination with high surcharges (~$97 avg gross per trip). |
| **Staten Island** | **#1** | **Arden Heights** | 108 | **$9,384.95** | Low-density suburban commute zone. |

#### Key Analytical Insights:
1. **JFK Airport Single-Zone Dominance:** Gross revenue generated at JFK Airport ($21.34M) exceeds the top 3 Manhattan zones combined ($18.96M), proving that **airport flat-rate tariffs generate the highest revenue concentration** in the entire NYC taxi network.
2. **Manhattan Spatial Clustering:** The top 5 Manhattan zones alone account for **$29.1M (over 20% of total Manhattan revenue)**, split evenly between high-density residential hubs (Upper East Side) and transit/business centers (Midtown, Penn Station).

### Step 6: Advanced Ranking & Window Functions

#### Q6.2: Trip Cost Distribution & Distance Profiling (NTILE 4 Quartiles)

| Cost Quartile | Total Trips | Fare Range ($) | Avg Fare ($) | Avg Distance (mi) | Gross Revenue ($) | Revenue Share (%) | Operational & Fare Profiling Context |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **Q1 (Lowest)** | 2,231,344 | $0.04 – $11.80 | $9.94 | 1.13 | $22,179,996.10 | 12.5% | **Short Local Hoppers:** Very short intra-neighborhood trips heavily concentrated in dense Manhattan cores. |
| **Q2 (Mid-Low)** | 2,231,344 | $11.80 – $15.30 | $13.41 | 2.62 | $29,923,546.80 | 16.8% | **Standard Commutes:** Mid-town to lower/upper Manhattan cross-town rides. |
| **Q3 (Mid-High)** | 2,231,343 | $15.30 – $20.80 | $17.59 | 4.97 | $39,259,543.51 | 22.1% | **Longer Borough Routes:** Mid-distance commutes connecting upper Manhattan to outer borough edges. |
| **Q4 (Highest)** | 2,231,343 | $20.80 – $401,095.62* | $38.69 | 14.35 | $86,330,178.54 | 48.6% | **High-Yield & Long-Haul:** Airport runs (JFK/LGA), cross-borough transit, and long-distance out-of-town fares. |
| **Total / Overall** | **8,925,374** | **—** | **$19.91** | **5.77** | **$177,693,264.95** | **100.0%** | **Q1 2022 Cost Quartile Baseline** |

*\*Note: The maximum value of $401,095.62 reflects rare uncleaned meter log anomalies present in Q4 raw logs.*

#### Key Analytical Insights:
1. **Pareto Concentration in Q4:** The highest cost quartile (Q4) accounts for **48.6% of all gross revenue** (~$86.3M) despite making up exactly 25% of the total trip volume.
2. **Distance-to-Fare Scaling:** Average trip distance scales nearly 13x from Q1 (1.13 miles) to Q4 (14.35 miles), demonstrating that distance is the primary structural driver separating standard metered fares from high-yield long-haul and airport routes.

### Step 6: Advanced Ranking & Window Functions

#### Q6.3: Month-over-Month (MoM) Revenue Growth by Vendor (LAG Window Function)

| Vendor ID | Month | Quarter | Month Name | Current Month Revenue ($) | Previous Month Revenue ($) | MoM Growth (%) | Performance & Operational Context |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **1 (Creative Mobile Tech)** | 1 | Q1 | Jan | $13,920,989.78 | *NULL* | *N/A* | Baseline month for Q1 2022. |
| | 2 | Q1 | Feb | $16,684,808.16 | $13,920,989.78 | **+19.85%** | Strong post-holiday demand recovery across urban routes. |
| | 3 | Q1 | Mar | $20,923,460.57 | $16,684,808.16 | **+25.40%** | Peak Q1 acceleration driven by increased business commuting. |
| **2 (VeriFone Inc.)** | 1 | Q1 | Jan | $32,445,677.36 | *NULL* | *N/A* | Baseline month; holds primary market share (~70%+). |
| | 2 | Q1 | Feb | $40,431,839.34 | $32,445,677.36 | **+24.61%** | Substantial revenue gain across airport and high-turnover cores. |
| | 3 | Q1 | Mar | $52,407,716.50 | $40,431,839.34 | **+29.62%** | Highest single-month growth and total revenue contribution ($52.4M). |
| **5 (Specialty / Other)** | 1 | Q1 | Jan | $2,159.92 | *NULL* | *N/A* | Low-volume niche or test terminal logging. |
| | 2 | Q1 | Feb | $1,834.31 | $2,159.92 | **-15.08%** | Volume contraction on specialty rides. |
| | 3 | Q1 | Mar | $762.10 | $1,834.31 | **-58.45%** | Sharp drop-off in terminal logging activity. |
| **6 (Specialty / Other)** | 1 | Q1 | Jan | $212,038.93 | *NULL* | *N/A* | Baseline month for secondary technology supplier. |
| | 2 | Q1 | Feb | $285,967.41 | $212,038.93 | **+34.87%** | High percentage surge on modest baseline. |
| | 3 | Q1 | Mar | $376,010.57 | $285,967.41 | **+31.49%** | Consistent expansion through late Q1. |

#### Key Analytical Insights:
1. **Accelerating Market Expansion:** Both primary vendors (**Vendor 1** and **Vendor 2**) experienced compounding MoM revenue growth each month in Q1, moving from **~20–24% growth in February** up to **~25–29% growth in March**.
2. **Vendor Market Share Dominance:** **Vendor 2 (VeriFone)** generated **$125.28M total in Q1** compared to **Vendor 1's $51.53M**, capturing over **70% of total vendor revenue** while maintaining slightly faster MoM growth trajectories.
