# Data Source

## Source

This project uses publicly available Airbnb listing data obtained from **Inside Airbnb**.

* **Source:** https://insideairbnb.com/
* **Dataset:** Toronto Airbnb Listings
* **Download Date:** July 24, 2025

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

* Removing inactive listings from the dataset
* Removing redundant, low-value, and non-analytical fields that were not relevant to the project's objectives
* Handling missing and inconsistent values
* Standardizing fields and data formats
* Creating additional classification categories, including listing type, property groupings, and accommodation categories
* Normalizing multi-value attributes, such as amenities, into dedicated relational tables
* Structuring the dataset into a normalized relational database format

After preparation, the final dataset contained approximately **11,177 active listings**.

## Data Availability

The dataset used for this project was downloaded from Inside Airbnb on July 24, 2025. To ensure the project remains reproducible even if the source data changes or becomes unavailable, the cleaned dataset used for analysis is included in this repository.
