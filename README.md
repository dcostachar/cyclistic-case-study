# Cyclistic Case Study: How Does A Bike-Share Company Navigate Speedy Success? 

Author: Charlene D'Costa <br />
Date: November 1, 2024 <br />
Capstone project for the Google Data Analytics Professional Certificate. <br />

[Tableau Dashboard](https://public.tableau.com/app/profile/charlene.d.costa/viz/CyclisticBikeShareAnalysisDashboard_17280817981870/CyclisticBikeShareAnalysisDashboard)

# Phase 1: Ask 

<details>
  <summary>Defining the business problem.</summary>

## 1.1 Project Overview

Cyclistic is a bike-share company based in Chicago, offering a diverse range of over 5,800 bicycles and 600 docking stations throughout the city. The company sets itself apart by providing inclusive options like reclining bikes, hand tricycles, and cargo bikes, catering to people with disabilities and those who prefer alternative bike types. While the majority of users ride for leisure, 30% utilize Cyclistic bikes for their daily commutes.

Since its launch in 2016, Cyclistic has rapidly expanded, becoming a key player in Chicago's urban mobility landscape. The company offers flexible pricing plans, including single-ride passes, full-day passes, and annual memberships. Cyclistic’s finance team has identified that annual members are significantly more profitable than casual riders, prompting the director of marketing, Lily Moreno, to focus on converting casual riders into annual members. Moreno believes that a deeper understanding of the usage patterns between casual riders and annual members is essential to achieve this.

## 1.2 Business Task

Analyze Cyclistic historical bike trip data to understand the differences in usage patterns between casual riders and annual members. Use these insights to inform the development of a targeted marketing strategy to convert casual riders into annual members, ultimately driving Cyclistic’s growth and profitability.

## 1.3 Key Stakeholders

* **Lily Moreno:** Director of Marketing at Cyclistic, responsible for overseeing the marketing strategy and driving the initiative to increase annual memberships.
* **Cyclistic Marketing Analytics Team:** A group of data analysts responsible for collecting, analyzing, and reporting data that helps guide Cyclistic's marketing strategies.
* **Cyclistic Executive Team:** The decision-making body that will evaluate and approve the proposed marketing strategy based on the analysis and recommendations.

</details>

# Phase 2: Prepare

<details>
  <summary>Collecting and validating relevant data for analysis.</summary>

## 2.1 About the Dataset

Since Cyclistic is a fictional company, the Google Data Analytics program has recommended using data from Chicago's Divvy bicycle-sharing service for this case study. This data is provided by Motivate International Inc. under a specific data license agreement. The dataset used in this case study spans 12 months of trip data from 2023, covering over 5,800 bicycles across 600 docking stations. It includes user usage data, including bike types, start and end times, start and end stations, ride duration, and user types (casual or member).

## 2.2 Data Compliance and Accessibility

The data is publicly available from Lyft Bikes and Scooters, LLC under a non-exclusive, royalty-free, and perpetual license. Users can access, reproduce, analyze, and distribute the data for any lawful purpose, with certain conditions. The dataset cannot be used unlawfully, sold as a stand-alone commercial product, or linked to personally identifiable customer information.

## 2.3 Data Integrity and Credibility

The dataset is sourced from a reliable and publicly accessible platform, ensuring its credibility for analytical purposes. While it is comprehensive in terms of ride details, it lacks personal demographic information about the riders, limiting the ability to conduct analyses that link ride data to specific user demographics. However, the data's reliability is bolstered by its currentness (from 2023), consistent format, and comprehensive coverage over an entire year, providing a robust foundation for analysis.

## 2.4 Data Organization and Verification

The dataset is organized into 12 CSV files, each representing one month of the year and containing detailed ride information. The data is presented in a long format, where each row corresponds to a single observation linked to a unique ride ID, and each column captures a specific attribute of that ride, including bike type, start and end times, start and end stations, ride duration, and user type. This structured organization allows for efficient data processing, cleaning, and analysis. 

</details>

# Phase 3: Process

<details>
  <summary>Cleaning and transforming data for analysis. </summary>

## 3.1 Importing the Data

### Creating a SQL Table and Importing CSV Data

In this phase, I will clean and transform the data to prepare it for analysis. I will use Docker to run
a MySQL database and connect it to DataGrip, my integrated development environment (IDE), to perform the analysis.

First, I will download 12 months of trip data from 2023, with each month’s data stored in a separate CSV file. Upon
inspecting the data, I observe that all 12 CSV files follow a consistent format in terms of the number of columns,
column names, and data types. I also identify that `ride_id` serves as a primary key (i.e., the unique identifier for
each record). Now that the structure of the data is understood, I can create my table in SQL to store the data. This
involves mapping the columns from the CSV files to the SQL table and assigning the appropriate data types.

```sql 
CREATE TABLE trips
(
    ride_id            VARCHAR(32),
    rideable_type      VARCHAR(32),
    started_at         DATETIME,
    ended_at           DATETIME,
    start_station_name VARCHAR(100),
    start_station_id   VARCHAR(100),
    end_station_name   VARCHAR(100),
    end_station_id     VARCHAR(20),
    start_lat          DECIMAL(10, 8),
    start_lng          DECIMAL(11, 8),
    end_lat            DECIMAL(10, 8),
    end_lng            DECIMAL(11, 8),
    member_casual      VARCHAR(32)
);
```

## 3.2 Data Validation

Checking the number of characters in each `ride_id`. Each `ride_id` has the same number of characters (16).

```sql
SELECT LENGTH(ride_id) AS ride_id_length, COUNT(*) AS ride_id_length_count
FROM trips
GROUP BY ride_id_length;
```

Checking the dataset contains only data from 2023. There are 45 rows containing trip data from 2024. However, upon
further inspection, each of these records corresponds to a trip that started on December 31, 2023, and concluded the
next day on January 1, 2024. This confirms that there are no inaccuracies with the data in these records.

```sql
SELECT *
FROM trips
WHERE YEAR (started_at) != 2023
   OR YEAR (ended_at) != 2023;
```

Checking the dataset contains all 12 months from 2023. Confirmed that they do.

```sql
SELECT DISTINCT MONTH (started_at) AS month
FROM trips
ORDER BY month;

SELECT DISTINCT MONTH (ended_at) AS month
FROM trips
ORDER BY month;
```

Checking the number of bike types. There are 3: electric, classic, and docked.

```sql
SELECT DISTINCT(rideable_type), COUNT(rideable_type)
FROM trips
GROUP BY rideable_type
ORDER BY COUNT(rideable_type) DESC; 
```

Checking the number of membership types. There are 2: member and casual.

```sql 
SELECT DISTINCT (member_casual), COUNT(member_casual)
from trips
GROUP BY member_casual
ORDER BY COUNT(member_casual) DESC;
```

## 3.3 Data Cleaning

### Identifying and Removing Duplicates

No duplicated `ride_id`. This is important as `ride_id` serves as a primary key.

```sql
SELECT COUNT(ride_id) - COUNT(distinct ride_id) AS duplicate_rows
FROM trips;
```

### Identifying and Handling NULL Values

Checking for the number of NULL values in each column of the table. I observed that the station-related data, along
with the corresponding latitude and longitude data, contain the majority of the NULL values. Therefore, I will focus
most of my time on cleaning this part of the dataset. Here is a breakdown of columns with NULL values:

- `start_station_name`: 875716
- `start_station_id`: 875848
- `end_station_name`: 929202
- `end_station_id`: 929343
- `end_lat`: 6990
- `end_long`: 6990

```sql
SELECT COUNT(*) - COUNT(ride_id)            AS ride_id,
       COUNT(*) - COUNT(rideable_type)      AS rideable_type,
       COUNT(*) - COUNT(started_at)         AS started_at,
       COUNT(*) - COUNT(ended_at)           AS ended_at,
       COUNT(*) - COUNT(start_station_name) AS start_station_name,
       COUNT(*) - COUNT(start_station_id)   AS start_station_id,
       COUNT(*) - COUNT(end_station_name)   AS end_station_name,
       COUNT(*) - COUNT(end_station_id)     AS end_station_id,
       COUNT(*) - COUNT(start_lat)          AS start_lat,
       COUNT(*) - COUNT(start_lng)          AS start_lng,
       COUNT(*) - COUNT(end_lat)            AS end_lat,
       COUNT(*) - COUNT(end_lng)            AS end_lng,
       COUNT(*) - COUNT(member_casual)      AS member_casual
FROM trips;
```

### Creating and Cleaning the Station Data Table

I'll begin by creating a new table called `station_data_cleaned` containing the station-related data. Creating a new
table from the existing one will give me a safe and structured environment to clean and validate my data. It preserves
the original data set and allows for focused data manipulation without the risk of corrupting the source data. After
cleaning the data in my `station_data_cleaned`, I will merge it back into the original table in a controlled manner,
ensuring accuracy and integrity in the final dataset.

Because I want to clean all station names, I will combine both the start and end station names into a single column
called `station_name`, which corresponds to the appropriate `station_id`.

```sql
CREATE TABLE station_data_cleaned
(
    station_name VARCHAR(100),
    station_id   VARCHAR(100)
);

INSERT INTO station_data_cleaned (station_name, station_id)
SELECT trips.start_station_name, trips.start_station_id
FROM trips;

INSERT INTO station_data_cleaned (station_name, station_id)
SELECT trips.end_station_name, trips.end_station_id
FROM trips;
```

Now that I've created a separate table, I'll proceed with cleaning the station data by applying various functions to
clean the string values.

I'll start by removing the "Public Rack" prefix in station names to standardize the station names for consistency and
comparison.

```sql
UPDATE station_data_cleaned
SET station_name = TRIM(REPLACE(station_name, 'Public Rack - ', ''))
WHERE station_name LIKE 'Public Rack%';
```

I will apply the same logic for removing the suffix "(Temp)".

```sql
UPDATE station_data_cleaned
SET station_name = TRIM(REPLACE(station_name, '(Temp)', ''))
WHERE station_name LIKE '% (Temp)';
```

Now I will remove any leading or trailing spaces from `station_names`.

```sql
UPDATE station_data_cleaned
SET station_name = TRIM(station_name);
```

Next, I'll convert all station names to lowercase to eliminate any inconsistent casing.

```sql
UPDATE station_data_cleaned
SET station_name = LOWER(station_name);
```

### Identifying and Handling Missing Data in Station Data

Now that I've cleaned my data, I want to investigate instances where a station name exists without a corresponding
station ID, and vice versa. This is important because I need to join on the station ID to connect the cleaned data to
my source table. Therefore, I want to identify how many stations are missing IDs.

From the queries below, I can see that there are no instances where the station name is NULL and the station ID is not.
However, there are 273 rows where a station name exists but no station ID is present. Upon further inspection, I found
that two stations appear repeatedly in the list of 273 records: Elizabeth St & Randolph St and Stony Island Ave & 63rd
St.

```sql
SELECT *
FROM station_data_cleaned
WHERE station_name IS NOT NULL
  AND station_id IS NULL;

SELECT *
FROM station_data_cleaned
WHERE station_name IS NULL
  AND station_id IS NOT NULL;

SELECT COUNT(DISTINCT station_name)
FROM station_data_cleaned
WHERE station_name IS NOT NULL
  AND station_id IS NULL;
```

I will now perform a fuzzy match on these station names to compare them with the other names in the station data table.
This will help us determine if they might match another station name in the table and already have an ID, with the
missing ID potentially due to a data entry error or a similar issue.

Good news! I found a station ID match for Elizabeth St & Randolph St. I will now insert the correct ID, 23001, into
all rows that have a missing ID for this station.

```sql
SELECT station_name,
       MIN(station_id)
FROM station_data_cleaned
GROUP BY station_name
HAVING station_name LIKE 'elizabeth%'

UPDATE station_data_cleaned
SET station_id = '23001'
WHERE station_name = 'elizabeth st & randolph st'
  AND station_id IS NULL;
```

More good news! A station ID match was also found for Stony Island Ave & 63rd St. I will now insert the correct ID,
653B, into the rows where the ID is missing.

```sql
SELECT station_name,
       MIN(station_id)
FROM station_data_cleaned
GROUP BY station_name
HAVING station_name LIKE 'stony island%'

UPDATE station_data_cleaned
SET station_id = '653B'
WHERE station_name = 'stony island ave & 63rd st'
  AND station_id IS NULL;
```

### Identifying and Handling NULL Values in Station Data

I'll run the same query as above to verify that the station IDs were updated correctly and to confirm that there are no
more NULL station IDs in our station data table.

```sql
SELECT *
FROM station_data_cleaned
WHERE station_name IS NOT NULL
  AND station_id IS NULL;
```

Since there are no issues, I'll remove the rows where the `station_name` and `station_id` are NULL.

```sql
DELETE
FROM station_data_cleaned
WHERE station_name IS NULL
  AND station_id IS NULL;
```

### Identifying and Removing Duplicates in Station Data

My last step is to remove the duplicates from the `station_data_cleaned` table so that each row is unique (i.e. all
station names and ids are distinct) and my upcoming joins can be carried out correctly.

If I simply use SELECT DISTINCT on `station_name` and `station_id`, I may encounter situations where the same station
name is incorrectly linked to multiple IDs or vice versa. To avoid this, instead of using SELECT DISTINCT, I'll use a
GROUP BY on both `station_name` and `station_id` to identify which grouping provides fewer rows. This will help me
prevent the issue described earlier.

Grouping by station_name returns 1582 rows, while grouping by station_id returns 1537 rows. This indicates that there
are more duplicate station names (i.e., multiple names linked to different IDs). Therefore, I will use the grouping by
station id to create my final station lookup table, as it results in fewer rows and eliminates duplicate data when
grouped by ID.

```sql
SELECT COUNT(*)
FROM (SELECT station_name, MIN(station_id) as station_id
      FROM station_data_cleaned
      GROUP BY station_name) AS station_name_grouping;

SELECT COUNT(*)
FROM (SELECT station_id, MIN(station_name) as station_name
      FROM station_data_cleaned
      GROUP BY station_id) AS station_id_grouping;
```

### Saving the Cleaned Station Data Table

Creating my final station lookup table with the cleaned station data, where each row is unique, with distinct station
names and IDs. This will ensure that the upcoming joins are performed correctly. To do this, I'll use a CTAS
command (Create Table As Select), which allows me to create a new table and populate it with the result set of a SELECT
query (the one used above), effectively combining both table creation and data insertion in a single step.

Upon creating my final station lookup table with the cleaned station data and giving it a final review, I noticed a few
station names and IDs that appear to be duplicates (e.g., the same station name with minor differences such as special
characters) or station names with "test" in the name, which seem to be invalid/inaccurate entries. Cleaning this
thoroughly would require a significant amount of time to review each row in detail. However, I’ve decided to timebox
this task, and for the purpose of this assignment, I will proceed knowing that I’ve already done as thorough a job as
possible cleaning the data while preserving as much of it as I could. Additionally, in some cases changing the station
ID in the final station lookup table could result in missing rows when the join is performed with the trips table. This
is another reason why I have decided to hold off on cleaning the final station lookup table data further.

```sql
CREATE TABLE final_station_data_cleaned AS
SELECT station_id, MIN(station_name) as station_name
FROM station_data_cleaned
GROUP BY station_id
```

### Preparing the Source Table for Merging with the Cleaned Station Data Table

Moving on to the next task: preparing the source table for merging with the cleaned data. This involves standardizing
the station names (string data) in the source table to match the formatting of the station names in the
`final_station_data_cleaned` table, ensuring the upcoming joins are carried out correctly.

I'll repeat the same steps as above, starting with removing the "Public Rack" prefix in station names to standardize
the station names for consistency and comparison. I'll check my work before updating the rows in the source table for
each step.

```sql
SELECT start_station_name                                      AS start_station_name_original,
       TRIM(REPLACE(start_station_name, 'Public Rack - ', '')) AS start_station_name_after,
       end_station_name                                        AS end_station_name_original,
       TRIM(REPLACE(end_station_name, 'Public Rack - ', ''))   AS end_station_name_after
FROM trips
WHERE start_station_name LIKE 'Public Rack%'
   OR end_station_name LIKE 'Public Rack%';

UPDATE trips
SET start_station_name = TRIM(REPLACE(start_station_name, 'Public Rack - ', ''))
WHERE start_station_name LIKE 'Public Rack%';

UPDATE trips
SET end_station_name = TRIM(REPLACE(end_station_name, 'Public Rack - ', ''))
WHERE end_station_name LIKE 'Public Rack%';

```

I will apply the same logic for removing the suffix "(Temp)".

```sql
SELECT start_station_name                              AS start_station_name_original,
       TRIM(REPLACE(start_station_name, '(Temp)', '')) AS start_station_name_after,
       end_station_name                                AS end_station_name_original,
       TRIM(REPLACE(end_station_name, '(Temp)', ''))   AS end_station_name_after
FROM trips
WHERE start_station_name LIKE '% (Temp)'
   OR end_station_name LIKE '% (Temp)';

UPDATE trips
SET start_station_name = TRIM(REPLACE(start_station_name, '(Temp)', ''))
WHERE start_station_name LIKE '% (Temp)';

UPDATE trips
SET end_station_name = TRIM(REPLACE(end_station_name, '(Temp)', ''))
WHERE end_station_name LIKE '% (Temp)';
```

Now I will remove any leading or trailing spaces from `station_names`.

```sql
SELECT start_station_name       AS start_station_name_original,
       TRIM(start_station_name) AS start_station_name_after,
       end_station_name         AS end_station_name_original,
       TRIM(end_station_name)   AS end_station_name_after
FROM trips;

UPDATE trips
SET start_station_name = TRIM(start_station_name);

UPDATE trips
SET end_station_name = TRIM(end_station_name);
```

Next, I'll convert all station names to lowercase to eliminate any inconsistent casing.

```sql
SELECT start_station_name        AS start_station_name_original,
       LOWER(start_station_name) AS start_station_name_after,
       end_station_name          AS end_station_name_original,
       LOWER(end_station_name)   AS end_station_name_after
FROM trips;

UPDATE trips
SET start_station_name = LOWER(start_station_name);

UPDATE trips
SET end_station_name = LOWER(end_station_name);
```

Now that I've standardized the station names (string data) in the source table, I will remove the rows where the
station data is fully or partially incomplete.

```sql
DELETE
FROM trips
WHERE (start_station_name IS NULL AND start_station_id IS NULL)
   OR (end_station_name IS NULL AND end_station_id IS NULL);
```

### Joining the Cleaned Source Table with the Cleaned Station Data Table

To join the source table `trips` with the cleaned station data table `final_station_data_cleaned` I will use Common
Table Expressions—temporary tables created from select statements using the WITH command—to update the
`start_station_id` and `end_station_id` in the `trips` table by joining them with the corresponding `station_id` from
the `final_station_data_cleaned` table.

In the first CTE I will fix the `start_station_id` for each trip by performing an INNER JOIN between the `trips` table
and `final_station_data_cleaned` table, matching the `start_station_id` in `trips` with the `station_id` in the
`final_station_data_cleaned` table. Building upon the first CTE, the second CTE will take the results of the first CTE
and fix the `end_station_id` for each trip by performing an INNER JOIN between the output of the first CTE, which is the
creation of the `start_station_id_fixed_trips` table and the `final_station_data_cleaned` table. Here, I matched the
`end_station_id` in `trips` with the `station_id` in the `final_station_data_cleaned` table. This ensures that
both the `start_station_id` and `end_station_id` are updated correctly for each trip.

The final SELECT * retrieves all rows from the `start_and_end_station_id_fixed_trips` CTE, which now includes fixed
values for both `start_station_id` and `end_station_id`.

This entire query ensures that both the start and end station IDs are correctly mapped to the cleaned station data for
all trips, providing a cleaner and more accurate dataset. The query ultimately affects 4,331,823 rows, reflecting all
the trips where station IDs have been corrected or updated.

```sql
WITH start_station_id_fixed_trips AS (SELECT trips.ride_id,
                                             trips.rideable_type,
                                             trips.started_at,
                                             trips.ended_at,
                                             station.station_name as start_station_name,
                                             station.station_id   as start_station_id,
                                             trips.end_station_name,
                                             trips.end_station_id,
                                             trips.start_lat,
                                             trips.start_lng,
                                             trips.end_lat,
                                             trips.end_lng,
                                             trips.member_casual
                                      FROM final_station_data_cleaned station
                                               INNER JOIN trips ON station.station_id = trips.start_station_id),

     start_and_end_station_id_fixed_trips AS (SELECT start_station_id_fixed_trips.ride_id,
                                                     start_station_id_fixed_trips.rideable_type,
                                                     start_station_id_fixed_trips.started_at,
                                                     start_station_id_fixed_trips.ended_at,
                                                     start_station_id_fixed_trips.start_station_name,
                                                     start_station_id_fixed_trips.start_station_id,
                                                     station.station_name as end_station_name,
                                                     station.station_id   as end_station_id,
                                                     start_station_id_fixed_trips.start_lat,
                                                     start_station_id_fixed_trips.start_lng,
                                                     start_station_id_fixed_trips.end_lat,
                                                     start_station_id_fixed_trips.end_lng,
                                                     start_station_id_fixed_trips.member_casual
                                              FROM final_station_data_cleaned station
                                                       INNER JOIN start_station_id_fixed_trips
                                                                  ON station.station_id = start_station_id_fixed_trips.end_station_id)
SELECT *
FROM start_and_end_station_id_fixed_trips;
-- 4,331,823 rows
```

Creating my final table, with my cleaned, normalized, and merged data.

```sql
CREATE TABLE trips_cleaned AS
WITH start_station_id_fixed_trips AS (SELECT trips.ride_id,
                                             trips.rideable_type,
                                             trips.started_at,
                                             trips.ended_at,
                                             station.station_name as start_station_name,
                                             station.station_id   as start_station_id,
                                             trips.end_station_name,
                                             trips.end_station_id,
                                             trips.start_lat,
                                             trips.start_lng,
                                             trips.end_lat,
                                             trips.end_lng,
                                             trips.member_casual
                                      FROM final_station_data_cleaned station
                                               INNER JOIN trips ON station.station_id = trips.start_station_id),

     start_and_end_station_id_fixed_trips AS (SELECT start_station_id_fixed_trips.ride_id,
                                                     start_station_id_fixed_trips.rideable_type,
                                                     start_station_id_fixed_trips.started_at,
                                                     start_station_id_fixed_trips.ended_at,
                                                     start_station_id_fixed_trips.start_station_name,
                                                     start_station_id_fixed_trips.start_station_id,
                                                     station.station_name as end_station_name,
                                                     station.station_id   as end_station_id,
                                                     start_station_id_fixed_trips.start_lat,
                                                     start_station_id_fixed_trips.start_lng,
                                                     start_station_id_fixed_trips.end_lat,
                                                     start_station_id_fixed_trips.end_lng,
                                                     start_station_id_fixed_trips.member_casual
                                              FROM final_station_data_cleaned station
                                                       INNER JOIN start_station_id_fixed_trips
                                                                  ON station.station_id = start_station_id_fixed_trips.end_station_id)
SELECT *
FROM start_and_end_station_id_fixed_trips;
```
</details>

# Phase 4: Analyze

<details>
  <summary>Analyzing data using SQL to uncover trends and generate insights.</summary>

## 4.1 Bike Usage Patterns

### Casuals vs. Members: Most and Least Used Bike Types

Let's begin by getting a breakdown of casuals and members in the dataset. I see that there are ~2.8 million members
and ~1.5 million casuals.

```sql
SELECT member_casual, COUNT(member_casual)
FROM trips_cleaned
GROUP BY member_casual;
```

Bike usage patterns for casuals vs. members. Classic bikes are more popular than electric bikes across both
categories, while docked bikes are only used by casuals.

```sql
SELECT rideable_type, COUNT(rideable_type)
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY rideable_type
ORDER BY COUNT(rideable_type) DESC;

SELECT rideable_type, COUNT(rideable_type)
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY rideable_type
ORDER BY COUNT(rideable_type) DESC;
```

## 4.2 Trip Distance and Duration Trends

### Casuals vs. Members: Most and Least Popular Start and End Stations

10 most popular start stations for casuals vs. members.

```sql
SELECT start_station_name, COUNT(start_station_name)
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY start_station_name
ORDER BY COUNT(start_station_name) DESC LIMIT 10;

SELECT start_station_name, COUNT(start_station_name)
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY start_station_name
ORDER BY COUNT(start_station_name) DESC LIMIT 10;
```

Common popular start stations. There are none.

```sql
WITH casual_ss AS (SELECT start_station_name, COUNT(start_station_name)
                   FROM trips_cleaned
                   WHERE member_casual = 'casual'
                   GROUP BY start_station_name
                   ORDER BY COUNT(start_station_name) DESC
    LIMIT 10
    )
   , member_ss AS (
SELECT start_station_name, COUNT (start_station_name)
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY start_station_name
ORDER BY COUNT (start_station_name) DESC
    LIMIT 10
    )
SELECT casual_ss.start_station_name
FROM casual_ss
         INNER JOIN member_ss
                    ON casual_ss.start_station_name = member_ss.start_station_name;
```

Least popular start stations for casuals vs. members determined by < 10 visits in total for the year. 233 stations in
common.

```sql
SELECT start_station_name, COUNT(start_station_name) AS cnt_start_station_name_casual
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY start_station_name
HAVING cnt_start_station_name_casual < 10;
-- 391 stations

SELECT start_station_name, COUNT(start_station_name) AS cnt_start_station_name_member
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY start_station_name
HAVING cnt_start_station_name_member < 10;
-- 367 stations
```

10 most popular end stations for casuals vs. members.

```sql
SELECT end_station_name, COUNT(end_station_name)
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY end_station_name
ORDER BY COUNT(end_station_name) DESC LIMIT 10;

SELECT end_station_name, COUNT(end_station_name)
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY end_station_name
ORDER BY COUNT(end_station_name) DESC LIMIT 10;
```

Common popular end stations. 1 station in common: wells st & concord ln.

```sql
WITH casual_es AS (SELECT end_station_name, COUNT(end_station_name) AS cnt_casual
                   FROM trips_cleaned
                   WHERE member_casual = 'casual'
                   GROUP BY end_station_name
                   ORDER BY COUNT(end_station_name) DESC
    LIMIT 10
    )
   , member_es AS (
SELECT end_station_name, COUNT (end_station_name) AS cnt_member
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY end_station_name
ORDER BY COUNT (end_station_name) DESC
    LIMIT 10
    )
SELECT casual_es.end_station_name
FROM casual_es
         INNER JOIN member_es
                    ON casual_es.end_station_name = member_es.end_station_name;
```

Least popular end stations for casuals vs. members determined by < 10 visits in total for the year. 233 stations in
common.

```sql
SELECT end_station_name, COUNT(end_station_name) AS cnt_end_station_name_casual
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY end_station_name
HAVING cnt_end_station_name_casual < 10;
-- 402 stations

SELECT end_station_name, COUNT(end_station_name) AS cnt_end_station_name_member
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY end_station_name
HAVING cnt_end_station_name_member < 10;
-- 370 stations
```

### Casuals vs. Members: Most Popular Trips

10 most popular trips for casuals vs. members determined by `start_station_name` to `end_station_name`.

```sql
SELECT start_station_name, end_station_name, COUNT(*) AS count
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY start_station_name, end_station_name
ORDER BY count DESC
    LIMIT 10;

SELECT start_station_name, end_station_name, COUNT(*) AS count
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY start_station_name, end_station_name
ORDER BY count DESC
    LIMIT 10;
```

Most popular common trips. There is only 1: ellis ave & 60th st to ellis ave & 55th st.

```sql
WITH casual_trips AS (SELECT start_station_name, end_station_name, COUNT(*) AS count
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY start_station_name, end_station_name
ORDER BY count DESC
    LIMIT 10
    ),
    member_trips AS (
SELECT start_station_name, end_station_name, COUNT (*) AS count
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY start_station_name, end_station_name
ORDER BY count DESC
    LIMIT 10
    )
SELECT casual_trips.start_station_name, casual_trips.end_station_name
FROM casual_trips
         INNER JOIN member_trips
                    ON casual_trips.start_station_name = member_trips.start_station_name
                        AND casual_trips.end_station_name = member_trips.end_station_name;
```

### Casuals vs. Members: Average Ride Duration and Distance

Average ride duration for casuals vs. members (i.e. 50% of users in each category exhibit this behaviour).

```sql
SELECT AVG(TIMESTAMPDIFF(MINUTE, started_at, ended_at)) AS avg_ride_duration_mins_casual
FROM trips_cleaned
WHERE member_casual = 'casual';
-- ~22 minutes

SELECT AVG(TIMESTAMPDIFF(MINUTE, started_at, ended_at)) AS avg_ride_duration_mins_member
FROM trips_cleaned
WHERE member_casual = 'member';
-- ~12 minutes
```

Average ride distance for casuals vs. members (i.e. 50% of users in each category exhibit this behaviour).

```sql
-- Inputting a function to compute the distance between two points in km using latitude and longitude data. 
DELIMITER
$$

CREATE FUNCTION haversine_distance(lat1 FLOAT, lon1 FLOAT, lat2 FLOAT, lon2 FLOAT)
    RETURNS FLOAT
    DETERMINISTIC
BEGIN
    DECLARE
R INTEGER DEFAULT 6371;  -- Radius of the Earth in kilometers
    DECLARE
lat1_rad FLOAT;
    DECLARE
lon1_rad FLOAT;
    DECLARE
lat2_rad FLOAT;
    DECLARE
lon2_rad FLOAT;
    DECLARE
dlat FLOAT;
    DECLARE
dlon FLOAT;
    DECLARE
a FLOAT;
    DECLARE
c FLOAT;
    DECLARE
distance FLOAT;

    -- Convert degrees to radians
    SET
lat1_rad = RADIANS(lat1);
    SET
lon1_rad = RADIANS(lon1);
    SET
lat2_rad = RADIANS(lat2);
    SET
lon2_rad = RADIANS(lon2);

    -- Calculate differences
    SET
dlat = lat2_rad - lat1_rad;
    SET
dlon = lon2_rad - lon1_rad;

    -- Apply the Haversine formula
    SET
a = SIN(dlat / 2) * SIN(dlat / 2) + COS(lat1_rad) * COS(lat2_rad) * SIN(dlon / 2) * SIN(dlon / 2);
    SET
c = 2 * ATAN2(SQRT(a), SQRT(1 - a));

    -- Calculate the distance
    SET
distance = R * c;

RETURN distance; -- Distance in kilometers
END$$

DELIMITER ;

-- applying the function for our use case. 
SELECT AVG(haversine_distance(start_lat, start_lng, end_lat, end_lng)) AS avg_distance_in_km
FROM trips_cleaned
WHERE member_casual = 'casual';
-- ~2 km

SELECT AVG(haversine_distance(start_lat, start_lng, end_lat, end_lng)) AS avg_distance_in_km
FROM trips_cleaned
WHERE member_casual = 'member';
-- ~2 km
```

## 4.3 Trip Timing Trends Across Month, Day, and Hour

### Casuals vs. Members: Trip Timing Patterns by Month, Day, and Hour

Monthly ride patterns for casuals vs. members.

```sql
SELECT EXTRACT(MONTH from started_at) as month,
       COUNT(ride_id) as trips_per_month
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY month
ORDER BY trips_per_month DESC;

SELECT EXTRACT(MONTH from started_at) as month,
       COUNT(ride_id) as trips_per_month
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY month
ORDER BY trips_per_month DESC;
```

Daily ride patterns for casuals vs. members.

```sql
SELECT DAYNAME(started_at) as day_of_week,
       COUNT(ride_id)      as trips_per_day_of_week
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY day_of_week
ORDER BY trips_per_day_of_week DESC;

SELECT DAYNAME(started_at) as day_of_week,
       COUNT(ride_id)      as trips_per_day_of_week
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY day_of_week
ORDER BY trips_per_day_of_week DESC;
```

Hourly ride patterns for casuals vs. members.

```sql
SELECT HOUR (started_at) as hour_of_day, COUNT (ride_id) as trips_per_hour_of_day
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY hour_of_day
ORDER BY trips_per_hour_of_day DESC;

SELECT HOUR (started_at) as hour_of_day, COUNT (ride_id) as trips_per_hour_of_day
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY hour_of_day
ORDER BY trips_per_hour_of_day DESC;
```

</details>

# Phase 5: Share

<details>
  <summary>Presenting findings through Tableau visualizations to make insights accessible and actionable.</summary>

## Distribution of Casuals vs. Members

Let's begin by breaking down casuals and members in our dataset. We see that there are ~2.8 million members and ~1.5 million casuals.

<div align="center">
<img width="201" alt="1" src="https://github.com/user-attachments/assets/34cd8708-3f1a-4311-bb5f-dc955f9f5a1d">
</div>

<div align="center">
<img width="302" alt="2" src="https://github.com/user-attachments/assets/b0a67d06-8481-4ede-874d-9790a5c98932">
</div>

## Most and Least Used Bike Types

Classic bikes are more popular than electric bikes across both categories, while docked bikes are only used by casuals.

<div align="center">
<img width="421" alt="3" src="https://github.com/user-attachments/assets/d5bffcbf-fa0b-4d20-8451-ba1230973bb2">
</div>

## Top 10 Start and End Stations 

Looking at the most popular start and end stations for casual riders, here are my key insights: 

**Tourist and Scenic Spots:** Many locations are well-known scenic or tourist areas in Chicago, such as Streeter Dr & Grand Ave (near Navy Pier and Lake Michigan), Millennium Park, Shedd Aquarium, Theater on the Lake, and Adler Planetarium. This suggests that casual riders may be tourists or people exploring popular sights.

**Lakefront Locations:** Several stations are located along or near Lake Michigan, including DuSable Lake Shore Dr & Monroe St, Michigan Ave & Oak St, and Montrose Harbor, indicating that casual riders are drawn to scenic lakefront routes.

**Recreational Areas:** High usage at stations near parks and recreational spots, like Millennium Park and DuSable Harbor, supports the idea that casual riders often use bikes for leisure rather than commuting

**Round Trips:** The overlap between popular start and end locations suggests casual riders frequently take round trips, likely for sightseeing or short rides that begin and end near major attractions.

Overall, these patterns indicate that casual riders primarily use the bike-sharing service for leisure and sightseeing, particularly around popular attractions and the lakefront, rather than for daily commuting.

<div align="center">
<img width="859" alt="image" src="https://github.com/user-attachments/assets/0987abab-31a3-49b6-b81b-4ed06b29bce5">
</div>

<br /> Looking at the most popular start and end stations for members, here are my key insights: <br /> 

**Downtown Locations:** Many stations, such as Clinton St & Washington Blvd, Kingsbury St & Kinzie St, Clark St & Elm St, and Clinton St & Madison St, are near key intersections in busy downtown areas. This suggests that member riders are likely using bike-sharing for commuting or accessing frequently visited spots in the city center, such as offices.

**University and Residential Areas:** Stations like University Ave & 57th St, Loomis St & Lexington St, and Ellis Ave & 60th St suggest that some members may be students or residents who use bike-sharing regularly within their neighbourhoods or for commuting to nearby facilities.

**Broader Distribution Across Residential, Commercial, and Practical Locations:** Unlike casual riders, who tend to cluster around tourist-heavy areas, member trips are spread across a wider range of residential and commercial locations. This indicates that members prioritize practical locations closer to workplaces, residences, and transit hubs, highlighting a focus on commuting and utility trips rather than leisure or tourism.

**Consistent Start and End Patterns:** Similar to casual riders, the overlap between popular start and end stations suggests that members often take round trips or short point-to-point rides within the same area, which aligns with typical commuting behaviour.

Overall, these patterns indicate that members primarily use the bike-sharing service for commuting to work or school or for routine travel within the city, focusing on practical and accessible locations over tourist destinations or scenic spots.

<div align="center">
<img width="859" alt="image" src="https://github.com/user-attachments/assets/7653ce55-6912-46dc-92c4-a911209bf27a">
</div>

## Average Ride Durations and Distances

Looking at the average ride duration and distance for casual riders vs. members, we observe that casual riders, on average, use the service twice as long as members; however, the actual distance covered is comparable. This suggests that casual riders are likelier to take long, leisurely trips, possibly for sightseeing or recreation. In contrast, members use the service more efficiently, taking shorter, practical trips like commuting.

In summary, this pattern indicates that casual riders use the service for leisure-oriented, extended rides, while members prioritize efficiency, consistent with a commuting or task-focused approach to bike-sharing.

<div align="center">
<img width="529" alt="6" src="https://github.com/user-attachments/assets/62e99864-3f60-4ada-a388-f54b78c67ce6">
</div>

## Trip Timing Patterns by Month, Day, and Hour

Observing trip timing patterns by month, day, and hour, here are my key insights:

**Monthly Trends:** Both casual and member riders show increased activity in warmer months, peaking from May to September. Casual riders exhibit a more pronounced summer peak, especially around July, suggesting they use the service primarily for leisure or tourism, which is more seasonal. In contrast, member usage remains relatively steady year-round, with only a slight increase in summer, indicating consistent use likely for commuting or regular transportation needs.

**Daily Trends:** Casual riders prefer weekends, especially Saturdays, while members have steady weekday usage with minor fluctuations. This pattern reinforces the idea that casual users are primarily engaging in leisure or recreational rides on weekends, whereas members’ consistent weekday usage aligns with commuting or routine trips.

**Hourly Trends:** Casual riders’ trips peak in the afternoon (3 PM - 5 PM), suggesting a preference for leisurely rides during those hours. Member riders show two distinct peaks: one around 8 AM and another around 5 PM, typical of commuting patterns as users ride to and from work during rush hours. Both groups have significantly lower activity late at night, indicating the service is primarily used during daytime and evening hours.

Overall, these patterns indicate that casual riders use the bike-sharing service seasonally, favouring summer months, weekends, and afternoons, reflecting a leisure-oriented use. Member riders display a more consistent, year-round pattern, with weekday peaks during commute hours, suggesting practical, task-focused usage.

<div align="center">
<img width="874" alt="7" src="https://github.com/user-attachments/assets/c8763de8-9cd3-4bfd-b5a4-1119161cae77">
</div>

</details>

# Phase 6: Act 

<details>
  <summary>Reporting the results of the analysis to project stakeholders and providing recommendations to address the business problem.</summary>
    
<br /> Based on our findings on casual and member rider patterns, here are targeted marketing strategies to encourage casual riders to become members: <br />

**Seasonal Promotions During Peak Months:** Casual riders are most active in the summer, so offering limited-time discounts or promotional rates for new memberships during these months (e.g., 20% off if they sign up in July) can capitalize on when they’re most engaged with the service. Summer-only perks like free ride credits or priority access to busy stations can further incentivize them to join.

**Weekend-Exclusive Membership Benefits:** Since casual riders favour weekend rides, create a “Weekend Warrior” membership option that includes benefits such as extra ride time or priority access to high-demand stations on weekends. Emphasize the cost savings for frequent weekend usage, making the membership appealing to those who primarily ride on weekends.

**Cost-Comparison Campaigns Near Popular Tourist and Leisure Spots:** Many casual riders may not realize the cost benefits of a membership. Targeted ads at popular casual rider locations (like Streeter Dr, Millennium Park, and Shedd Aquarium) can showcase how a membership helps avoid per-ride fees, which is ideal for those exploring the city. Partnering with local attractions or events near these stations to offer temporary discounts or “day passes” with an upgrade option can also boost membership interest.

**Free Trial or Flexible Membership Options for Leisure Riders:** Providing a one-week or one-month trial during summer allows casual riders to experience the benefits of having a membership with no risk. Offering a discounted first month after the trial could encourage them to stay. Alternatively, short-term memberships with flexible terms, such as pausing or cancelling in off-peak seasons, can attract riders who don’t want a year-round commitment.

**Incentives for Longer Rides:** Since casual riders tend to enjoy scenic, longer rides, offer rewards for completing rides over a certain distance or duration (e.g., 5 km or 20 minutes), reinforcing the value of a membership for those seeking leisurely experiences.

These strategies leverage casual riders’ seasonal and weekend preferences and appeal to their potential for more frequent use while reducing the perceived risk of commitment.

</details>
