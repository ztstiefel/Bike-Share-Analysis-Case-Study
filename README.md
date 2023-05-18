# Bike Share Case Analysis Study

This case study was completed as part of the [Google Data Analytics Certificate](https://grow.google/certificates/data-analytics/#?modal_active=none) course hosted on [Coursera](https://www.coursera.org/google-certificates/data-analytics-certificate?utm_source=google&utm_medium=institutions&utm_campaign=gwgsite).

## Scenario
This case study for a fictional company is based on datasets made public by [Motivate International Inc.](https://motivateco.com/) under the this [license](https://ride.divvybikes.com/data-license-agreement).

In this case study, we are presented with the following business task: To design marketing strategies aimed at converting casual riders into annual members. To do so, we are asked to answer the following guiding questions:

1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?
3. How can Cyclistic use digital media to influence casual riders to become members?

## Preparation

The data for this case study was first downloaded from [here](https://divvy-tripdata.s3.amazonaws.com/index.html). The downloaded data consists of 12 .CSV files containing ride data for each of the last twelve months. In our example, that was from April 2022 - March 2023. After downloading, I consolidated these files to a directory on my personal computer called "Case_Study_Project_01". This dataset has already been stripped of any PII (Personal Identifiable Information). 

[To Be Continued]

## Processing

Immediately after examing the .CSV files I downloaded for this case study, I identified that each .CSV contained hundreds of thousands rows, and that analyzing the data while in a spreadsheet format would be inefficient. As a result, I chose to use a PostgreSQL server I set up a while ago to host this data. This server has minimal resources; It exists as an Ubuntu 22.04 LXC container on a Proxmox hypervisor with 1 CPU and 512MB RAM.

On the PostgreSQL server called "db001", I executed the following query to create the database and its schema based on the columns and content of the .CSV files:

```sql
CREATE TABLE caseStudy01(
	ride_id varchar,
	rideable_type varchar,
	started_at timestamp,
	ended_at timestamp,
	start_station_name varchar,
	start_station_id varchar,
	end_station_name varchar,
	end_station_id varchar,
	start_lat numeric,
	start_lng numeric,
	end_lat numeric,
	end_lng numeric,
	member_casual varchar
);
```

Then, I wrote the following Python script to upload all twelve .CSV files to the newly created table:

```python
import psycopg2, os

dir = 'C:\\Users\\<USER>\\<DIRECTORY>

os.chdir(dir)

conn = psycopg2.connect('host=##.#.#.## port=5432 dbname=<DB_NAME> user=<USER> password=<PASSWORD>')
cur = conn.cursor()

sql = "COPY casestudy01 (ride_id,rideable_type,started_at,ended_at,start_station_name,start_station_id,end_station_name,end_station_id,start_lat,start_lng,end_lat,end_lng,member_casual) FROM STDIN DELIMITER ',' CSV"

for file in os.listdir(dir):
    f = os.path.join(dir, file)
    if os.path.isfile(f):
        with open(f, 'r') as f:
            next(f)
            cur.copy_expert(sql, f)
        conn.commit()

cur.close()
conn.close()
```

After the script finished copying each .CSV file to the PostgreSQL table, I was ready to start querying the table.

## Analysis

1. Determine the total number of rows in the table:

```sql
SELECT
     COUNT(*)
FROM
	casestudy01
```

| |count|
|-|-------|
|1|5803720|

2. Determine how many rides were by members or casual riders:

```sql
SELECT 
     SUM(CASE WHEN member_casual = 'member' THEN 1 ELSE 0 END) AS member_rides,
     SUM(CASE WHEN member_casual = 'casual' THEN 1 ELSE 0 END) AS casual_rides
FROM
     casestudy01
```
|member_ride |casual_rides|
|------------|------------|
|3466281|2337439|

3. Determine what kinds of bikes were preferred by members and casual riders:

```sql
SELECT
     to_char(started_at, 'month') AS month,
     member_casual,
     AVG(ended_at - started_at) AS avg_ride_length
FROM 
     casestudy01
GROUP BY
     month, member_casual
ORDER BY
MIN(started_at)
```
rideable_type|	member_casual|	count
-|-|-
classic_bike	|casual|	889890
classic_bike	|member|	1749669
docked_bike	|casual	|173747
electric_bike	|casual|	1273802
electric_bike	|member|	1716612

Visualization (Tableau):

4. Calculate the average ride length for members and casual riders:

```sql
SELECT
	member_casual,
	 AVG(EXTRACT(EPOCH FROM (ended_at - started_at)) / 60.0) AS avg_ride_length
FROM
	casestudy01
WHERE
	ended_at > started_at
GROUP BY
member_casual
```
member_casual | avg_ride_length
------------- | ---------------
casual        |     28.60512
member        |     12.49493

5. Calculate the average ride length by member / casual rider by day of the week:

```sql
SELECT
    CASE EXTRACT(DOW FROM started_at)
        WHEN 0 THEN 'sunday'
        WHEN 1 THEN 'monday'
        WHEN 2 THEN 'tuesday'
        WHEN 3 THEN 'wednesday'
        WHEN 4 THEN 'thursday'
        WHEN 5 THEN 'friday'
        WHEN 6 THEN 'saturday'
    END AS day_of_week,
    member_casual,
    AVG(ended_at - started_at) AS avg_ride_length
FROM 
     casestudy01
GROUP BY
     EXTRACT(DOW FROM started_at), day_of_week, member_casual
ORDER BY
     EXTRACT(DOW FROM started_at)
```

day_of_week|	member_casual|	avg_ride_length|	avg_ride_length_conv_to_decimal
-|-|-|-
sunday|	casual|	33:32|	33.53|
sunday|	member|	13:50|	13.83|
monday|	casual|	28:28|	28.47|
monday|	member|	12:01|	12.02|
tuesday|	casual|	25:20|	25.33|
tuesday|	member|	11:54|	11.90|
wednesday|	casual|	24:01|	24.02|
wednesday|	member|	11:50|	11.83|
thursday|	casual|	24:49|	24.82|
thursday|	member|	12:06|	12.10|
friday|	casual|	27:42|	27.70|
friday|	member|	12:20|	12.33|
saturday|	casual|	32:20|	32.33|
saturday|	member|	13:58|	13.97|
               
6. Calculate total ridership for each member type by day of the week:

```sql
SELECT
    CASE EXTRACT(DOW FROM started_at)
        WHEN 0 THEN 'sunday'
        WHEN 1 THEN 'monday'
        WHEN 2 THEN 'tuesday'
        WHEN 3 THEN 'wednesday'
        WHEN 4 THEN 'thursday'
        WHEN 5 THEN 'friday'
        WHEN 6 THEN 'saturday'
    END AS day_of_week,
    member_casual,
    COUNT(ride_id) AS ridership
FROM 
    casestudy01
GROUP BY
    EXTRACT(DOW FROM started_at), day_of_week, member_casual
ORDER BY
    EXTRACT(DOW FROM started_at)
```
day_of_week|	member_casual|	ridership
-----------|----------------|----------
sunday|	casual|	389458
sunday|	member|	397990
monday|	casual|	275696
monday|	member|	484571
tuesday|	casual|	271387
tuesday|	member|	544342
wednesday|	casual|	276662
wednesday|	member|	546227
thursday|	casual|	311743
thursday|	member|	551161
friday|	casual|	341012
friday|	member|	489857
saturday|	casual|	471481
saturday|	member|	452133

7. Calculate monthly ridership, broken down by member and casual status:

```sql
SELECT
	TO_CHAR(started_at, 'Month') AS month_name,
	member_casual,
	COUNT(ride_id) AS ridership_count
FROM
	casestudy01
GROUP BY
	month_name, member_casual
ORDER BY
    MIN(started_at)
```

month_name|	member_casual|	ridership_count
----------|-----------------|----------------
April|    member|	244832
April|   casual|	126417
May|      casual|	280415
May|      member|	354443
June|     member|	400153
June|    casual|	369051
July|     casual|	406055
July|     member|	417433
August|   casual|	358924
August|   member|	427008
September|member|	404642
September|casual|	296697
October|  member|	349696
October|  casual|	208989
November| casual|	100772
November|member|	236963
December| member|	136912
December| casual|	44894
January|  casual|	40008
January|  member|	150293
February| member|	147429
February| casual|	43016
March|    member|	196477
March|    casual|	62201

8. Calculate the average ride length of members and casual riders by month:

```sql
SELECT
	to_char(started_at, 'month') AS month,
	member_casual,
	AVG(EXTRACT(EPOCH FROM (ended_at - started_at)) / 60.0) AS avg_ride_length
FROM 
	casestudy01
GROUP BY
	month, member_casual
ORDER BY
	MIN(started_at)
```

month|	member_casual|	avg_ride_length
-----|---------------|-----------------
april|    	member|	11.49240411
april|    	casual|	29.53242707
may|      	casual|	30.86961171
may|      	member|	13.36667687
june|     	member|	13.99843389
june|     	casual|	32.09697521
july|     	casual|	29.27808782
july|     	member|	13.71834015
august|   	casual|	29.31004832
august|   	member|	13.38416349
september|	member|	12.95013996
september|	casual|	27.98516888
october|  	member|	11.95817134
october|  	casual|	26.38742685
november| 	casual|	21.28623526
november| 	member|	11.12862191
december| 	member|	10.61948782
december| 	casual|	22.28956431
january|  	casual|	22.91483995
january|  	member|	10.36176424
february| 	member|	10.71427037
february| 	casual|	23.19251635
march|    	member|	10.44222123
march|    	casual|	21.41227553

After gathering the results of the SQL queries, I was left with a number of .CSV files. However, Tableau will only work with .XLSX files. So, I wrote the following Python script to take all the .CSV files in a directory and convert them to .XLSX for use with Tableau:

```Python
import os
import pandas as pd

dir = 'C:\\Users\\ztsti\\Downloads\\Case_Study_Project_01\\casestudy01_sql_data'

def convert_csv_xlsx():
    for file in os.listdir(dir):
        f = os.path.join(dir, file)
        if os.path.isfile(f):
            df = pd.read_csv(os.path.join(dir, f))
            xlsx_file = os.path.splitext(f)[0] + '.xlsx'
            xlsx_path = os.path.join(dir, xlsx_file)
            df.to_excel(xlsx_path, index=False)

convert_csv_xlsx()
```
After converting all the .CSV files to .XLSX, files, I was ready to begin visualizing the data.

## Share

Visalization (Tableau):

![image](https://github.com/ztstiefel/Bike-Share-Case-Analysis-Study/assets/41418360/180fb7b8-8ef0-44ee-9c54-2ed3ae2bf084)

<img src = https://github.com/ztstiefel/Bike-Share-Case-Analysis-Study/assets/41418360/afa59a9d-e3ad-4159-924e-0dc4d44bd899 width="300" height="500" />

![overall_ride_length_minutes](https://github.com/ztstiefel/Bike-Share-Case-Analysis-Study/assets/41418360/f286a667-bc19-44ca-abe3-70fed3229371)

<img src = https://github.com/ztstiefel/Bike-Share-Case-Analysis-Study/assets/41418360/c9524c3d-4dcb-4913-93d6-324ded16aa11 width = "500" height = "600" />

<img src = https://github.com/ztstiefel/Bike-Share-Case-Analysis-Study/assets/41418360/2917dc3d-3704-4136-aaf6-85cd64ea66c9 width = "500" height = "600" />

<img src = https://github.com/ztstiefel/Bike-Share-Case-Analysis-Study/assets/41418360/40aba75e-65e0-48ac-a1a5-14cf7ac6d7c8 width = "500" height = "600" />

<img src = https://github.com/ztstiefel/Bike-Share-Case-Analysis-Study/assets/41418360/99f7fce7-a27a-4b19-8891-499cfd000397 width = "600" height = "600" />
