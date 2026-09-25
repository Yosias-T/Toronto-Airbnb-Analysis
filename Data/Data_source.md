# Data Source

## Source

This project uses publicly available Airbnb listing data obtained from **Inside Airbnb**.

* **Source:** https://insideairbnb.com/
* **Dataset:** Toronto Airbnb Listings

## Dataset Overview

The original dataset contained **21,093 Airbnb listings** and included information related to:

* Listing details and property characteristics
* Host information
* Location and neighbourhood data
* Pricing information
* Availability metrics
* Reviews and estimated performance metrics

## Data Preparation

The raw dataset was cleaned and transformed prior to analysis.

Preparation included:

* Filtering the dataset to active listings only
* Removing redundant, low-value, and non-analytical fields that were not relevant to the project's objectives
* Handling missing and inconsistent values
* Standardizing data types and field formats
* Creating additional classification categories, including rental scope, property groupings, accommodation size categories, and stay length categories
* Normalizing multi-value attributes, such as amenities, into dedicated relational tables
* Structuring the dataset into a normalized relational database design
* Transforming the normalized database into a dimensional reporting model optimized for Power BI analytics

After preparation, the final reporting dataset contained approximately **11,177 active listings**.

## Repository Data

The original source dataset was substantially larger and contained numerous fields not required for analytical reporting. To keep the repository focused, reproducible, and within GitHub file size constraints, the dataset included in this repository is the cleaned reporting source used for analysis.

The included dataset represents the curated output of the data preparation process and contains only the records and attributes required for reporting, modeling, and dashboard development.

## Data Availability

To ensure the project remains reproducible even if the source data changes or becomes unavailable, the cleaned dataset used for SQL modeling, Power BI development, and analysis is included in this repository.
