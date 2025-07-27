# Case Study Practice 2 
This project serves as the first hand-on practice on my Business Intelligence (BI) skills. It is based on a given fictional topic and with some guidances from the course I am taking. BigQuery, SQL and Tableau will be used in the ETL process and data visualisation. Since some of the information from the course, I will try to explain the process as the course did give exemplars for references in the end. It aims to show all my understanding and thought process here.

### Introduction
**Situation** 

Imagine I am a business analyst in Cyclistic, a fictional bike-share company in New York City. The growth team wants to understand how their customers are using their bikes; their top priority is identifying customer demand at different station locations.

To keep it simple, I will make a brief summary here. For details, please refer to the three exemplar documents in the repository.

**Stakeholder requirements**

R - required 

•	A table or map visualization exploring starting and ending station locations, aggregated by location.

•	A visualization showing which destination (ending) locations are popular based on the total trip minutes.

•	A visualization showing the percent growth in the number of trips year over year.

•	Gather insights about the number of trips across all starting and ending locations.

•	Gather insights about peak usage by time of day, season, and the impact of weather.

D - desired 

•	A visualization that focuses on trends from the summer of 2015.


**Datasets Given**

Most of them are sources in BigQuery, I will examine them later. 

Primary dataset: 

[NYC Citi Bike Trips](https://console.cloud.google.com/marketplace/details/city-of-new-york/nyc-citi-bike?inv=1&invt=Ab35KA)

Secondary datasets: 

[Census Bureau US Boundaries](https://console.cloud.google.com/marketplace/product/united-states-census-bureau/us-geographic-boundaries?inv=1&invt=Ab35KA&project=iconic-medium-466721-f4)

[GSOD from the National Oceanic and Atmospheric Administration](https://console.cloud.google.com/marketplace/details/noaa-public/gsod?inv=1&invt=Ab35Kg&project=iconic-medium-466721-f4)

[Zip code spreadsheet](https://docs.google.com/spreadsheets/d/1IIbH-GM3tdmM5tl56PHhqI7xxCzqaBCU0ylItxk_sy0/template/preview#gid=806359255)

### ETL

After the business tasks are clearly defined, I may begin to the work of data flow.

To begin with, let investigate what major schema is in the datasets.

NYC Citi Bike Trips:

Trip duration, user type (Customer = 24-hour pass or 7-day pass user, Subscriber = Annual Member), starting and ending time and location (station id, name, latitude and longtitude), bike ID

Census Bureau US Boundaries & Zip code spreadsheet

zip codes and location (state/city/borought/neightbourhood etc.)

GSOD from the National Oceanic and Atmospheric Administration

date, temperature, station number, windspeed, precipitation and other weather measurements

Luckily, these datasets are from BigQuery, which are pretty reliable sources and does not required cleaning, yet I suggest to do a preliminary checking first.

(Since the dataset of bike trips has already stopped update from 2018, there is no way to set up a data pipeline)

Now, let's move on to the ETL part. I will explain the SQL command.

```
SELECT
  TRI.usertype,
  ZIPSTART.zip_code AS zip_code_start,
  ZIPSTARTNAME.borough borough_start,
  ZIPSTARTNAME.neighborhood AS neighborhood_start,
  ZIPEND.zip_code AS zip_code_end,
  ZIPENDNAME.borough borough_end,
  ZIPENDNAME.neighborhood AS neighborhood_end,
  WEA.temp AS day_mean_temperature, -- Mean temp
  WEA.wdsp AS day_mean_wind_speed, -- Mean wind speed
  WEA.prcp day_total_precipitation, -- Total precipitation
  -- Group trips into 10 minute intervals to reduces the number of rows
  ROUND(CAST(TRI.tripduration / 60 AS INT64), -1) AS trip_minutes,
  COUNT(TRI.bikeid) AS trip_count
FROM
  `bigquery-public-data.new_york_citibike.citibike_trips` AS TRI
INNER JOIN
  `bigquery-public-data.geo_us_boundaries.zip_codes` ZIPSTART
  ON ST_WITHIN(
    ST_GEOGPOINT(TRI.start_station_longitude, TRI.start_station_latitude),
    ZIPSTART.zip_code_geom)
INNER JOIN
  `bigquery-public-data.geo_us_boundaries.zip_codes` ZIPEND
  ON ST_WITHIN(
    ST_GEOGPOINT(TRI.end_station_longitude, TRI.end_station_latitude),
    ZIPEND.zip_code_geom)
INNER JOIN
  `bigquery-public-data.noaa_gsod.gsod20*` AS WEA
  ON PARSE_DATE("%Y%m%d", CONCAT(WEA.year, WEA.mo, WEA.da)) = DATE(TRI.starttime)
INNER JOIN
  `BIcapstone.zip_codes` AS ZIPSTARTNAME
  ON ZIPSTART.zip_code = CAST(ZIPSTARTNAME.zip AS STRING)
INNER JOIN
  `BIcapstone.zip_codes` AS ZIPENDNAME
  ON ZIPEND.zip_code = CAST(ZIPENDNAME.zip AS STRING)
WHERE
  -- This takes the weather data from one weather station
  WEA.wban = '94728' -- NEW YORK CENTRAL PARK
  -- Use data from 2014 and 2015
  AND EXTRACT(YEAR FROM DATE(TRI.starttime)) BETWEEN 2014 AND 2015
--AND DATE(TRI.starttime) BETWEEN DATE('2015-07-01') AND DATE('2015-09-30')  // To obtain a table dedicated for 2015 summer
GROUP BY
  1,
  2,
  3,
  4,
  5,
  6,
  7,
  8,
  9,
  10,
  11,
  12,
  13
```

Basically, from the ```SELECT``` section, there a few main tables appeared. TRI is meant by TRIP, the main sheet which is NYC Citi Bike Trips. ZIPSTART is used to retrieve the zip code of the starting location, while ZIPSTARTNAME would be the name of the starting location, and so do ZIPEND and ZIPENDNAME. Note that the zip code acts as a primary key on matching the actual location. The column names are self-explanatory. Here concludes all the time and location for each trip.

Next, we have to know the weather condition of every day, so the mean temperature, wind speed, and total precipitation are extracted. One may add extra information to investigate, such as fog and snow, but it is not a common weather in New York.

Last but not least, the duration of trip and the number of trips are calculated. Note that the trip duration was recorded as in interger and seconds, but we want it to be in minutes. It is rounded into the nearest 10 minutes for better grouping, but the CAST is not a must, it can avoid any data type accidentally input as string or something else.

The follow part is all about inner join, that is combining all the needed information into a target table. 

ST_WITHIN enables location (by using ST_GEOPOINT to convert latitude and longitude to a geograpgical data type) act as a outside key to connect other tables and gives to corresponding locations information such as zip codes, neighborhood and so on. As for the weather information, here a asterisk (*) is used to allow merging from an indefinite table. In other words, the notation ```gsod20*``` allows finding year 2000-2099 (if any). The inner join is completed by matching the date in both table. For zip code, as it is stored as a string in US Boundaries, a cast function is used.

Finally, the ```WHERE``` section filters out the New York area by using the weather station ID and extract a specific period for the analysis. The numbers under ```GROUP BY``` represents the ordinal columns. The output table is saved when the query is completed.


### Dashboard





