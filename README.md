#  NYC Yellow Taxi Revenue Optimization & Spatial Demand Analysis (Q1 2022)
### *A T-SQL & Power BI Exploratory Analysis of 8.9M+ Trips Across Jan 1 – Mar 31, 2022 to Identify High-Yield Corridors & Fleet Dispatch Inefficiencies*
---

## Executive Summary: Top 3 Business Findings

> **Core Objective:** Analyze 8,925,374 NYC Yellow Taxi trips from **Jan 1 – Mar 31, 2022** using T-SQL to identify high-yield transit corridors, evaluate borough efficiency, and pinpoint driver dispatch bottlenecks.

1. **The Airport Single-Zone Dominance ($21.3M JFK Corridor):**
   * **JFK Airport** generated **$21,344,994.12** in gross revenue across 352,035 trips—making it the single highest-grossing zone in NYC. JFK alone generated more revenue than Manhattan’s top 3 commercial/residential zones combined ($18.96M).

2. **The Borough Efficiency Paradox (Density vs. Ticket Size):**
   * **Manhattan** drives the network's highest volume (8.1M trips) and revenue density (**$3.67/mile**), but yields a low average ticket size ($11.15 base fare). **Queens** operates on the opposite spectrum: delivering 3.4x higher ticket sizes (**$38.80 base fare**) driven by long-haul airport runs. Outer boroughs (**Brooklyn & Bronx**) suffer severe efficiency loss (**$0.44–$0.47/mile**) due to unpaid deadhead transit between rides.

3. **Pareto Revenue Concentration in High-Cost Trips:**
   * Segmenting the dataset into 4 cost quartiles (`NTILE(4)`) proves that the top 25% highest-cost rides (Q4: fares over $20.80) account for **48.6% of total market revenue ($86.33M)** with an average distance of 14.35 miles.
  *Visual Indicator:* **Tableau Dual-Axis Chart & Efficiency Matrix** (Compares Revenue per Mile against Average Fare per Borough).

## Business Problem & Project Goals

### Background
NYC Yellow Taxis processed over **8.9 million trips** in Q1 2022. However, fleet managers struggle with driver allocation, unpaid miles in outer boroughs, and understanding which routes make the most money.

### Core Business Questions
1. **Revenue Growth:** How did monthly revenue grow from January to March 2022, and which technology vendor made more money?
2. **Borough Efficiency:** Which boroughs give the best revenue per mile, and where are drivers losing money on long empty trips?
3. **Top Locations:** What are the top 5 highest-grossing zones in each borough? How do airport trips compare to regular city rides?
4. **Fare Distribution:** How much total revenue comes from short cheap rides versus long expensive rides?



---

3contribution by quartile).

---
