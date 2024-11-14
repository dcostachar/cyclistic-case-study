# Phase 3: Process

In this phase, we will clean and transform our data to prepare it for analysis.

## 3.1 Importing the Data

### Creating a SQL Table and Importing CSV Data

First, we will download 12 months of trip data from 2023, with each month’s data stored in a separate CSV file. Upon
inspecting the data, we observe that all 12 CSV files follow a consistent format in terms of the number of columns,
column names, and data types. We also identify that `ride_id` serves as a primary key (i.e., the unique identifier for
each record). Now that the structure of the data is understood, we can create our table in SQL to store the data. This
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

Checking for the number of NULL values in each column of the table. We observed that the station-related data, along
with the corresponding latitude and longitude data, contain the majority of the NULL values. Therefore, we will focus
most of our time on cleaning this part of the dataset. Here is a breakdown of columns with NULL values:

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

We'll begin by creating a new table called `station_data_cleaned` containing the station-related data. Creating a new
table from the existing one will give us a safe and structured environment to clean and validate our data. It preserves
the original data set and allows for focused data manipulation without the risk of corrupting the source data. After
cleaning the data in our `station_data_cleaned`, we will merge it back into the original table in a controlled manner,
ensuring accuracy and integrity in the final dataset.

Because we want to clean all station names, we will combine both the start and end station names into a single column
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

Now that we've created a separate table, we'll proceed with cleaning the station data by applying various functions to
clean the string values.

We'll start by removing the "Public Rack" prefix in station names to standardize the station names for consistency and
comparison.

```sql
UPDATE station_data_cleaned
SET station_name = TRIM(REPLACE(station_name, 'Public Rack - ', ''))
WHERE station_name LIKE 'Public Rack%';
```

We will apply the same logic for removing the suffix "(Temp)".

```sql
UPDATE station_data_cleaned
SET station_name = TRIM(REPLACE(station_name, '(Temp)', ''))
WHERE station_name LIKE '% (Temp)';
```

Now we will remove any leading or trailing spaces from `station_names`.

```sql
UPDATE station_data_cleaned
SET station_name = TRIM(station_name);
```

Next, we'll convert all station names to lowercase to eliminate any inconsistent casing.

```sql
UPDATE station_data_cleaned
SET station_name = LOWER(station_name);
```

### Identifying and Handling Missing Data in Station Data

Now that we've cleaned our data, we want to investigate instances where a station name exists without a corresponding
station ID, and vice versa. This is important because we need to join on the station ID to connect the cleaned data to
our source table. Therefore, we want to identify how many stations are missing IDs.

From the queries below, we can see that there are no instances where the station name is NULL and the station ID is not.
However, there are 273 rows where a station name exists but no station ID is present. Upon further inspection, we found
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

We will now perform a fuzzy match on these station names to compare them with the other names in the station data table.
This will help us determine if they might match another station name in the table and already have an ID, with the
missing ID potentially due to a data entry error or a similar issue.

Good news! We found a station ID match for Elizabeth St & Randolph St. We will now insert the correct ID, 23001, into
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

More good news! A station ID match was also found for Stony Island Ave & 63rd St. We will now insert the correct ID,
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

We'll run the same query as above to verify that the station IDs were updated correctly and to confirm that there are no
more NULL station IDs in our station data table.

```sql
SELECT *
FROM station_data_cleaned
WHERE station_name IS NOT NULL
  AND station_id IS NULL;
```

Since there are no issues, we'll remove the rows where the `station_name` and `station_id` are NULL.

```sql
DELETE
FROM station_data_cleaned
WHERE station_name IS NULL
  AND station_id IS NULL;
```

### Identifying and Removing Duplicates in Station Data

Our last step is to remove the duplicates from the `station_data_cleaned` table so that each row is unique (i.e. all
station names and ids are distinct) and our upcoming joins can be carried out correctly.

If we simply use SELECT DISTINCT on `station_name` and `station_id`, we may encounter situations where the same station
name is incorrectly linked to multiple IDs or vice versa. To avoid this, instead of using SELECT DISTINCT, we'll use a
GROUP BY on both `station_name` and `station_id` to identify which grouping provides fewer rows. This will help us
prevent the issue described earlier.

Grouping by station_name returns 1582 rows, while grouping by station_id returns 1537 rows. This indicates that there
are more duplicate station names (i.e., multiple names linked to different IDs). Therefore, we will use the grouping by
station id to create our final station lookup table, as it results in fewer rows and eliminates duplicate data when
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

Creating our final station lookup table with the cleaned station data, where each row is unique, with distinct station
names and IDs. This will ensure that the upcoming joins are performed correctly. To do this, we'll use a CTAS
command (Create Table As Select), which allows us to create a new table and populate it with the result set of a SELECT
query (the one used above), effectively combining both table creation and data insertion in a single step.

Upon creating our final station lookup table with the cleaned station data and giving it a final review, I noticed a few
station names and IDs that appear to be duplicates (e.g., the same station name with minor differences such as special
characters) or station names with "test" in the name, which seem to be invalid/inaccurate entries. Cleaning this
thoroughly would require a significant amount of time to review each row in detail. However, I’ve decided to timebox
this task, and for the purpose of this assignment, I will proceed knowing that we’ve already done as thorough a job as
possible cleaning the data while preserving as much of it as we could. Additionally, in some cases changing the station
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

We'll repeat the same steps as above, starting with removing the "Public Rack" prefix in station names to standardize
the station names for consistency and comparison. We'll check our work before updating the rows in the source table for
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

We will apply the same logic for removing the suffix "(Temp)".

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

Now we will remove any leading or trailing spaces from `station_names`.

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

Next, we'll convert all station names to lowercase to eliminate any inconsistent casing.

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

Now that we've standardized the station names (string data) in the source table, we will remove the rows where the
station data is fully or partially incomplete.

```sql
DELETE
FROM trips
WHERE (start_station_name IS NULL AND start_station_id IS NULL)
   OR (end_station_name IS NULL AND end_station_id IS NULL);
```

### Joining the Cleaned Source Table with the Cleaned Station Data Table

To join the source table `trips` with the cleaned station data table `final_station_data_cleaned` we will use Common
Table Expressions—temporary tables created from select statements using the WITH command—to update the
`start_station_id` and `end_station_id` in the `trips` table by joining them with the corresponding `station_id` from
the `final_station_data_cleaned` table.

In the first CTE we will fix the `start_station_id` for each trip by performing an INNER JOIN between the `trips` table
and `final_station_data_cleaned` table, matching the `start_station_id` in `trips` with the `station_id` in the
`final_station_data_cleaned` table. Building upon the first CTE, the second CTE will take the results of the first CTE
and fix the `end_station_id` for each trip by performing an INNER JOIN between the output of the first CTE, which is the
creation of the `start_station_id_fixed_trips` table and the `final_station_data_cleaned` table. Here, we matched the
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

Creating our final table, with our cleaned, normalized, and merged data.

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

# Phase 4: Analyze

## 4.1 Bike Usage Patterns

### Casuals vs. Members: Most and Least Used Bike Types

Let's begin by getting a breakdown of casuals and members in our dataset. We see that there are ~2.8 million members
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
WITH casual_ss AS (
    SELECT start_station_name, COUNT(start_station_name)
    FROM trips_cleaned
    WHERE member_casual = 'casual'
    GROUP BY start_station_name
    ORDER BY COUNT(start_station_name) DESC
    LIMIT 10
),
member_ss AS (
    SELECT start_station_name, COUNT(start_station_name)
    FROM trips_cleaned
    WHERE member_casual = 'member'
    GROUP BY start_station_name
    ORDER BY COUNT(start_station_name) DESC
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
WITH casual_es AS (
    SELECT end_station_name, COUNT(end_station_name) AS cnt_casual
    FROM trips_cleaned
    WHERE member_casual = 'casual'
    GROUP BY end_station_name
    ORDER BY COUNT(end_station_name) DESC
    LIMIT 10
),
member_es AS (
    SELECT end_station_name, COUNT(end_station_name) AS cnt_member
    FROM trips_cleaned
    WHERE member_casual = 'member'
    GROUP BY end_station_name
    ORDER BY COUNT(end_station_name) DESC
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
WITH casual_trips AS (
    SELECT start_station_name, end_station_name, COUNT(*) AS count
    FROM trips_cleaned
    WHERE member_casual = 'casual'
    GROUP BY start_station_name, end_station_name
    ORDER BY count DESC
    LIMIT 10
),
member_trips AS (
    SELECT start_station_name, end_station_name, COUNT(*) AS count
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
DELIMITER $$

CREATE FUNCTION haversine_distance (lat1 FLOAT, lon1 FLOAT, lat2 FLOAT, lon2 FLOAT)
RETURNS FLOAT
DETERMINISTIC
BEGIN
    DECLARE R INTEGER DEFAULT 6371;  -- Radius of the Earth in kilometers
    DECLARE lat1_rad FLOAT;
    DECLARE lon1_rad FLOAT;
    DECLARE lat2_rad FLOAT;
    DECLARE lon2_rad FLOAT;
    DECLARE dlat FLOAT;
    DECLARE dlon FLOAT;
    DECLARE a FLOAT;
    DECLARE c FLOAT;
    DECLARE distance FLOAT;

    -- Convert degrees to radians
    SET lat1_rad = RADIANS(lat1);
    SET lon1_rad = RADIANS(lon1);
    SET lat2_rad = RADIANS(lat2);
    SET lon2_rad = RADIANS(lon2);

    -- Calculate differences
    SET dlat = lat2_rad - lat1_rad;
    SET dlon = lon2_rad - lon1_rad;

    -- Apply the Haversine formula
    SET a = SIN(dlat / 2) * SIN(dlat / 2) + COS(lat1_rad) * COS(lat2_rad) * SIN(dlon / 2) * SIN(dlon / 2);
    SET c = 2 * ATAN2(SQRT(a), SQRT(1 - a));

    -- Calculate the distance
    SET distance = R * c;

    RETURN distance;  -- Distance in kilometers
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
SELECT HOUR (started_at) as hour_of_day, COUNT(ride_id) as trips_per_hour_of_day
FROM trips_cleaned
WHERE member_casual = 'casual'
GROUP BY hour_of_day
ORDER BY trips_per_hour_of_day DESC;

SELECT HOUR (started_at) as hour_of_day, COUNT(ride_id) as trips_per_hour_of_day
FROM trips_cleaned
WHERE member_casual = 'member'
GROUP BY hour_of_day
ORDER BY trips_per_hour_of_day DESC;
```

# Phase 5: Share

# Phase 6: Act 