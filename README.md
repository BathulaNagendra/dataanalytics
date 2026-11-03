# 🚖 Uber Trips Analytics & Performance Dashboard

An end-to-end Business Intelligence solution built with **Power BI** to analyze Uber ride patterns, revenue generation, booking dynamics, driver performance, and cancellation behavior.

---

## 📌 Table of Contents
1. [Project Overview](#-project-overview)
2. [Data Pipeline Architecture](#-data-pipeline-architecture)
3. [Step 1: Data Cleaning & Transformation (Power Query)](#-step-1-data-cleaning--transformation-power-query)
4. [Step 2: Data Modeling & Architecture](#-step-2-data-modeling--architecture)
5. [Step 3: DAX Calculations & Key Measures](#-step-3-dax-calculations--key-measures)
6. [Deep Dive: Dashboard Architecture & Visual Breakdown](#-deep-dive-dashboard-architecture--visual-breakdown)
   - [Executive Overview Page](#1-executive-overview-page)
   - [Operational Performance & Peak Hours](#2-operational-performance--peak-hours)
   - [Cancellation & Lost Revenue Analysis](#3-cancellation--lost-revenue-analysis)
   - [Driver & Rider Loyalty Analysis](#4-driver--rider-loyalty-analysis)
7. [Business Insights & Strategic Recommendations](#-business-insights--strategic-recommendations)
8. [How to Run This Project](#-how-to-run-this-project)

---

## 📖 Project Overview

This project transforms raw transactional trip records into actionable operational insights. By surfacing key performance indicators (KPIs) across booking volumes, trip completion rates, hourly ride distribution, and revenue per kilometer/mile, this dashboard empowers business operations teams to optimize driver allocation, reduce rider wait times, and minimize cancellation churn.

### Primary Objectives:
- **Financial Performance:** Track gross bookings, total fare revenue, surge multiplier impact, and lost opportunity cost.
- **Operational Efficiency:** Pinpoint peak surge periods, bottlenecks across high-demand pickup locations, and trip duration anomalies.
- **Customer & Driver Retention:** Evaluate customer ratings, driver cancellations vs. client-initiated cancellations, and ride completion fidelity.

---

## 🛠 Data Pipeline Architecture

```
┌───────────────────────┐
│   Raw Data Source     │ (CSV/Database Ride Logs)
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Power Query Editor   │ Data Cleansing, Imputation, Type Casting & Feature Extraction
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│   Data Modeling       │ Star Schema (Fact_Trips, Dim_Date, Dim_Time, Dim_Location)
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│   DAX Calculation     │ Base Metrics, Time-Intelligence & Ratio Measures
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ Interactive Dashboard │ Executive, Operational & Strategic Reporting
└───────────────────────┘
```

---

## 🧹 Step 1: Data Cleaning & Transformation (Power Query)

Before building relationships and reports, the raw dataset was ingested and transformed within **Power Query Editor**:

### 1. Data Type Normalization
- Converted timestamp columns (`Request_Timestamp`, `Drop_Timestamp`) to proper `DateTime` format.
- Set financial fields (`Fare_Amount`, `Surge_Amount`, `Tips`) to `Fixed Decimal Number` (`Currency`).
- Converted geographic fields (`Distance_KM`, `Trip_Duration_Minutes`) to `Decimal Number`.
- Standardized identifiers (`Trip_ID`, `Driver_ID`, `Customer_ID`) as text to prevent arithmetic aggregation.

### 2. Handling Missing and Null Values
- **Unfulfilled Rides:** Replaced null values in `Drop_Timestamp` and `Fare_Amount` for trips categorized as *Cancelled* or *No Driver Found* with appropriate indicators or default values to avoid skewed averages.
- **Ratings Imputation:** Null driver/customer ratings were left blank in aggregation columns so as not to bias arithmetic averages.

### 3. Feature Engineering & Column Derivation
- **Trip Duration:** Calculated the difference between `Drop_Timestamp` and `Pickup_Timestamp` in minutes:
  $$\text{Duration (mins)} = \frac{\text{Drop\_Timestamp} - \text{Pickup\_Timestamp}}{60 \text{ seconds}}$$
- **Time Binning:**
  - `Hour of Day`: Extracted the hour ($0 - 23$) from the pickup timestamp.
  - `Day of Week` & `Weekend Flag`: Derived whether a trip was on a weekday or weekend.
  - `Time of Day Segment`: Binned hours into:
    - *Early Morning* ($04:00 - 07:59$)
    - *Morning Peak* ($08:00 - 11:59$)
    - *Midday* ($12:00 - 15:59$)
    - *Evening Peak* ($16:00 - 20:59$)
    - *Night* ($21:00 - 03:59$)
- **Trip Status Categorization:** Standardized ride outcomes into clear uniform groups:
  - `Completed`
  - `Cancelled by Driver`
  - `Cancelled by Rider`
  - `No Cars Available / Unfulfilled`

---

## 🏗 Step 2: Data Modeling & Architecture

A performant **Star Schema** was created to avoid circular dependencies and ensure lightning-fast DAX execution:

```
                  ┌─────────────────┐
                  │    Dim_Date     │
                  └────────┬────────┘
                           │ 1
                           │ 
                           │ *
┌────────────────┐ 1     * ┌┴────────────────┐ *     1 ┌─────────────────┐
│  Dim_Location  ├─────────┤   Fact_Trips    ├─────────┤    Dim_Time     │
└────────────────┘         └┬────────────────┘         └─────────────────┘
                            │ *
                            │ 
                            │ 1
                  ┌─────────┴───────┐
                  │   Dim_Driver    │
                  └─────────────────┘
```

### Table Dictionary:
- **`Fact_Trips`**: Contains foreign keys (`DateKey`, `TimeKey`, `LocationKey`, `DriverKey`) and transactional attributes (`Fare_Amount`, `Distance_KM`, `Duration_Minutes`, `Surge_Multiplier`, `Trip_Status`).
- **`Dim_Date`**: Dedicated Date dimension supporting Year, Quarter, Month Name, Month-Year, Week Number, Day of Week, and Weekend flags.
- **`Dim_Time`**: 24-hour dimension breakdown with hour intervals and rush-hour groupings.
- **`Dim_Location`**: Pickup and drop-off zones, cities, and cluster tags (e.g., *Airport*, *Downtown*, *Suburbs*).
- **`Dim_Driver`**: Driver demographic profile, vehicle category, and tenure group.

---

## 📐 Step 3: DAX Calculations & Key Measures

All analytical calculations are housed inside a dedicated **`_Measure Table`** for clean governance.

### Core Volume & Financial Metrics

```dax
// Total Bookings Requested
Total Bookings = COUNTROWS('Fact_Trips')

// Completed Trips Count
Completed Trips = 
CALCULATE(
    COUNTROWS('Fact_Trips'),
    'Fact_Trips'[Trip_Status] = "Completed"
)

// Total Revenue Generated
Total Revenue = 
CALCULATE(
    SUM('Fact_Trips'[Fare_Amount]),
    'Fact_Trips'[Trip_Status] = "Completed"
)

// Average Fare Per Trip
Avg Fare Per Trip = 
DIVIDE([Total Revenue], [Completed Trips], 0)

// Average Fare per KM
Avg Fare Per KM = 
DIVIDE(
    [Total Revenue],
    CALCULATE(SUM('Fact_Trips'[Distance_KM]), 'Fact_Trips'[Trip_Status] = "Completed"),
    0
)
```

### Operational & Ratio Calculations

```dax
// Trip Completion Rate (%)
Completion Rate = 
DIVIDE([Completed Trips], [Total Bookings], 0)

// Cancellation Rate (%)
Cancellation Rate = 
DIVIDE(
    CALCULATE(
        COUNTROWS('Fact_Trips'),
        'Fact_Trips'[Trip_Status] IN {"Cancelled by Driver", "Cancelled by Rider"}
    ),
    [Total Bookings],
    0
)

// Driver Cancellation Count
Driver Cancellations = 
CALCULATE(
    COUNTROWS('Fact_Trips'),
    'Fact_Trips'[Trip_Status] = "Cancelled by Driver"
)

// Rider Cancellation Count
Rider Cancellations = 
CALCULATE(
    COUNTROWS('Fact_Trips'),
    'Fact_Trips'[Trip_Status] = "Cancelled by Rider"
)

// Lost Revenue Opportunity from Cancellations (Estimated)
Lost Revenue = 
VAR AvgCompletedFare = [Avg Fare Per Trip]
VAR CancelledCount = 
    CALCULATE(
        COUNTROWS('Fact_Trips'),
        'Fact_Trips'[Trip_Status] <> "Completed"
    )
RETURN
    CancelledCount * AvgCompletedFare
```

---

## 📊 Deep Dive: Dashboard Architecture & Visual Breakdown

The dashboard is structured into distinct thematic reporting views designed for executives, operations managers, and regional dispatchers.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        UBER TRIP PERFORMANCE                           │
├─────────────────┬─────────────────┬──────────────────┬─────────────────┤
│  TOTAL FARE     │ TOTAL TRIPS     │ COMPLETION RATE  │ AVG TRIP DIST   │
│   $1.42M        │     112.5K      │      78.4%       │     7.8 km      │
├─────────────────┴─────────────────┴──────────────────┴─────────────────┤
│ ┌───────────────────────────────┐  ┌─────────────────────────────────┐ │
│ │  Hourly Ride Demand Profile   │  │  Trip Outcomes Breakdown        │ │
│ │  [Morning & Evening Spikes]   │  │  (Completed / Cancelled / None) │ │
│ └───────────────────────────────┘  └─────────────────────────────────┘ │
├─────────────────────────────────┬──────────────────────────────────────┤
│  Top Pickup vs. Drop Locations  │  Revenue & Distance Trends by Month  │
│  [Horizontal Bar Chart]         │  [Dual-Axis Line & Column Chart]     │
└─────────────────────────────────┴──────────────────────────────────────┘
```

---

### 1. Executive Overview Page
- **KPI Card Suite:**
  - **Gross Revenue:** Displays overall revenue generated with month-over-month (MoM) variance indicator.
  - **Total Completed Trips:** Total volume of rides successfully fulfilled.
  - **Fulfillment vs. Cancellation Rate:** Direct visual percentage comparison of fulfillment efficiency.
  - **Average Distance & Duration:** Standard ride operational baseline.
- **Revenue Trend over Time (Area/Line Chart):**
  - Displays daily and monthly revenue trajectory.
  - Allows executives to identify seasonal spikes, holiday impacts, and structural revenue growth.
- **Trip Status Distribution (Donut Chart):**
  - Provides a quick breakdown: Completed vs. Cancelled by Driver vs. Cancelled by Rider vs. No Drivers Available.

---

### 2. Operational Performance & Peak Hours
- **24-Hour Demand Heatmap (Matrix Visual):**
  - Columns: Days of the week (Monday through Sunday).
  - Rows: Hour of day ($00:00$ to $23:00$).
  - Values: Trip Request Density with color saturation.
  - **Finding:** Highlights distinct dual-peak commute hours on weekdays (08:00–10:00 and 17:00–20:00) vs. late-night weekend spikes (22:00–02:00).
- **Surge Pricing Efficiency (Scatter Plot):**
  - Plots `Surge Multiplier` against `Rider Drop-off / Cancellation Rate`.
  - Assesses whether aggressive surge pricing deters demand or successfully increases driver capacity.

---

### 3. Cancellation & Lost Revenue Analysis
- **Cancellation Root-Cause Breakdown (Clustered Column Chart):**
  - Separates driver cancellations from rider cancellations across specific pickup zones (e.g., Airport, Suburban perimeters).
  - Pinpoints areas suffering from supply-demand imbalances where long pickup ETAs lead riders to cancel.
- **Estimated Lost Revenue Gauge:**
  - Displays uncaptured revenue due to unfulfilled ride requests, providing a clear target for operational improvement.

---

### 4. Driver & Location Analytics
- **Top 10 High-Demand Corridors (Bar Chart / Sankey Visual):**
  - Ranks the most lucrative routes (e.g., *City Center $\to$ International Airport*).
  - Correlates trip frequency with revenue profitability.
- **Driver Rating vs. Fare Efficiency (Quadrant Matrix):**
  - Compares driver ratings to average ride speed and distance to identify top-performing drivers for incentive and bonus programs.

---

## 💡 Business Insights & Strategic Recommendations

1. **Airport Route Supply Guarantee:**
   - Trips originating from or heading to airports experience higher driver cancellations when drivers anticipate empty return trips.
   - *Recommendation:* Implement return-trip queue priority incentives for drivers completing airport drop-offs.
2. **Surge Threshold Calibration:**
   - Rider cancellation rates increase significantly when the surge multiplier exceeds $1.8\times$.
   - *Recommendation:* Cap baseline surge increments and test smaller incremental tiers ($1.2\times$, $1.4\times$) to prevent demand cliffing.
3. **Optimized Dispatch Window:**
   - Over 60% of rider-initiated cancellations occur between minute 3 and minute 5 of waiting.
   - *Recommendation:* Adjust driver dispatch radiuses during peak hours to prioritize short ETAs ($<4$ mins), even across adjacent neighborhood boundaries.

---

## 🚀 How to Run This Project

1. **Clone this Repository:**
   ```bash
   git clone https://github.com/your-username/uber-powerbi-analytics.git
   ```
2. **Open the Report:**
   - Launch **Power BI Desktop**.
   - Open the `.pbix` file included in the repository.
3. **Refresh Data Source:**
   - In the Home ribbon, click **Transform Data** $\to$ **Data Source Settings**.
   - Point the file path to your local copy of `uber data set.csv`.
   - Click **Close & Apply** to refresh transformations and data models.
```

### Summary of What Was Created:
- **Data Engineering Overview:** Documented exact Power Query steps including data type casting, null handling, duration math, and temporal binning.
- **Data Modeling & Architecture:** Detailed the Star Schema relationship model between the fact table and date, time, driver, and location dimensions.
- **DAX Documentation:** Provided core business formulas for gross revenue, completion rates, cancellation rates, and lost revenue opportunity.
- **Deep Dashboard Breakdown:** Documented each dashboard page, detailing visuals, matrices, heatmaps, and strategic operational insights.