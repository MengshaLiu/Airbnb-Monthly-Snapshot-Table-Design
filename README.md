# Airbnb-Monthly-Snapshot-Table-Design
Dimensional Data Modelling
## Table of Contents
- [Introduction](#introduction)
- [About Dataset](#about-dataset)
- [datamodel](#data-model)
- [Technologies Used](#technologies-used)
- [Data Source](#data-source)
- [Highlighted Features](#highlighted-features)
- [Results](#results)
- [Contact](#contact)

## Introduction
In this project, I aim to develop a monthly snapshot data model for Airbnb listings, enabling us to track and measure monthly performance metrics such as income and occupancy. Leveraging a state-of-the-art star schema, this model organizes data into a single periodic fact table and four supporting dimension tables, providing a structured and efficient framework for performance analysis.

## About Dataset
This dataset is sourced from [Inside Airbnb](https://insideairbnb.com/get-the-data/), with data scraped quarterly to provide a comprehensive view of listings. It includes information from four key perspectives: listing, calendar (availability), host, and neighborhood. A detailed data dictionary aproviding explanations for each data field and guidance on data usage, is available [here](https://docs.google.com/spreadsheets/d/1iWCNJcSutYqpULSQHlNyGInUvHg2BoUGoNRIGa6Szc4/edit?gid=1322284596#gid=1322284596). 

## Data Model
![airbnb-data-model](Airbnb-data-model.png)
This Data model comprises of a central fact table, monthly_listing_summary_snapshots, which aggregate income and occupancy metrics monthly for each listing. Connecting this are four dimentional tables(listing_dim, host_dim, month_dim and neighbourhood_dim) offering contextual information on the listing, host, neighbourhood and month.

## Implementation on Google BigQuery
#### Before Start

#### Create table and insert data for month_dim

```sql
CREATE TABLE IF NOT EXISTS `my-data-project-65962.airbnb_listings_AU_2024.month_dim` (
  month_id string NOT NULL, 
  year int64 NOT NULL,
  month int64 NOT NULL,
  PRIMARY KEY(month_id) NOT ENFORCED
);
INSERT INTO `my-data-project-65962.airbnb_listings_AU_2024.month_dim`(month_id,year,month)
SELECT
  min(FORMAT_DATE('%Y-%m',date)),
  EXTRACT(YEAR FROM date) AS year, 
  EXTRACT(MONTH FROM date) AS month
FROM `my-data-project-65962.airbnb_listings_AU_2024.calendar` 
GROUP BY year, month;
```
#### Create table and insert data for listing_dim

``` sql
CREATE TABLE IF NOT EXISTS `my-data-project-65962.airbnb_listings_AU_2024.listing_dim` (
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


INSERT INTO `my-data-project-65962.airbnb_listings_AU_2024.listing_dim`(
  listing_id, 
  listing_name, listing_description, neighbourhood_overview,
  property_type, 
  room_type, 
  accommodates,
  bathrooms,
  bedrooms,
  beds,
  minimum_nights,
  maximum_nights,
  review_scores_rating,
  review_scores_accuracy,
  review_scores_cleanliness,
  review_scores_checkin,
  review_scores_communication,
  review_scores_value,
  instant_bookable,
  license) 
SELECT 
  id AS listing_id, 
  name AS listing_name, 
  description AS listing_description, 
  neighborhood_overview AS neighbourhood_overview,
  property_type, 
  room_type,
  accommodates,
  bathrooms,
  bedrooms,
  beds,
  minimum_nights,
  maximum_nights,
  review_scores_rating,
  review_scores_accuracy,
  review_scores_cleanliness,
  review_scores_checkin,
  review_scores_communication,
  review_scores_value,
  instant_bookable,
  license
FROM `my-data-project-65962.airbnb_listings_AU_2024.listings`;
```

#### Create table and insert data for host_dim
```sql
CREATE TABLE IF NOT EXISTS `my-data-project-65962.airbnb_listings_AU_2024.host_dim` (
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

INSERT INTO `my-data-project-65962.airbnb_listings_AU_2024.host_dim`(
  host_id,
  host_since,
  host_location,
  host_about,
  host_response_time,
  host_response_rate, 
  host_acceptance_rate,
  host_is_superhost,
  host_neighbourhood,
  host_listings_count, 
  host_verifications,
  host_has_profile_pic, 
  host_identity_verified
)
SELECT distinct host_id,
  host_since,
  host_location,
  host_about,
  host_response_time,
  CASE WHEN host_response_rate = 'N/A' THEN NULL
        ELSE CAST(SUBSTRING(host_response_rate,1, LENGTH(host_response_rate)-1) AS INT64) 
        END AS host_response_rate, 
  CASE WHEN host_acceptance_rate = 'N/A' THEN NULL
        ELSE CAST(SUBSTRING(host_acceptance_rate,1, LENGTH(host_acceptance_rate)-1) AS INT64) 
        END AS host_acceptance_rate,
  host_is_superhost,
  host_neighbourhood,
  host_listings_count, 
  host_verifications,
  host_has_profile_pic, 
  host_identity_verified FROM `my-data-project-65962.airbnb_listings_AU_2024.listings`;
  ```
#### Create table and insert data for neighbourhood_dim
```sql
CREATE OR REPLACE TABLE `my-data-project-65962.airbnb_listings_AU_2024.neighbourhood_dim`(
  neighbourhood STRING,
  city STRING,
  country STRING,
  PRIMARY KEY(neighbourhood) NOT ENFORCED
);
INSERT INTO `my-data-project-65962.airbnb_listings_AU_2024.neighbourhood_dim`(
  neighbourhood,
  city,
  country
)
SELECT 
  neighbourhood,
  city,
  country
FROM `my-data-project-65962.airbnb_listings_AU_2024.neighbourhoods`;
```
#### Create table and insert data for monthly_listing_summary_snapshots
``` sql
CREATE OR REPLACE TABLE `my-data-project-65962.airbnb_listings_AU_2024.listing_monthly_summary_snapshots`(
listing_id INT64 NOT NULL,
host_id INT64 NOT NULL,
neighbourhood STRING NOT NULL,
month_id STRING NOT NULL,
monthly_income FLOAT64,
monthly_occupancy INT64,
monthly_occupancy_rate FLOAT64,
PRIMARY KEY(listing_id,host_id,neighbourhood, month_id) NOT ENFORCED,
FOREIGN KEY(listing_id) REFERENCES airbnb_listings_AU_2024.listing_dim(listing_id) NOT ENFORCED,
FOREIGN KEY(host_id) REFERENCES airbnb_listings_AU_2024.host_dim(host_id) NOT ENFORCED,
FOREIGN KEY(neighbourhood) REFERENCES airbnb_listings_AU_2024.neighbourhood_dim(neighbourhood) NOT ENFORCED,
FOREIGN KEY(month_id) REFERENCES airbnb_listings_AU_2024.month_dim(month_id) NOT ENFORCED
);

INSERT INTO my-data-project-65962.airbnb_listings_AU_2024.listing_monthly_summary_snapshots(
  listing_id,
  host_id,
  neighbourhood,
  month_id,
  monthly_income,
  monthly_occupancy,
  monthly_occupancy_rate
)
with last_day AS (
  SELECT 
    EXTRACT(DAY FROM LAST_DAY(min(date))) AS last_day, 
    EXTRACT(MONTH FROM date) as month, 
    EXTRACT(YEAR FROM date) as year 
  FROM `my-data-project-65962.airbnb_listings_AU_2024.calendar` 
    GROUP BY month, year),
listing_calendar as (
  SELECT 
    listing_id, 
    EXTRACT(YEAR FROM date) AS year, 
    EXTRACT(MONTH FROM date) AS month, 
    COUNT(*) AS monthly_occupancy, 
    SUM(price) AS monthly_income
  FROM `my-data-project-65962.airbnb_listings_AU_2024.calendar` 
  WHERE available = false
  GROUP BY listing_id, year, month)
SELECT 
  lc.listing_id, 
  lt.host_id,
  lt.neighbourhood_cleansed AS neighbourhood,
  CONCAT(lc.year,"-", lc.month) AS month_id,
  lc.monthly_income, 
  lc.monthly_occupancy, 
  lc.monthly_occupancy/ld.last_day AS monthly_occupacy_rate 
FROM listing_calendar lc
LEFT JOIN last_day ld ON lc.month=ld.month AND lc.year=ld.year
LEFT JOIN `my-data-project-65962.airbnb_listings_AU_2024.listings` lt ON lc.listing_id = lt.id
'''

