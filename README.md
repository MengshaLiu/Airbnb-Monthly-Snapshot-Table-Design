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
#### create table and insert data for month_dim

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
#### create table and insert data for listing_dim

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

#### create table and insert data for host_dim
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
