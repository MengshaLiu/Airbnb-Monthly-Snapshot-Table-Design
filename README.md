# Airbnb-Monthly-Snapshot-Table-Design
Dimensional Data Modelling
## Table of Contents
- [Introduction](#introduction)
- [About Dataset](#about-dataset)
- [data model](#data-model)
- [Implementation on Google BigQuery](#Implementation-on-Google-BigQuery)

## Introduction
In this project, I aim to develop a monthly snapshot data model for Airbnb listings, enabling us to track and measure monthly performance metrics such as income and occupancy. Leveraging a state-of-the-art star schema, this model organizes data into a single periodic fact table and four supporting dimension tables, providing a structured and efficient framework for performance analysis.

## About Dataset
This dataset is sourced from [Inside Airbnb](https://insideairbnb.com/get-the-data/), with data scraped quarterly to provide a comprehensive view of listings. It includes information from four key perspectives: listing, calendar (availability), host, and neighborhood. A detailed data dictionary aproviding explanations for each data field and guidance on data usage, is available [here](https://docs.google.com/spreadsheets/d/1iWCNJcSutYqpULSQHlNyGInUvHg2BoUGoNRIGa6Szc4/edit?gid=1322284596#gid=1322284596). 

## Data Model
![airbnb-data-model](Airbnb-data-model.png)
This Data model comprises of a central fact table, listing_monthly_summary_snapshots, which aggregate income and occupancy metrics monthly for each listing. Connecting this are four dimentional tables(listing_dim, host_dim, month_dim and neighbourhood_dim) offering contextual information on the listing, host, neighbourhood and month.

## Implementation on Google BigQuery
#### Before Start
The two source files (calender.csv.gz and listings.csv.gz) inside the dataset folder must be imported into BigQuery.  
#### Create table and insert data for month_dim

```sql
CREATE TABLE IF NOT EXISTS `airbnb_listings_AU_2024.month_dim` (
  month_id string NOT NULL, 
  year int64 NOT NULL,
  month int64 NOT NULL,
  PRIMARY KEY(month_id) NOT ENFORCED
);
INSERT INTO `airbnb_listings_AU_2024.month_dim`(month_id,year,month)
SELECT
  min(FORMAT_DATE('%Y-%m',date)),
  EXTRACT(YEAR FROM date) AS year, 
  EXTRACT(MONTH FROM date) AS month
FROM `airbnb_listings_AU_2024.calendar` 
GROUP BY year, month;
```
#### Create table and insert data for listing_dim

``` sql
CREATE TABLE IF NOT EXISTS `airbnb_listings_AU_2024.listing_dim` (
  listing_id INT64 NOT NULL,
  listing_name STRING NOT NULL,
  listing_description STRING,
  neighbourhood_overview STRING,
  property_type STRING,
  room_type STRING,
  accommodates INT64,
  bathrooms FLOAT64,
  bedrooms INT64,
  beds INT64,
  minimum_nights INT64,
  maximum_nights INT64,
  review_scores_rating FLOAT64,
  review_scores_accuracy FLOAT64, 
  review_scores_cleanliness FLOAT64,
  review_scores_checkin FLOAT64,
  review_scores_communication FLOAT64,
  review_scores_value FLOAT64, 
  instant_bookable BOOL,
  license STRING,
  PRIMARY KEY(listing_id) NOT ENFORCED
);
```
#### Insert sample data(listing_dim, host_dim, month_dim, neighbourhood_dim) into their dimension table accordingly.
#### Create table for host_dim
```sql
CREATE TABLE IF NOT EXISTS `airbnb_listings_AU_2024.host_dim` (
  host_id INT64 NOT NULL,
  host_since DATE NOT NULL,
  host_location STRING,
  host_about STRING,
  host_response_time STRING,
  host_response_rate FLOAT64, 
  host_acceptance_rate FLOAT64,
  host_is_superhost BOOL,
  host_neighbourhood STRING,
  host_listings_count INT64 NOT NULL, 
  host_verifications STRING,
  host_has_profile_pic BOOL, 
  host_identity_verified BOOL,
  PRIMARY KEY(host_id) NOT ENFORCED
);


  ```
#### Create table and insert data for neighbourhood_dim
```sql
CREATE OR REPLACE TABLE `airbnb_listings_AU_2024.neighbourhood_dim`(
  neighbourhood STRING,
  city STRING,
  country STRING,
  PRIMARY KEY(neighbourhood) NOT ENFORCED
```
#### Create stage table and monthly_listing_summary_snapshots fact table
``` SQL

#Create schema for staging table
CREATE OR REPLACE TABLE `airbnb_listings_AU_2024.monthly_listing_summary_staging`(
listing_id INT64 NOT NULL,
host_id INT64 NOT NULL,
neighbourhood STRING NOT NULL,
month_id STRING NOT NULL,
monthly_income FLOAT64,
monthly_occupancy INT64,
monthly_occupancy_rate FLOAT64,
load_time DATE
);

CREATE OR REPLACE TABLE `my-data-project-65962.airbnb_listings_AU_2024.listing_monthly_summary_snapshots`(
listing_id INT64 NOT NULL,
host_id INT64 NOT NULL,
neighbourhood STRING NOT NULL,
month_id STRING NOT NULL,
monthly_income FLOAT64,
monthly_occupancy INT64,
monthly_occupancy_rate FLOAT64,
load_time DATE,
PRIMARY KEY(listing_id,host_id,neighbourhood, month_id) NOT ENFORCED,
FOREIGN KEY(listing_id) REFERENCES airbnb_listings_AU_2024.listing_dim(listing_id) NOT ENFORCED,
FOREIGN KEY(host_id) REFERENCES airbnb_listings_AU_2024.host_dim(host_id) NOT ENFORCED,
FOREIGN KEY(neighbourhood) REFERENCES airbnb_listings_AU_2024.neighbourhood_dim(neighbourhood) NOT ENFORCED,
FOREIGN KEY(month_id) REFERENCES airbnb_listings_AU_2024.month_dim(month_id) NOT ENFORCED
);
```

#### Insert monthly records aggregated from the source table into stage table
``` sql
# Declair the current_load_time varieble. This is useful whenthere is a event of failure, we know the point in time we have to restart the process from.
DECLARE current_load_time DATE DEFAULT CURRENT_DATE();

#Populate data from source tables and insert value into staging table
#Assume that the monthly_listing_snapshots fact table is updated monthly at the start of every month

INSERT INTO `airbnb_listings_AU_2024.monthly_listing_summary_staging` (
  listing_id,
  host_id,
  neighbourhood,
  month_id,
  monthly_income,
  monthly_occupancy,
  monthly_occupancy_rate,
  load_time
)
with calendar_monthly AS (
  SELECT 
    listing_id, 
    EXTRACT(YEAR FROM date) AS year, 
    EXTRACT(MONTH FROM date) AS month, 
    COUNT(*) AS monthly_occupancy,
    ROUND(COUNT(*)/EXTRACT(DAY FROM LAST_DAY(min(date))),2) AS monthly_occupancy_rate,  
    SUM(price) AS monthly_income
  FROM `project-65962.airbnb_listings_AU_2024.calendar` c
  WHERE available = false AND EXTRACT(YEAR FROM date) = EXTRACT( YEAR FROM CURRENT_DATE()) AND EXTRACT(MONTH FROM date) = EXTRACT( MONTH FROM CURRENT_DATE())-1
  GROUP BY listing_id, year, month)
SELECT 
  l.id AS listing_id, 
  l.host_id,
  l.neighbourhood_cleansed AS neighbourhood,
  COALESCE(m.month_id, CONCAT(EXTRACT(YEAR FROM current_date()),'-',(EXTRACT(MONTH FROM current_date())))) AS month_id,
  COALESCE(c.monthly_income,0) AS monthly_income, 
  COALESCE(c.monthly_occupancy,0) AS monthly_occupancy,
  COALESCE(c.monthly_occupancy_rate,0) AS monthly_occupancy_rate,
  current_load_time
FROM `airbnb_listings_AU_2024.listings` l
LEFT JOIN calendar_monthly c
ON l.id = c.listing_id
LEFT JOIN `project-65962.airbnb_listings_AU_2024.month_dim` m
ON m.year = c.year AND m.month = c.month;
```
#### Insert newly added monthly records to  listing_monthly_summary_snapshots table based on the result of left join staging table with fact table.
```sql
INSERT INTO `airbnb_listings_AU_2024.listing_monthly_summary_snapshots`(
  listing_id,
  host_id,
  neighbourhood,
  month_id,
  monthly_income,
  monthly_occupancy,
  monthly_occupancy_rate,
  load_time
)
SELECT *
FROM `airbnb_listings_AU_2024.monthly_listing_summary_staging` s
WHERE NOT EXISTS(
  SELECT listing_id 
  FROM `airbnb_listings_AU_2024.listing_monthly_summary_snapshots` f 
  WHERE f.listing_id = s.listing_id AND f.month_id=s.month_id
);
```
#### Cleanup staging table
``` sql
TRUNCATE TABLE `airbnb_listings_AU_2024.monthly_listing_summary_staging`
```
