<img width="158" height="158" alt="UTH_greek_logo" src="https://github.com/user-attachments/assets/9dd2a451-bca2-4688-aa4a-61c447407a13" />

# NY Yellow Taxi Analytics with Apache Spark & Hadoop (2024–2025)

> Implemented by: Michail Solomonidis & Konstantinos Kiousis

This repository contains the code and related reports for a semester project on large-scale data processing using Apache Spark and Apache Hadoop.
The project focuses on **information extraction and performance evaluation** over real New York City taxi trip data using:

* MapReduce with the **Spark RDD API**
* **SparkSQL** with the **DataFrame API**
* Different storage formats (**CSV vs Parquet**)

---

## 1. Technologies & Environment

The project is developed and executed in a distributed environment provided by the course infrastructure. You are expected to:

* Use the **remote Kubernetes** environment described in the course lab guides.
* Configure **Spark Job History Server** locally (via Docker & Docker Compose) to read job logs from HDFS on the remote cluster.
* Use the following software versions:

  * **Apache Hadoop**: ≥ 3.3
  * **Apache Spark**: ≥ 3.5

---

## 2. Datasets

All datasets are real NYC yellow taxi trip data. Trip records for 2024 are available online:
[https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

In the lab HDFS, the following files are provided:

| Dataset                    | Description                                    | HDFS URI                                                  |
| -------------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| `yellow_tripdata_2024.csv` | 2024 taxi trips (zone-based, TLC Location IDs) | `hdfs://hdfs-namenode:9000/data/yellow_tripdata_2024.csv` |
| `yellow_tripdata_2015.csv` | 2015 taxi trips (with GPS coordinates)         | `hdfs://hdfs-namenode:9000/data/yellow_tripdata_2015.csv` |
| `taxi_zone_lookup.csv`     | NYC zone metadata (Borough, Zone, etc.)        | `hdfs://hdfs-namenode:9000/data/taxi_zone_lookup.csv`     |

### 2.1. 2024 Trip Data — `yellow_tripdata_2024.csv`

Each row describes a single taxi trip in 2024. Example:

VendorID,tpep_pickup_datetime,tpep_dropoff_datetime,passenger_count,trip_distance,RatecodeID,store_and_fwd_flag,PULocationID,DOLocationID,payment_type,fare_amount,extra,mta_tax,tip_amount,tolls_amount,improvement_surcharge,total_amount,congestion_surcharge,Airport_fee
1,2024-09-01T00:05:51.000,2024-09-01T00:45:03.000,1,9.8,1,N,138,48,1,47.8,10.25,0.5,13.3,6.94,1.0,79.79,2.5,1.75

**Field descriptions:**

| Field                   | Description                                                                                                                                                                 |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `VendorID`              | Code indicating the TPEP provider: 1 = Creative Mobile Technologies, LLC; 2 = Curb Mobility, LLC; 6 = Myle Technologies Inc; 7 = Helix.                                     |
| `tpep_pickup_datetime`  | Date & time when the meter was engaged.                                                                                                                                     |
| `tpep_dropoff_datetime` | Date & time when the meter was disengaged.                                                                                                                                  |
| `passenger_count`       | Number of passengers in the vehicle.                                                                                                                                        |
| `trip_distance`         | Trip distance in miles (as reported by the meter).                                                                                                                          |
| `RatecodeID`            | Final rate code at the end of the trip. Examples: 1 = Standard; 2 = JFK; 3 = Newark; 4 = Nassau/Westchester; 5 = Negotiated fare; 6 = Group ride; 99 = Unknown/Unavailable. |
| `store_and_fwd_flag`    | Whether the trip record was stored in the vehicle before being sent to the vendor (`Y` = stored, `N` = not stored).                                                         |
| `PULocationID`          | TLC zone ID where the meter was engaged (pickup location).                                                                                                                  |
| `DOLocationID`          | TLC zone ID where the meter was disengaged (dropoff location).                                                                                                              |
| `payment_type`          | Numeric payment method code: 0 = Flex Fare; 1 = Credit card; 2 = Cash; 3 = No charge; 4 = Dispute; 5 = Unknown; 6 = Voided trip.                                            |
| `fare_amount`           | Metered fare based on time and distance.                                                                                                                                    |
| `extra`                 | Additional charges and surcharges.                                                                                                                                          |
| `mta_tax`               | MTA tax, automatically triggered based on the meter rate.                                                                                                                   |
| `tip_amount`            | Tip amount (automatically filled for credit card tips; cash tips not included).                                                                                             |
| `tolls_amount`          | Total tolls paid on the trip.                                                                                                                                               |
| `improvement_surcharge` | Improvement surcharge applied at trip start (introduced in 2015).                                                                                                           |
| `total_amount`          | Total amount charged to passengers (excluding cash tips).                                                                                                                   |
| `congestion_surcharge`  | Total congestion surcharge collected on the trip.                                                                                                                           |
| `Airport_fee`           | Surcharge for pickups at LaGuardia and JFK airports only.                                                                                                                   |
| `cbd_congestion_fee`    | Per-trip congestion relief zone fee (MTA) introduced on 5 January 2025.                                                                                                     |

---

### 2.2. 2015 Trip Data — `yellow_tripdata_2015.csv`

This file contains 2015 taxi trips, including **raw GPS coordinates**.

Key fields:

| Column                  | Description                                                      |
| ----------------------- | ---------------------------------------------------------------- |
| `VendorID`              | Provider ID (1 or 2).                                            |
| `tpep_pickup_datetime`  | Pickup date and time.                                            |
| `tpep_dropoff_datetime` | Dropoff date and time.                                           |
| `passenger_count`       | Number of passengers.                                            |
| `trip_distance`         | Trip distance in miles.                                          |
| `pickup_longitude`      | Pickup longitude (degrees).                                      |
| `pickup_latitude`       | Pickup latitude (degrees).                                       |
| `RateCodeID`            | Rate code (1 = Standard, 2 = JFK, etc.).                         |
| `store_and_fwd_flag`    | Whether the trip was stored before sending (`Y`/`N`).            |
| `dropoff_longitude`     | Dropoff longitude.                                               |
| `dropoff_latitude`      | Dropoff latitude.                                                |
| `payment_type`          | Payment method (1 = Credit, 2 = Cash, etc.).                     |
| `fare_amount`           | Fare amount.                                                     |
| `extra`                 | Extra charges (e.g., night surcharge, weather).                  |
| `mta_tax`               | MTA tax ($0.50).                                                 |
| `tip_amount`            | Tip amount.                                                      |
| `tolls_amount`          | Toll amount.                                                     |
| `improvement_surcharge` | $0.30 surcharge from 2015 onwards.                               |
| `total_amount`          | Total amount (fare + extras + tip + tolls + taxes + surcharges). |

---

### 2.3. Zone Metadata — `taxi_zone_lookup.csv`

This file maps TLC Location IDs to boroughs and zones. Example:

"LocationID","Borough","Zone","service_zone"
1,"EWR","Newark Airport","EWR"
2,"Queens","Jamaica Bay","Boro Zone"
3,"Bronx","Allerton/Pelham Gardens","Boro Zone"
4,"Manhattan","Alphabet City","Yellow Zone"

Fields:

* **LocationID**: Unique numeric ID for each zone.
* **Borough**: Name of the borough (`"Manhattan"`, `"Queens"`, `"Brooklyn"`, `"Bronx"`, `"Staten Island"`, `"EWR"`).
* **Zone**: Neighborhood / zone name.
* **service_zone**:

  * `"Yellow Zone"` – zones mainly served by yellow cabs.
  * `"Boro Zone"` – zones outside core Manhattan, served by green cabs/other services.
  * `"EWR"` – Newark Airport.

These fields let us join trip records with geographic and service-zone information using `LocationID`.

---

## 3. Project Goals

The main goals of the assignment are:

1. **Hands-on experience with distributed systems**
   Installation, configuration, and management of **Apache Spark** and **Apache Hadoop**.

2. **Data analysis at scale**
   Using Spark APIs (RDD, DataFrame, SQL) to process large datasets.

3. **Performance & optimization understanding**
   Studying how data formats, cluster configurations, and Spark’s **Catalyst Optimizer** affect query planning and execution.

---

## 4. Part 1A — Information Extraction (SQL & MapReduce)

Although the data is provided as plain text (CSV), analytical queries executed over raw CSV are not efficient.
To improve performance, Spark (like traditional DBMSs) can load data into optimized **binary formats** (e.g., Parquet) that:

* Reduce memory and disk footprint → less I/O and faster execution.
* Store auxiliary statistics (e.g., min/max per block) that help skip unnecessary data during query execution.

### Task

You must compute the queries in **Table 1** in **two different ways**:

1. **MapReduce using the RDD API**

   * Implement MapReduce-style logic directly over the CSV files.

2. **SparkSQL using the DataFrame API**

   * Load CSVs into DataFrames and express the queries in SparkSQL.

Then:

3. **Convert CSV → Parquet**

   * Transform text files into Parquet.
   * Load Parquet files as DataFrames.
   * Re-run the SparkSQL variants of the queries.
   * Measure the **time required for the CSV → Parquet conversion**.

4. **Compare execution times**
   For each query in Table 1, collect and plot the execution time for:

   * (a) RDD/MapReduce over CSV
   * (b) SQL over CSV
   * (c) SQL over Parquet

   Comment on the observed performance differences.

---

## 5. Table 1 — Queries Q1–Q6

### Q1 — Average Pickup Coordinates by Hour (2015 dataset)

For each hour of the day (00–23), compute the **average pickup latitude and longitude**, ignoring records with zero coordinates.

* Use the **2015 dataset** (`yellow_tripdata_2015.csv`).
* Sort results by hour (ascending).

Example output:

| HourOfDay | Avg Latitude | Avg Longitude |
| --------- | ------------ | ------------- |
| 00        | 40.…         | -73.…         |
| 01        | 40.…         | -73.…         |
| …         | …            | …             |

---

### Q2 — Max Haversine Distance per Vendor (2015 dataset)

For each **VendorID**, compute:

* The **maximum geodesic distance (Haversine, in km)** between pickup and dropoff coordinates.
* The **duration** of the trip (in minutes) that achieved this maximum distance.

Use the **2015 dataset**.

Example output:

| VendorID | Max Haversine Distance (km) | Duration (min) |
| -------- | --------------------------- | -------------- |
| 1        | 38.72                       | 47.3           |
| 2        | 42.15                       | 53.8           |
| 6        | 34.90                       | 41.5           |
| 7        | 39.35                       | 46.2           |

**Haversine Distance Formula**

Let φ be latitude and λ be longitude (in radians). For two points (φ₁, λ₁) and (φ₂, λ₂):

* Δφ = φ₂ − φ₁
* Δλ = λ₂ − λ₁

Then:

* ( a = \sin^2(\frac{Δφ}{2}) + \cos(φ_1) \cdot \cos(φ_2) \cdot \sin^2(\frac{Δλ}{2}) )
* ( c = 2 \cdot \text{atan2}(\sqrt{a}, \sqrt{1-a}) )
* ( d = R \cdot c ), where ( R = 6371 ) km is Earth’s radius.

> Note: The **speed** of a trip can be defined as `distance / duration`.

---

### Q3 — Total Trips per Borough (2024 dataset)

For each **Borough** in NYC, compute the total number of trips that **start and end** in that borough.

* Use the **2024 dataset** (`yellow_tripdata_2024.csv`) joined with `taxi_zone_lookup.csv`.
* Sort the result by **total trips (descending)**.

Example output:

| Borough   | TotalTrips |
| --------- | ---------: |
| Manhattan |     124520 |
| Queens    |      78250 |
| Brooklyn  |      45200 |
| Bronx     |      36710 |

---

### Q4 — Night Trips per Vendor (23:00–07:00, 2024 dataset)

Compute the total number of trips that start between:

* 23:00–23:59 **or**
* 00:00–06:59

For each such trip:

1. Extract the hour from `tpep_pickup_datetime`.
2. Filter trips within the above time range.
3. Group by `VendorID` and count trips.

Use the **2024 dataset**.

Example output:

| VendorID | NightTrips |
| -------- | ---------: |
| 1        |      54820 |
| 2        |      47210 |

---

### Q5 — Most Frequent Zone Pairs (2024 dataset)

Find zone pairs with the highest number of trips between them, excluding trips where pickup and dropoff zones are the same.

* Use the **2024 dataset** and `taxi_zone_lookup.csv`.
* Group by `PickupZone → DropoffZone`.
* Count trips per pair.
* Sort by **TotalTrips (descending)**.

Example output:

| Pickup Zone     | Dropoff Zone       | TotalTrips |
| --------------- | ------------------ | ---------: |
| Midtown Center  | Upper East Side    |      32140 |
| Upper West Side | Times Square       |      28790 |
| JFK Airport     | Midtown Center     |      25310 |
| Chelsea         | Financial District |      19840 |

---

### Q6 — Revenue Breakdown per Borough (2024 dataset)

For each **origin borough** (pickup borough), compute the total revenue from all trips starting there, broken down into:

* `fare_amount` (base fare)
* `tip_amount` (tips)
* `tolls_amount` (tolls)
* `extra` (additional charges)
* `mta_tax`
* `congestion_surcharge`
* `airport_fee`
* `total_amount` (overall charge)

Use the **2024 dataset** with zone metadata.

* Group by borough.
* Sum each of the above fields.
* Sort the results by **total_amount (descending)** to find the most economically active boroughs.

Example output:

| Borough     |     Fare ($) |   Tips ($) |  Tolls ($) | Extras ($) | MTA Tax ($) | Congestion ($) | Airport Fee ($) | Total Revenue ($) |
| ----------- | -----------: | ---------: | ---------: | ---------: | ----------: | -------------: | --------------: | ----------------: |
| Manhattan   | 1,254,320.50 | 234,189.75 | 112,340.00 |  95,812.40 |   52,130.00 |      88,390.00 |       19,230.00 |      1,856,412.65 |
| Queens      |   874,520.30 | 132,450.20 |  98,420.00 |  54,210.75 |   36,740.00 |      45,210.00 |       12,800.00 |      1,254,351.25 |
| Brooklyn    |   432,105.80 |  64,891.55 |  41,295.00 |  28,675.60 |   18,330.00 |      22,870.00 |        4,310.00 |        612,478.95 |
| Bronx       |   216,430.25 |  21,840.00 |  12,760.00 |  14,380.90 |    9,200.00 |      10,890.00 |        1,210.00 |        286,711.15 |
| Staten Isl. |    42,350.00 |   3,240.00 |   2,180.00 |   1,985.20 |    1,080.00 |         880.00 |            0.00 |         51,715.20 |

---

## 6. Part 1B — Studying the SparkSQL Optimizer (Joins)

SparkSQL’s **Catalyst Optimizer** supports multiple join strategies (e.g., `BROADCAST`, `MERGE`, `SHUFFLE_HASH`, `SHUFFLE_REPLICATE_NL`).
It automatically chooses a join implementation based on:

* Data size and statistics
* User-defined configuration (e.g., broadcast thresholds)
* Query structure

Your tasks:

### 6.1. Join with Limited Taxi Records

1. Take the **first 50 records** from the taxi company dataset.
2. Execute a join using SparkSQL on the corresponding Parquet datasets, returning all fields.
3. Use `EXPLAIN` (and/or Job History Server) to determine:

   * Which join implementation Spark chose.
   * Why this implementation was selected based on:

     * Data sizes
     * Default Spark settings (e.g., `spark.sql.autoBroadcastJoinThreshold`).

You may implement the limit using a subquery with `LIMIT`.

### 6.2. Forcing a Different Join Strategy

1. Adjust Spark’s optimizer-related configurations so that Spark **does not** choose the join implementation from the previous step.
2. Re-run the same query.
3. Record:

   * New execution time.
   * New join strategy.
4. Compare the execution times and explain the differences.

For more details, see the official Spark SQL performance tuning docs:
[https://spark.apache.org/docs/latest/sql-performance-tuning.html](https://spark.apache.org/docs/latest/sql-performance-tuning.html)

---

## 7. Implementation Requirements

You must:

1. **Connect to the remote Kubernetes cluster** and set up the environment as described in the course guides.

2. **Configure Spark Job History Server** locally via Docker/Docker Compose so it reads job history from remote HDFS.

3. **Write code to:**

   * Read all datasets from HDFS.

   * Clean/transform them as needed.

   * Store them in **Parquet** format at:
     `hdfs://hdfs-namenode:9000/user/{username}/data/parquet/`

   > Pay special attention to `yellow_tripdata_2024.csv`, which is not initially in a Spark-friendly schema and may require preprocessing.

4. **Implement:**

   * **Q1** using:

     * RDD API
     * DataFrame API (with and without UDFs)
       Comment on performance differences.
   * **Q2** using:

     * RDD API
     * DataFrame API
     * SQL API
       Compare execution times.
   * **Q3** using:

     * (a) DataFrame API
     * (b) SQL API
       Run with CSV and Parquet input and discuss the impact of the input format.
   * **Q4** using:

     * SQL API
       Again, compare CSV vs Parquet input.
   * **Q5** using:

     * DataFrame API
       Compare CSV vs Parquet input.
   * **Q6** using:

     * DataFrame API
       Perform **horizontal and vertical scaling** by varying:

       * `spark.executor.instances`
       * `spark.executor.cores`
       * `spark.executor.memory`

---

## 8. Cluster Resource Configurations

Run your implementation using a total of **8 cores** and **16 GB** memory, with the following Spark configurations:

1. `2 executors  × 4 cores / 8 GB memory`
2. `4 executors  × 2 cores / 4 GB memory`
3. `8 executors  × 1 core / 2 GB memory`

For each configuration:

* Measure and compare performance.
* Comment on how scaling (horizontal vs vertical) affects query runtimes.

---

## 9. Join Strategy Analysis (Q3, Q4, Q5)

For **every join** in the implementations of Q3, Q4, and Q5:

1. Use `explain()` or Spark Job History Server to identify the chosen join strategy:

   * `BROADCAST`
   * `MERGE`
   * `SHUFFLE_HASH`
   * `SHUFFLE_REPLICATE_NL`
   * etc.
2. Include:

   * Output logs, EXPLAIN plans, or screenshots.
3. Based on theory:

   * Justify whether the chosen strategy makes sense for:

     * Data sizes
     * Join conditions
     * Cluster configuration

---

## 10. Notes & Hints

1. **RDD API input format**

   * Each line of the input file is read as a `String`.
   * To perform computations, you must parse fields into proper types (e.g. `Int`, `Double`, `Timestamp`).
   * Date parsing: you can use patterns like `'%Y-%m-%d %H:%M:%S'` (adapt for your language/library).

2. **Q2 (Speed)**

   * Speed of a trip can be defined as:
     `speed = distance / duration`.

3. **SQL & Optimizer**

   * Use `EXPLAIN` to inspect execution plans.
   * Tuning configurations (e.g., broadcast thresholds) can significantly change join strategies and runtimes.

---

**Academic Year:** 2024–2025
