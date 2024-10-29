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
#### create table for the four dimention tables and insert the value accrodingly
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
#### create table for the four dimtion tables accrodingly
