

# Cloudera Data Warehouse Workshops - Hands on Workshop

Analyze Stored Data with Trino and Iceberg

## Introduction
This workshop gives you an overview of how to use the Cloudera Data Warehouse service to quickly explore raw data, create curated versions of the data for reporting and dashboarding, and then scale up usage of the curated data by exposing it to more users. It highlights the performance and automation capabilities that help ensure performance is maintained while controlling cost.  

Star Schema Diagram of tables we use in todays workshop:
- fact table: flights (86mio rows)
- dimension tables: airlines (1.5k rows), airports (3.3k rows) and planes (5k rows)
- federation table: customer_complains (50k rows)

![](images/starschema001.png)


-----
## Lab 1 - Create Schema

Navigate to Data Warehouse, then Trino Virtual Warehouse and open the Cloudera Data Explorer SQL Authoring tool (formerly known as HUE).

Create new schema for your user to be used, or use one that is already created for you.

```sql
-- 1. Create the schema in the catalogs for Hive and Iceberg
CREATE SCHEMA ${your_dbname};

-- 2. Switch your session context to the Iceberg catalog and your new schema
USE iceberg.${your_dbname};

-- 3. Verify your context (should show 'iceberg' and 'user_0')
SELECT current_catalog, current_schema;

```
-----
## Lab 2 - External Tables

Run DDL to create four external tables on the CSV data files, which are already in cloud object storage.

```sql
-- 1. Planes Table
DROP TABLE IF EXISTS hive.${your_dbname}.flights_csv;
CREATE TABLE hive.${your_dbname}.flights_csv (
    month VARCHAR,
    dayofmonth VARCHAR,
    dayofweek VARCHAR,
    deptime VARCHAR,
    crsdeptime VARCHAR,
    arrtime VARCHAR,
    crsarrtime VARCHAR,
    uniquecarrier VARCHAR,
    flightnum VARCHAR,
    tailnum VARCHAR,
    actualelapsedtime VARCHAR,
    crselapsedtime VARCHAR,
    airtime VARCHAR,
    arrdelay VARCHAR,
    depdelay VARCHAR,
    origin VARCHAR,
    dest VARCHAR,
    distance VARCHAR,
    taxiin VARCHAR,
    taxiout VARCHAR,
    cancelled VARCHAR,
    cancellationcode VARCHAR,
    diverted VARCHAR,
    carrierdelay VARCHAR,
    weatherdelay VARCHAR,
    nasdelay VARCHAR,
    securitydelay VARCHAR,
    lateaircraftdelay VARCHAR,
    year VARCHAR
)
WITH (
    format = 'CSV',
    csv_separator = ',',
    external_location = 's3a://trino-hol-cp-buk-498916c0/data/flights_csv',
    skip_header_line_count = 1
);

-- 2. Planes Table
DROP TABLE IF EXISTS hive.${your_dbname}.planes_csv;
CREATE TABLE hive.${your_dbname}.planes_csv (
    tailnum VARCHAR,
    owner_type VARCHAR,
    manufacturer VARCHAR,
    issue_date VARCHAR,
    model VARCHAR,
    status VARCHAR,
    aircraft_type VARCHAR,
    engine_type VARCHAR,
    year VARCHAR -- Changed from INTEGER to VARCHAR
)
WITH (
    format = 'CSV',
    csv_separator = ',',
    external_location = 's3a://trino-hol-cp-buk-498916c0/data/planes_csv',
    skip_header_line_count = 1
);

-- 3. Airlines Table
DROP TABLE IF EXISTS hive.${your_dbname}.airlines_csv;
CREATE TABLE hive.${your_dbname}.airlines_csv (
    code VARCHAR,
    description VARCHAR
)
WITH (
    format = 'CSV',
    csv_separator = ',',
    external_location = 's3a://trino-hol-cp-buk-498916c0/data/airlines_csv',
    skip_header_line_count = 1
);

-- 4. Airports Table
DROP TABLE IF EXISTS hive.${your_dbname}.airports_csv;
CREATE TABLE hive.${your_dbname}.airports_csv (
    iata VARCHAR,
    airport VARCHAR,
    city VARCHAR,
    state VARCHAR,
    country VARCHAR,
    lat VARCHAR,
    lon VARCHAR
)
WITH (
    format = 'CSV',
    csv_separator = ',',
    external_location = 's3a://trino-hol-cp-buk-498916c0/data/airports_csv',
    skip_header_line_count = 1
);

```


Check that you created tables

```sql
SHOW TABLES;
```


Results


|TAB_NAME|
| :- |
|airlines_csv|
|airports_csv|
|flights_csv|
|planes_csv|

Query external tables to see few samples pointing to the right files

```sql
SELECT
  *
FROM  
  hive.${your_dbname}.airports_csv
LIMIT 3;
```

Results


|airports_csv.iata | airports_csv.airport |airports_csv.city |airports_csv.country |airports_csv.lat| airports_csv.lon|
| :- | :- | :- | :- | :- | :- |
|00M	|Thigpen	|Bay Springs |USA	|31.95376472	|-89.23450472 |
|00R	|Livingston Municipal	|Livingston |USA	|30.68586111	|-95.0179277 |
|00V	|Meadow Lake |Colorado Springs |USA	|38.94574889	|-104.5698933 |


Run exploratory queries to understand the data. This reads the CSV data, converts it into a columnar in-memory format, and executes the query.

QUERY: Airline Delay Aggregate Metrics by Airplane.

DESCRIPTION: Customer Experience Reporting showing airplanes that have the highest average delays, causing the worst customer experience.

*Do all these steps in the* **“db\_user001”..”db\_user020”** *unless otherwise noted.*

```sql
SELECT
  tailnum,
  count(*) as flights_count,
  -- 1. NULLIF turns '' into NULL
  -- 2. CAST turns NULL (or the string) into an INTEGER
  -- 3. COALESCE turns that resulting NULL into 0
  sum(coalesce(try_cast(nullif(depdelay, '') as integer), 0)) AS departure_delay_minutes,

  sum(case when coalesce(try_cast(nullif(depdelay, '') as integer), 0) > 0 then 1 else 0 end) as departure_delay_count
FROM
  hive.${your_dbname}.flights_csv
GROUP BY
  tailnum
ORDER BY
  departure_delay_minutes DESC
LIMIT 5;
```
Note: Running the first time may take some time.

Results

|tailnum	| flights_count | departure_delay_minutes |	 departure_delay_count|
| :- | :- | :- | :- |
|N381UA	| 25287 |341368 | 12280	|
|N375UA	| 25147 |341103	| 12162 |
|N673	| 30616 |333744	| 12835	|
|N366UA	| 24808 |331318	| 12113	|
|N377UA	| 25105 |328546	| 12163	|



-----
## Lab 3 - Iceberg Tables

Run “CREATE TABLE AS SELECT” queries to create full features ICEBERG v2 type of the tables. This creates curated versions of the data which are optimal for BI usage.

*Do all these steps in * **“iceberg.db\_user001”..”db\_user020”**

```sql
-- 1. Airlines Table
DROP TABLE IF EXISTS iceberg.${your_dbname}.dim_airlines;
CREATE TABLE iceberg.${your_dbname}.dim_airlines
WITH (format = 'PARQUET')
AS
SELECT
  code,
  description
FROM hive.${your_dbname}.airlines_csv;

-- 2. Airports Table
DROP TABLE IF EXISTS iceberg.${your_dbname}.dim_airports;
CREATE TABLE iceberg.${your_dbname}.dim_airports
WITH (format = 'PARQUET')
AS
SELECT
  iata,
  airport,
  city,
  state,
  country,
  TRY_CAST(lat AS DOUBLE) as lat,
  TRY_CAST(lon AS DOUBLE) as lon
FROM hive.${your_dbname}.airports_csv;

-- 3. Planes Table
DROP TABLE IF EXISTS iceberg.${your_dbname}.dim_planes;
CREATE TABLE iceberg.${your_dbname}.dim_planes
WITH (format = 'PARQUET')
AS
SELECT
  tailnum, owner_type, manufacturer, issue_date, model,
  status, aircraft_type, engine_type,
  TRY_CAST(NULLIF(year, '') AS INTEGER) as year
FROM hive.${your_dbname}.planes_csv;

-- 4. Flights Table (Partitioned by Year)
DROP TABLE IF EXISTS iceberg.${your_dbname}.fct_flights;
CREATE TABLE iceberg.${your_dbname}.fct_flights
WITH (
  format = 'PARQUET',
  partitioning = ARRAY['year']
)
AS
SELECT
  TRY_CAST(NULLIF(year, '') AS INTEGER) as year,
  TRY_CAST(NULLIF(month, '') AS INTEGER) as month,
  TRY_CAST(NULLIF(dayofmonth, '') AS INTEGER) as dayofmonth,
  TRY_CAST(NULLIF(dayofweek, '') AS INTEGER) as dayofweek,
  TRY_CAST(NULLIF(deptime, '') AS INTEGER) as deptime,
  TRY_CAST(NULLIF(crsdeptime, '') AS INTEGER) as crsdeptime,
  TRY_CAST(NULLIF(arrtime, '') AS INTEGER) as arrtime,
  TRY_CAST(NULLIF(crsarrtime, '') AS INTEGER) as crsarrtime,
  uniquecarrier,
  TRY_CAST(NULLIF(flightnum, '') AS INTEGER) as flightnum,
  tailnum,
  TRY_CAST(NULLIF(actualelapsedtime, '') AS INTEGER) as actualelapsedtime,
  TRY_CAST(NULLIF(crselapsedtime, '') AS INTEGER) as crselapsedtime,
  TRY_CAST(NULLIF(airtime, '') AS INTEGER) as airtime,
  TRY_CAST(NULLIF(arrdelay, '') AS INTEGER) as arrdelay,
  TRY_CAST(NULLIF(depdelay, '') AS INTEGER) as depdelay,
  origin,
  dest,
  TRY_CAST(NULLIF(distance, '') AS INTEGER) as distance,
  TRY_CAST(NULLIF(taxiin, '') AS INTEGER) as taxiin,
  TRY_CAST(NULLIF(taxiout, '') AS INTEGER) as taxiout,
  TRY_CAST(NULLIF(cancelled, '') AS INTEGER) as cancelled,
  cancellationcode,
  diverted,
  TRY_CAST(NULLIF(carrierdelay, '') AS INTEGER) as carrierdelay,
  TRY_CAST(NULLIF(weatherdelay, '') AS INTEGER) as weatherdelay,
  TRY_CAST(NULLIF(nasdelay, '') AS INTEGER) as nasdelay,
  TRY_CAST(NULLIF(securitydelay, '') AS INTEGER) as securitydelay,
  TRY_CAST(NULLIF(lateaircraftdelay, '') AS INTEGER) as lateaircraftdelay
FROM hive.${your_dbname}.flights_csv;

```

This takes a few minutes to read and write the data back.

Check that you created managed & external tables

```sql
SHOW TABLES;
```

Results

|TAB_NAME|
| :- |
|airlines_csv|
|dim_airlines|
|airports_csv|
|dim_airports|
|flights_csv|
|fct_flights|
|planes_csv|
|dim_planes|

The shows detailed information about the table.

 ```sql
DESCRIBE iceberg.${your_dbname}.fct_flights ;
 ```
Result: column names with types

|col_name| data_type| comment|
| :- | :- |:- |
|year| integer | |
|month| integer | |
|dayofmonth| integer | |
|dayofweek| integer | |
...

Show column statistics of the created iceberg table.

 ```sql
SHOW STATS FOR iceberg.${your_dbname}.fct_flights;
 ```

Result: column data statistics

|#|column_name|data_size|distinct_values_count|nulls_fraction|row_count|low_value|high_value|
| :- |:- |:- |:- |:- |:- |:- |:- |
|1|year|NULL|14|0|NULL|1995|2008|
|2|month|NULL|12|0|NULL|1|12|
|3|dayofmonth|NULL|31|0|NULL|1|31|
|4|dayofweek|NULL|7|0|NULL|1|7|
|5|deptime|NULL|1619|0.0218189|NULL|1|2318|
|6|crsdeptime|NULL|1293|0|NULL|1|1927|
...

You see statistics immediately after create table as select (CTAS) in Trino's Iceberg connector is due to a specific feature called "Collect on Write."

Looking deeper into the partitioning as in Apache Iceberg, partitions are tracked in Manifest Files. Trino isn't touching your data files at all; it is performing a high-speed metadata-only read.

Lets look deeper into partitions of the "fct_flights" table:

 ```sql
 SELECT partition, record_count, file_count, total_size   
 FROM iceberg.${your_dbname}."fct_flights$partitions"
 ORDER BY partition;
 ```
Result: showing all 14 partitions with keys (years)

|partition |      record_count|    file_count   |   total_size|
| :- |:- |:- |:- |
|[1995] | 5327435 |5      | 57774143|
|[1996] | 5351983 |7      | 58347109|
|[1997] | 5411843 |7      | 59631458|
|[1998] | 5384721 |6      | 59213838|
...

Uniform File Distribution: You have roughly 5 to 7 files per partition for ~5M to 7M rows. This is a very "healthy" distribution. These files are relatively small (under 100MB), Trino's can pull these files into memory, decompress the columns, and process them in parallel across your worker nodes effortlessly.


Experiment with different queries to see effects of the columnar storage format and cache.

QUERY: Airline Delay Aggregate Metrics by Airplane on managed table

```sql
SELECT
  tailnum,
  count(*) as flights_count,
  -- COALESCE is the Trino/Standard SQL
  sum(coalesce(depdelay, 0)) AS departure_delay_minutes,
  -- Adding ELSE 0 ensures the SUM handles non-matches as zero
  sum(case when coalesce(depdelay, 0) > 0 then 1 else 0 end) as departure_delay_count
FROM
  iceberg.${your_dbname}.fct_flights
GROUP BY
  tailnum
ORDER BY
  departure_delay_minutes DESC
LIMIT 5;
```

Results (same as previous query)
|tailnum	| flights_count | departure_delay_minutes |	 departure_delay_count|
| :- | :- | :- | :- |
|N381UA	| 25287 |341368 | 12280	|
|N375UA	| 25147 |341103	| 12162 |
|N673	| 30616 |333744	| 12835	|
|N366UA	| 24808 |331318	| 12113	|
|N377UA	| 25105 |328546	| 12163	|


The "Airline Marathon" Common Table Expression (CTE) structure

This is a CTE-type query (using the WITH clause). It first calculates the top 5 airlines by total mileage in an initial sub-block, then joins that result to the flight data to find the single longest route for each.

```sql
WITH TopAirlines AS (
    -- Identify the 5 airlines with the most total mileage
    SELECT
        uniquecarrier,
        SUM(distance) as total_fleet_miles
    FROM iceberg.${your_dbname}.fct_flights
    WHERE cancelled = 0
    GROUP BY 1
    ORDER BY 2 DESC
    LIMIT 5
),
LongestFlights AS (
    -- Find the max distance flight for those specific airlines
    SELECT
        a.description AS airline_name,
        f.origin,
        f.dest,
        f.distance,
        f.airtime,
        -- Rank flights within each airline by distance
        ROW_NUMBER() OVER(PARTITION BY a.description ORDER BY f.distance DESC) as rank_id
    FROM iceberg.${your_dbname}.fct_flights f
    JOIN iceberg.${your_dbname}.dim_airlines a ON f.uniquecarrier = a.code
    WHERE a.code IN (SELECT uniquecarrier FROM TopAirlines)
)
SELECT
    airline_name,
    origin || ' to ' || dest AS route,
    distance AS marathon_miles,
    airtime AS duration_minutes
FROM LongestFlights
WHERE rank_id = 1
ORDER BY marathon_miles DESC;
```

Expected Output:

| airline_name |	route	| marathon_miles |	duration_minutes |
| :- | :- | :- | :- |
| Delta Air Lines Inc.	| ATL to HNL |	4502 |	541 |
| United Air Lines Inc. |	ORD to HNL |	4243	| 506 |
| American Airlines Inc. |	ORD to HNL |	4243 |	462 |
| US Airways Inc. (Merged with America West 9/05. Reporting for both starting 10/07.)	| LIH to PHX |	2979	| 313 |
| Southwest Airlines Co. |	OAK to PHL |	2510 | 284 |


### SQL AI Assistant - makes SQL development faster, easier, and less error-prone

The SQL AI Assistant is an AI-powered tool designed to enhance SQL development, making it faster, more intuitive, and less prone to errors. By leveraging advanced contextual understanding of your data, it provides accurate and relevant SQL code suggestions that improve productivity. Integrated into Hue within Cloudera, this assistant harnesses the capabilities of Large Language Models (LLMs) for a range of SQL tasks, including query creation, editing, optimization, debugging, and summarization.

Click on the blue dot to launch the SQL AI Assistant

![](images/cdw-lab1-ai001.png)

this unfolds this bar and click on EXPLAIN

![](images/cdw-lab1-ai002.png)

The SQL AI Assistant will take a few seconds to generate a outcome.

![](images/cdw-lab1-ai004.png)

This can be inserted for documentation purposes.



### Geospatial Query 

Trino's geospatial functions convert raw latitude and longitude data into a relational graph of physical proximity, enabling distance calculations and point-in-polygon joins across federated data sources.

This query identifies all airports within a 50-kilometer radius of San Francisco International Airport (SFO) by dynamically retrieving SFO's coordinates and calculating the spherical distance to every other airport in the table using a geospatial join.

```sql
WITH reference_point AS (
    -- Get the base coordinates for SFO
    SELECT
        lat AS ref_lat,
        lon AS ref_lon
    FROM iceberg.${your_dbname}.dim_airports
    WHERE iata = 'SFO'
)
SELECT
    a.iata,
    a.airport,
    a.city,
    -- Calculate distance using built-in Great Circle function
    ROUND(great_circle_distance(a.lat, a.lon, r.ref_lat, r.ref_lon), 2) AS distance_km,
    lon,
    lat
FROM
    iceberg.${your_dbname}.dim_airports a
CROSS JOIN
    reference_point r
WHERE
    -- Filter within 50km radius
    great_circle_distance(a.lat, a.lon, r.ref_lat, r.ref_lon) <= 50
    AND a.iata != 'SFO' -- Exclude the origin point
ORDER BY
    distance_km ASC;
```

Expect output

| iata	| airport	| city	|	distance_km
| :- | :- | :- |:- |
|HAF |	Half Moon Bay	|Half Moon Bay	|	16.14 |
|SQL |	San Carlos |	San Carlos		| 16.25 |
|OAK | Metropolitan Oakland | International	Oakland	|	17.7 |
|HWD |	Hayward Executive |	Hayward		| 22.67 |
|PAO |	Palo Alto Arpt of Santa Clara Co |	Palo Alto	|	28.86 |
|SJC |	San Jose International |	San Jose	|	48.63 |
|LVK | Livermore Municipal	| Livermore	|	49.51 |
|CCR |	Buchanan	| Concord	|	49.79 |

To visualize that in a map select Marker Map from the drop down lists

![](images/cdw-geoquery-001.png)

and select the fields lon = Longitude and lat = Latitude for the Map

![](images/cdw-geoquery-002.png)

### Surrogate Key 

Trino can use UUID as surrogate keys easy & distributable & fast, but not in sequence and has gaps.

```sql
DROP TABLE IF EXISTS iceberg.${your_dbname}.dim_airlines_with_surrogate_key;

CREATE TABLE iceberg.${your_dbname}.dim_airlines_with_surrogate_key (
    -- Generates a 128-bit unique identifier string
    id VARCHAR,
    code VARCHAR,
    description VARCHAR
);

INSERT INTO iceberg.${your_dbname}.dim_airlines_with_surrogate_key (id, code, description)
SELECT
  cast( uuid() as varchar),
  code, description
FROM
  hive.${your_dbname}.airlines_csv;

SELECT
 *
FROM  
 iceberg.${your_dbname}.dim_airlines_with_surrogate_key
ORDER BY
 id
LIMIT 3;
```

Result:

|id	| code |	 description|
| :- | :- | :- |
|0089fbea-c17d-4754-88e3-a9aa391bd45a	| AC |	Air Canada |
|009d2fa7-41f5-4fd0-88c7-74303407fe31 |	BAC	| Business Aircraft Corp. |
|0103f5b3-d9be-455e-ab9d-0dcb2531196a	| ECR	| East Coast Airways |

Note: the first column is the new unique SURROGATE_KEY

### Create a SEQUENCE - optional

```sql
-- 1. Create the target table structure
DROP TABLE IF EXISTS iceberg.${your_dbname}.airlines_with_seq;

CREATE TABLE iceberg.${your_dbname}.airlines_with_seq (
    id BIGINT,
    code VARCHAR,
    description VARCHAR
);

-- 2. Insert with a gapless sequence
INSERT INTO iceberg.${your_dbname}.airlines_with_seq (id, code, description)
SELECT
    row_number() OVER () AS id, -- This generates the gapless 1, 2, 3...
    code,
    description
FROM
    hive.${your_dbname}.airlines_csv;

-- 3. Verify
SELECT * FROM iceberg.${your_dbname}.airlines_with_seq ORDER BY id LIMIT 3;
```

Result:

|id	| code | description|
| :- | :- | :- |
|1 |02Q |Titan Airways |
|2 |04Q |Tradewind Aviation |
|3 |05Q |Comlux Aviation |

------


## Lab 4 - Federated Query

Trino’s federated query capability serves as a modern architectural feature by allowing users to execute a single SQL statement across multiple, diverse data sources like PostgreSQL, Iceberg, and Hive or Impala without moving the data. This "query-in-place" approach eliminates the need for time-consuming ETL processes, enabling real-time insights by joining live operational data with massive historical datasets stored in a data lake.

Quick check query a table in PostgreSQL
```sql
select * from  postgres.airlinedata.customer_complaints Limit 3;
 ```

Expected outcome

|complaint_id|	complaint_date|	customer_email	|complaint_category	|complaint_text	|uniquecarrier|	flightnum|	delay_minutes	|severity_score|
| :- | :- | :- | :- | :- | :- | :- | :- | :- |
|256|	2001-06-20 20:42:00.000	|m.garcia256@gmail.com|	DOT Refund Eligible| Delay	Sitting on the tarmac for hours. This violates the 3-hour domestic rule.	|DL	|744|	227	|4|
|257|	2001-06-22 00:00:00.000	|alex.chen257@outlook.com|	Involuntary Cancellation|	Flight 744 was cancelled. I am stuck at ATL and the rebooking app is crashing.|DL|	744	|NULL	|5|
|258|	2001-06-23 18:48:00.000	|sarah_j258@icloud.com|	DOT Refund Eligible Delay	|Sitting on the tarmac for hours. This violates the 3-hour domestic rule.	|DL	|744|	205|	4|


Lets create a federated query with dataset from PostgreSQL and Iceberg.

![](images/fq-sample001.png)



Purpose of this query is to  identify service trends by carrier and aircraft model. By performing complex cross-catalog joins and data type conversions, it allows for a unified analysis of customer sentiment against physical assets without the need for data movement or pre-processing.

 ```sql
SELECT
    f.uniquecarrier,
    p.model AS aircraft_model,
    COUNT(c.complaint_id) AS total_complaints,
    ROUND(AVG(CAST(c.severity_score AS DOUBLE)), 2) AS avg_severity
FROM
    iceberg.${your_dbname}.fct_flights f
JOIN
    postgres.airlinedata.customer_complaints c
    ON f.uniquecarrier = c.uniquecarrier
    AND CAST(f.flightnum AS VARCHAR) = CAST(c.flightnum AS VARCHAR)
    AND f.year = CAST(EXTRACT(year FROM c.complaint_date) AS integer)
    AND f.month = CAST(EXTRACT(month FROM c.complaint_date) AS integer)
    AND f.dayofmonth = CAST(EXTRACT(day FROM c.complaint_date) AS integer)
JOIN
    iceberg.${your_dbname}.dim_planes p
    ON f.tailnum = p.tailnum
WHERE
    p.model <> ''
GROUP BY
    f.uniquecarrier, p.model
ORDER BY
    total_complaints DESC
LIMIT 3;
 ```

Expected outcome (may vary)

 |uniquecarrier	|aircraft_model	|total_complaints	|avg_severity|
 | :- | :- | :- | :- |
 |DL |	MD-88	|2278	|3.3|
 |AA	|DC-9-82(MD-82)|	1696	|2.71|
 |DL|	757-232	|1402	|3.1|


 This lab demonstrates that Trino’s query federation effectively collapses data silos by enabling real-time joins between operational PostgreSQL feedback and historical Iceberg flight archives. By eliminating the need for data movement, you’ve established a high-performance architecture that delivers immediate visibility into how specific aircraft models impact the overall customer experience.

----

## Lab 5 - Data Governance and Security

Lets explore a important component of the data security that the dynamic policy enforcement that operates by pushing security rules directly to lightweight plugins within the Trino. This architecture ensures zero-latency authorization because the access check happens locally at the point of request.



In this example we defined a dynamic masking policy on the ***customer_email*** to redact the field.


![](images/rangerpolicy.png)

Query the data

```sql
select
  complaint_date,
  customer_email,
  complaint_category,
  severity_score
from
  postgres.airlinedata.customer_complaints
limit 3;
```

results

|complaint_date |	customer_email|	complaint_category|	severity_score|
| :- | :- | :- | :- |
|2001-06-20 20:42:00.000|	x.xxxxxx000@xxxxx.xxx	|DOT Refund Eligible Delay |	4 |
|2001-06-22 00:00:00.000|	xxxx.xxxx000@xxxxxxx.xxx	|Involuntary Cancellation	| 5 |
|2001-06-23 18:48:00.000|	xxxxx_x000@xxxxxx.xxx	|DOT Refund Eligible Delay |	4 |

The enforcement engine intercepted the request and alters the data depending on the Ranger policy.

### Data Redaction - Targeted Queries Return Zero Results - Optinal

When a Redaction policy is active, the engine evaluates the WHERE clause against the transformed value (e.g., x.xxxxxx000@xxxxx.xxx), causing a mismatch with the original clear-text string.

```sql
select
  complaint_date,
  customer_email,
  complaint_category,
  severity_score
from
  postgres.airlinedata.customer_complaints
where
  customer_email = 'm.garcia256@gmail.com'
```

 Done. 0 results.

 This ensures that even if an unauthorized user knows a specific email address, Cloudera SDX prevents them from confirming its existence or accessing the record.
 
## Lab - Data Security & Governance

The combination of the Data Warehouse with SDX offers a list of powerful features like rule-based masking columns based on a user’s role and/or group association or rule-based row filters.

For this workshop we are going to explore Attribute-Based Access Control a.k.a. Tage-based security policies.

First we are going to create a series of tables in your work database.

In the SQL editor, select your database and run this script:

```sql
CREATE TABLE hive.${your_dbname}.emp_fname (id int, fname varchar);
insert into hive.${your_dbname}.emp_fname(id, fname) values (1, 'Carl'),(2, 'Clarence');

CREATE TABLE hive.${your_dbname}.emp_name (id int, lname varchar);
insert into hive.${your_dbname}.emp_name(id, lname) values (1, 'Rickenbacker'), (2, 'Fender');

CREATE TABLE hive.${your_dbname}.emp_age (id int, age smallint);
insert into hive.${your_dbname}.emp_age(id, age) values (1, 35),(2, 55);

CREATE TABLE hive.${your_dbname}.emp_denom (id int, denom char(2), email varchar);
insert into hive.${your_dbname}.emp_denom(id, denom, email) values (1, 'rk','cr@yahoo.com'),(2, 'na','cfender@gmail.com');

CREATE TABLE hive.${your_dbname}.emp_id (id int, empid integer);
insert into hive.${your_dbname}.emp_id(id, empid) values (1, 1146651),(2, 239125);

CREATE TABLE hive.${your_dbname}.emp_all as
  (select a.id, a.fname, b.lname, c.age, d.denom,d.email,e.empid from hive.${your_dbname}.emp_fname a
	inner join hive.${your_dbname}.emp_name b on b.id = a.id
	inner join hive.${your_dbname}.emp_age c on c.id = b.id
	inner join hive.${your_dbname}.emp_denom d on d.id = c.id
	inner join hive.${your_dbname}.emp_id e on e.id = d.id);

create table hive.${your_dbname}.emp_younger as (select * from hive.${your_dbname}.emp_all where emp_all.age <= 45);

create table hive.${your_dbname}.emp_older as (select * from hive.${your_dbname}.emp_all where emp_all.age > 45);
```

After this script executes, a simple

```sql
select * from hive.${your_dbname}.emp_all;
```

… should give the contents of the emp\_all table, which only has a couple of lines of data.

For the next step we will switch to the UI of Atlas, the CDP component responsible for metadata management and governance: in the Cloudera Data Warehouse *Overview* UI, select Database Catalog. Click on the three-dot menu of this DB catalog and select “Open Atlas” in the associated pop-up menu:

![](images/RangerUIOpen.png)

This should open the Atlas UI. CDP comes with a newer, improved user interface which can be enabled through the __“Switch to Beta”__ item in the user menu on the upper right corner of the screen. Do this now.

The Atlas UI has a left column which lists the Entities, Classifications, Business Metadata and Glossaries that belong to your CDP Environment.

![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.007.png)

We just created a couple of tables in the Data Warehouse, let’s look at the associated metadata. Under “Entities”, click on “hive\_db”. This should produce a list of databases.
Select you workshop database, this will result in the database’s metadata being displayed.

Select the “Tables” tab (the rightmost)
![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.008.png)

Select the “emp\_all” table from the list, this will result in Atlas displaying the metadata for this table; select the “lineage” tab:
   ![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.009.png)
This lineage graph shows the inputs, outputs as well as the processing steps resulting from the execution of our SQL code in the Data Warehouse.

The red circle marks the currently selected entity. Atlas will always display the current entity's type in braces next to the entity name (middle, top of the page, e.g. "hive_table"). Clicking on one of the nodes will display a popup menu, which allows us to navigate through the lineage graph.


## Lab 6 - Data Visualization

A example of dashboard:

![](images/dataviz-010.png)

A quick way to create a new dashboard by the following steps:

Navigate to DataVisualizaton and click on NEW DATASET

Enter:

Dataset Title: ```Top Grumpy Routes```
Dataset Source:  ```SQL```
Enter SQL below:

```sql
SELECT
  o.city || ' to ' || d.city AS route,
  o.city as origion,
  d.city as destination,
  f.uniquecarrier ,
  COUNT(c.complaint_id) AS complaint_volume
FROM postgres.airlinedata.customer_complaints c
JOIN iceberg.cpearce.fct_flights f ON c.uniquecarrier = f.uniquecarrier AND c.flightnum = cast ( f.flightnum as varchar)
JOIN iceberg.cpearce.dim_airports o ON f.origin = o.iata
JOIN iceberg.cpearce.dim_airports d ON f.dest = d.iata
GROUP BY 1,2,3,4
ORDER BY 2 DESC;
```

Click on Show Data
Click on CREATE


This Dataset shows and click on New Dashboard

![](images/cdwlab10-0010.png)


The UI create a new visual screen and click on Explore Options in the upper right corner.

![](images/cdwlab10-0011.png)

Click on Visual Types

![](images/cdwlab10-0012.png)

Select the dimenions (three fields)
- Routes
- uniquecarrier
- complaint_volume

Click on SEE ALL VISUALS

![](images/cdwlab10-0013.png)

Scroll up/down and select one of the proposed visuals i.e. Scattered w/Trendline

![](images/cdwlab10-0014.png)

The selected visual appears in the window.

![](images/cdwlab10-0015.png)

This show a fast Dashboard creation.

### Optional - Lab 6 Data Visualization - Step by Step

![](images/dataviz-011.png)


`	`Open DataViz


|**Step**|**Description**|
| :-: | :- |
|1|<p>Open Data Visualization ![](images/cdw-lab9-00nav.png) ![](images/cdw-lab9-01nav.png)</p><p></p><p></p><p></p><p></p>|
|2|<p>Overview</p><p>![](images/cdw-lab9-02nav.png)</p>|
|3|<p>Switch to Data Tab</p><p>![](images/cdw-lab9-03nav.png)</p><p>There a demo datasets shown here (you can explore by your own)</p>|
|4|<p>Click on the Connection and then new dataset</p><p></p><p>![](images/cdw-lab9-04nav.png)</p><p></p>|
|5|<p>Enter name: airline_logistics the select database: airlinedata and table: flights_orc and click CREATE</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.023.png)</p><p></p><p></p><p>     </p><p></p>|
|6|<p>Edit Dataset - click on the name: airline_logistics</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.024.png)</p><p></p><p>The Dataset Details</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.026.png)</p>|
|7|<p>Click on Fields - List fields of the Dataset</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.027.png) <p>Show fields, each column of the FLIGHTS table, in two categories: Dimensions and Measures</p>    ![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.028.png)</p><p></p><p></p>|
|8|<p>Join PLANES table with FLIGHTS table - click on Data Model</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.029.png)</p><p>Then Click on Edit Model and the + </p><p>![](images/cdw-lab9-08nav.png)</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.030.png)  </p><p></p><p></p><p>Select the source and target column to join the two tables ![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.031.png)</p>|
|9|<p>Add AIRLINES table to the Dataset</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.032.png)  </p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.033.png)</p><p>DON'T FORGET to click on SAVE ! </p><p>![](images/cdw-lab9-09nav.png)</p> |
|10|<p>Click on SHOW DATA to view dataset</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.034.png)</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.035.png)</p><p></p><p>Scroll right for all columns of the dataset</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.036.png)</p>|
|11|<p>Go back to Data Model to Edit Fields</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.037.png)</p><p></p><p>Click on EDIT FIELDS</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.038.png)</p><p></p><p>Start with changing the display title of the field: deptdelay</p><p></p><p>Edit Field properties</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.041.png)</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.042.png)  </p><p>Change Field Name into: Dep Delay</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.043.png)</p><p></p><p></p><p></p><p>Add a new field: route that concatenates two fields origin and dest</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.045.png)</p><p>First clone the field: origin</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.046.png)     ![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.047.png)</p><p></p><p></p><p>Change Display Name to “Route”</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.049.png)</p><p></p><p>Edit Expression</p><p></p><p>Expression: </p><p>**concat( [origin],'-', [dest])**</p><p></p><p>Validate (to check for any errors) and Click Apply (to accept changes)</p><p>![](images/cdw-lab9-10nav.png)</p>|
|12|<p>The Dataset with all fields looks:</p><p>![](images/cdw-lab9-12nav.png)</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.054.png)</p><p>Click Save that completes the dataset</p>|
|13|<p>Create Dashboard</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.055.png)</p><p>` `![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.056.png)</p>|
|14|<p>First Visual - select bar and drag the field: route into x-axis and Dep delay into y-axis </p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.057.png)</p><p></p><p></p><p>Change Dep Delay Aggregate to Average</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.058.png)</p><p></p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.059.png)</p><p></p><p></p><p>Change to only show Top 25 Avgs</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.060.png)</p><p></p><p>Click on []Enter/Edit Expression </p><p>![](images/datvizavg.png)</p><p></p><p>avg(([Dep delay])) as 'Avg Dep Delay' and select 'average' as the aggregate</p><p>![](images/dashdes.png)</p><p>The new alias is created</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.061.png)</p><p></p><p>That should look likes</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.062.png)</p><p></p><p>Now click on refresh Visual</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.063.png)</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.064.png)</p><p></p><p>Add Title & Subtitle for Dashboard</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.065.png)</p><p></p><p>Add Title & Subtitle for this chart</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.066.png)</p><p></p>|
|15|<p>Add Filter</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.078.png)</p><p></p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.079.png)</p><p></p><p></p><p>Select values from prompt</p><p>![](images/cdw-lab9-20nav.png)</p><p></p><p></p><p></p>|
|17|<p>Save Dashboard</p><p>![](images/Aspose.Words.10bb90cf-0d99-47f3-a995-23ef2b90be86.088.png)</p><p></p><p>![](images/cdw-lab9-22nav.png)</p>|


