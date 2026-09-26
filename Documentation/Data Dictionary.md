# Data Dictionary — Toronto Airbnb Market Analysis

This document defines every table and field in the Power BI reporting model, which follows a star-schema design: one fact table (`FACT_LISTINGS`) surrounded by four dimension tables (`DIM_HOSTS`, `DIM_NEIGHBOURHOODS`, `DIM_CLASSIFICATIONS`, `DIM_AMENITIES`) and one bridge table (`BRIDGE_AMENITIES`) resolving the many-to-many relationship between listings and amenities.

Data types reflect the PostgreSQL types used to build the reporting schema (see [`SQL_Code.md`](./SQL_Code.md)). "Business Purpose" describes how each field is actually used in analysis and reporting, not just what it technically stores.

> **Note on naming:** several fields carry different names at different stages of the project (e.g. `property_group` in the source model → `property_category` here). Where relevant, the prior name is noted for cross-reference with [`Source_Schema.md`](./Source_Schema.md).

---

## DIM_HOSTS

**Description:** One row per unique Airbnb host. Supports host-level analysis — for example, comparing hosts who manage a single listing against multi-listing hosts or Superhosts, and understanding how a host's portfolio composition (entire homes vs. private/shared rooms) relates to their overall scale in the market.

| Column Name | Data Type | Description | Business Purpose |
|---|---|---|---|
| `host_key` | `BIGINT` | Surrogate primary key generated for the reporting model, replacing the original Airbnb `host_id`. | Enables stable relationships between `FACT_LISTINGS` and this dimension, independent of any changes to source-system host identifiers. |
| `host_name` | `VARCHAR(50)` | The host's display name as listed on Airbnb. | Used for listing-level detail views and host identification in drill-through analysis. |
| `host_since` | `DATE` | The date the host's Airbnb account was created. | Enables tenure-based segmentation — e.g. comparing performance of long-tenured hosts against newer entrants to the Toronto market. |
| `host_location` | `VARCHAR(75)` | The host's self-reported location. | Indicates whether a host is locally based in Toronto or managing listings remotely; relevant to understanding local vs. absentee/professional management patterns. |
| `host_is_superhost` | `BOOLEAN` | Whether the host holds Airbnb's "Superhost" status. | A trust/quality signal; used to evaluate whether Superhost status is associated with stronger listing performance (revenue, occupancy, reviews). |
| `total_listings_count` | `SMALLINT` | Total number of listings associated with the host across Toronto, as calculated by Airbnb. | Distinguishes single-listing hosts from multi-listing operators/property managers — a key driver of scale and portfolio-level strategy. |
| `entire_home_listings` | `SMALLINT` | Count of the host's listings categorized as entire home/apartment rentals. | Supports analysis of host specialization — hosts who focus on entire-property rentals vs. shared-access models. |
| `private_room_listings` | `SMALLINT` | Count of the host's listings offering a private room within a shared property. | Same purpose as above, isolating the private-room segment of a host's portfolio. |
| `shared_room_listings` | `SMALLINT` | Count of the host's listings offering a shared room. | Completes the host's rental-scope portfolio breakdown; shared rooms are typically the smallest and lowest-revenue segment. |

*Assumption: `total_listings_count`, `entire_home_listings`, `private_room_listings`, and `shared_room_listings` are assumed to be host-level aggregates as calculated by Airbnb at the time of data collection (i.e., a snapshot, not a live count), consistent with how Inside Airbnb sources this data.*

---

## DIM_NEIGHBOURHOODS

**Description:** One row per Toronto neighbourhood, with both a granular neighbourhood name and a broader geographic grouping. Supports location-based analysis at two levels of detail — from citywide area comparisons down to individual neighbourhood rankings.

| Column Name | Data Type | Description | Business Purpose |
|---|---|---|---|
| `neighbourhood_id` | `BIGINT` | Surrogate primary key for the neighbourhood dimension. | Joins `FACT_LISTINGS` to this dimension for all location-based analysis and filtering. |
| `neighbourhood_name` | `VARCHAR(50)` | The specific Toronto neighbourhood name, as defined by the City of Toronto's official neighbourhood boundaries (via the source data's `neighbourhood_cleansed` field). | Powers granular, neighbourhood-level analysis — e.g. the Top 10 Neighbourhoods by Median Revenue view, used to identify specific high-value micro-markets. |
| `neighbourhood_group` | `VARCHAR(50)` | A broader geographic cluster (e.g. Downtown Core, Midtown, North York, Scarborough, Etobicoke, East End, West End) that the neighbourhood belongs to. Displayed in the Power BI report as **Neighbourhood Area**. | Reduces ~140 individual neighbourhoods to a manageable set of analytical regions, making area-level comparisons and slicers usable without overwhelming the report with too many categories. |

*Assumption: the mapping of individual neighbourhoods to broader groups reflects Toronto's commonly recognized district boundaries (e.g. former pre-amalgamation municipalities), applied manually during data preparation rather than sourced from an official geographic dataset.*

---

## DIM_CLASSIFICATIONS

**Description:** One row per unique combination of property and rental characteristics (208 combinations in the current dataset). This is a consolidated "junk dimension" — rather than maintaining four or five separate low-cardinality dimension tables, the classification attributes that are frequently filtered together are combined into a single dimension, which is both a common dimensional-modeling technique and a query-performance optimization.

| Column Name | Data Type | Description | Business Purpose |
|---|---|---|---|
| `classification_key` | `BIGINT` | Surrogate primary key for this dimension. | Joins `FACT_LISTINGS` to a single set of classification attributes, avoiding repeated category text on every fact row. |
| `rental_scope` | `VARCHAR(20)` | Whether the listing offers exclusive use of the property or shared/partial access: **Entire**, **Private Room**, or **Shared Room**. | A primary segmentation axis throughout the report; rental scope is one of the strongest differentiators of both price point and guest experience. |
| `property_category` | `VARCHAR(20)` | A standardized property classification (Apartment, House, Hotel, Villa, Castle, Farm, Other), derived from dozens of raw Airbnb property-type values. Referred to as `property_group` in the source/normalized schema. | Enables meaningful property-type comparisons that wouldn't be possible against the raw, highly fragmented `property_type` field — the basis for the report's Property Category-level revenue and occupancy analysis. |
| `accommodation_size` | `VARCHAR(20)` | A guest-capacity band — **Small** (1–2), **Medium** (3–4), **Large** (5–7), or **Group** (8+) — derived from the listing's `accommodates` value. Referred to as `accommodate_group` in the source schema and displayed in Power BI as **Listing Size**. | Groups listings into comparable size tiers for revenue/occupancy analysis, since raw guest-capacity values are too granular to analyze individually. |
| `stay_length_category` | `VARCHAR(20)` | A minimum-stay classification — **Short-term** (<28 nights), **Long-term** (28–89), **Seasonal** (90–179), or **Lease-style** (180+) — derived from the listing's minimum night requirement. | Distinguishes classic short-term rentals from longer-term stays, which behave differently in both pricing and regulatory context (Toronto's short-term rental bylaws influence minimum-stay behavior). |
| `property_type` | `VARCHAR(45)` | The original, unstandardized Airbnb property type text (e.g. "Entire rental unit," "Private room in bungalow"). | Preserved alongside the standardized `property_category` for transparency and to support drill-down into the most granular level of property detail when needed. |

---

## DIM_AMENITIES

**Description:** One row per unique, standardized amenity offered across all listings. Works together with `BRIDGE_AMENITIES` to resolve the many-to-many relationship between listings and amenities — a single listing can offer dozens of amenities, and a single amenity (e.g. Wi-Fi) can appear on thousands of listings.

| Column Name | Data Type | Description | Business Purpose |
|---|---|---|---|
| `amenity_id` | `BIGINT` | Surrogate primary key for the amenity dimension. | Joins `BRIDGE_AMENITIES` to a single, deduplicated amenity name. |
| `amenity` | `VARCHAR(100)` | The standardized amenity name (e.g. "Wifi," "Air Conditioning," "Lake View"), consolidated from dozens of raw wording variants in the source data. | Powers all amenity-level analysis in the report — most common amenities, amenities associated with higher revenue/occupancy, and amenity prevalence-vs-performance comparisons. Standardization is what makes this analysis possible at all; without it, wording variants would fragment the same amenity into dozens of near-duplicate categories. |

---

## FACT_LISTINGS

**Description:** The central fact table, with one row per active Toronto Airbnb listing (~11,177 rows). Each row represents a single listing's attributes, pricing, availability, review performance, and estimated financial performance over the trailing 365 days. This is the grain the entire report is built on — every measure in the Power BI model ultimately aggregates from this table.

| Column Name | Data Type | Description | Business Purpose |
|---|---|---|---|
| `listing_id` | `BIGINT` | Surrogate primary key uniquely identifying each listing. | The fact table's grain; the join key for every measure and every dimension relationship in the model. |
| `host_key` | `BIGINT` | Foreign key to `DIM_HOSTS`. | Links each listing to its host, enabling host-level rollups and Superhost/tenure-based analysis. |
| `neighbourhood_id` | `BIGINT` | Foreign key to `DIM_NEIGHBOURHOODS`. | Links each listing to its location for all geographic analysis. |
| `classification_key` | `BIGINT` | Foreign key to `DIM_CLASSIFICATIONS`. | Links each listing to its property category, listing size, stay length category, rental scope, and raw property type. |
| `listing_name` | `TEXT` | The listing's title as displayed on Airbnb. | Used in listing-level detail views (e.g. the Top 15 Listings by Revenue table) for human-readable identification. |
| `room_type` | `VARCHAR(20)` | Airbnb's native room-type classification (Entire home/apt, Private room, Shared room, Hotel room). | Retained as the original source-system classification alongside the derived `rental_scope` field, for traceability back to Airbnb's own categorization. |
| `accommodates` | `SMALLINT` | Maximum guest capacity of the listing. | The raw value underlying the `accommodation_size` (Listing Size) classification; also useful for precise, non-binned capacity analysis. |
| `bathrooms` | `NUMERIC(3,1)` | Number of bathrooms (supports half-bath values, e.g. 1.5). | A core property-size attribute used in listing-level detail and quality assessment. |
| `bedrooms` | `SMALLINT` | Number of bedrooms. | Same purpose as `bathrooms` — a standard property-size attribute for detail views. |
| `beds` | `SMALLINT` | Number of beds. | Same purpose as above; distinct from bedrooms since a single bedroom can contain multiple beds. |
| `price` | `NUMERIC(7,2)` | Nightly listing price in CAD, as set by the host (with a small number of missing values imputed — see `Source_Schema.md`). | The base pricing metric that, combined with occupancy, determines estimated revenue; also the basis for average price analysis. |
| `is_price_outlier` | `BOOLEAN` | Flags listings priced above the IQR-based upper fence ($434/night). | Allows analysis to include or exclude unusually high-priced listings, preventing a small number of luxury outliers from distorting average price or revenue figures. |
| `minimum_nights` | `INT` | The minimum number of nights a guest must book. | The raw value underlying `stay_length_category`; also directly relevant to understanding regulatory-driven booking constraints. |
| `maximum_nights` | `INT` | The maximum number of nights a guest may book. | Complements `minimum_nights` in describing a listing's booking-length policy. |
| `availability_30` | `SMALLINT` | Number of days available for booking in the next 30 days. | A short-term, forward-looking availability signal, useful for understanding near-term booking pressure. |
| `availability_365` | `SMALLINT` | Number of days available for booking in the next 365 days. | A longer-term availability signal; low values can indicate a listing that is heavily booked or intentionally restricted. |
| `availability_eoy` | `SMALLINT` | Number of days available for booking through the end of the current calendar year. | Supports seasonal/year-end booking-pressure analysis distinct from the rolling 30/365-day windows. |
| `number_of_reviews` | `INT` | Total number of reviews received over the listing's lifetime. | A proxy for a listing's overall booking history and guest engagement over time. |
| `number_of_reviews_ltm` | `SMALLINT` | Number of reviews received in the last twelve months. | A more current activity signal than lifetime review count, useful for assessing recent guest engagement. |
| `number_of_reviews_ly` | `SMALLINT` | Number of reviews received in the prior calendar year. | Supports year-over-year review-volume comparison. |
| `estimated_occupancy_l365d` | `SMALLINT` | Estimated number of nights booked over the trailing 365 days. | One of the two primary target variables in this project; the core occupancy metric behind every occupancy-related measure in the report. |
| `estimated_revenue_l365d` | `INT` | Estimated revenue over the trailing 365 days (derived from price × estimated occupancy where not directly available). | The primary target variable in this project — the central metric the entire analysis is built to explain. |
| `first_review_date` | `DATE` | Date of the listing's first review. | Serves as a practical proxy for how long a listing has been actively hosting guests. |
| `last_review_date` | `DATE` | Date of the listing's most recent review. | A recency signal indicating whether a listing is still actively receiving bookings. |
| `overall_rating` | `NUMERIC(3,2)` | The listing's overall guest rating. | The headline quality metric surfaced alongside revenue rankings (e.g. in the Top 15 Listings by Revenue table), adding a quality lens to performance analysis. |
| `accuracy_score` | `NUMERIC(3,2)` | Sub-rating for how accurately the listing was described. | One of Airbnb's six standard review sub-categories; supports more granular quality analysis than the overall rating alone. |
| `cleanliness_score` | `NUMERIC(3,2)` | Sub-rating for listing cleanliness. | Same purpose as above; cleanliness is typically one of the more revenue-sensitive sub-ratings. |
| `checkin_score` | `NUMERIC(3,2)` | Sub-rating for the check-in experience. | Same purpose as above. |
| `communication_score` | `NUMERIC(3,2)` | Sub-rating for host communication quality. | Same purpose as above; also reflects host responsiveness and service quality. |
| `location_score` | `NUMERIC(3,2)` | Sub-rating for the listing's location. | Same purpose as above; can be cross-referenced against neighbourhood-level revenue patterns. |
| `value_score` | `NUMERIC(3,2)` | Sub-rating for perceived value for money. | Same purpose as above; particularly relevant when analyzed alongside price and revenue. |
| `reviews_per_month` | `NUMERIC(5,2)` | Average number of reviews received per month since the listing's first review. | A normalized booking-velocity metric that accounts for how long a listing has been active, unlike raw review counts. |
| `instant_bookable` | `BOOLEAN` | Whether the listing can be booked immediately without host approval. | A booking-friction indicator; relevant to understanding whether ease of booking correlates with occupancy performance. |
| `latitude` | `NUMERIC(9,6)` | Geographic latitude of the listing. | Supports map-based visualization and precise geographic analysis beyond neighbourhood-level grouping. |
| `longitude` | `NUMERIC(9,6)` | Geographic longitude of the listing. | Same purpose as `latitude`. |

*Assumption: `first_review_date` and `last_review_date` correspond to the source fields `first_review` and `last_review`, renamed for clarity in the reporting layer. `estimated_revenue_l365d` reflects Airbnb's own trailing-365-day estimate where present, supplemented by a calculated value (price × estimated occupancy) for listings missing this figure directly — see the imputation methodology in [`Source_Schema.md`](./Source_Schema.md).*

---

## BRIDGE_AMENITIES

**Description:** A bridge (associative) table resolving the many-to-many relationship between listings and amenities. Each row represents one listing offering one amenity; a listing with 20 amenities produces 20 rows here. This structure is what allows amenity-level aggregation (e.g. "median revenue for listings with a Garage") without duplicating rows in `FACT_LISTINGS` itself.

| Column Name | Data Type | Description | Business Purpose |
|---|---|---|---|
| `listing_id` | `BIGINT` | Foreign key to `FACT_LISTINGS`. | Identifies which listing offers the associated amenity. |
| `amenity_id` | `BIGINT` | Foreign key to `DIM_AMENITIES`. | Identifies which standardized amenity is being associated with the listing. |

*Together, `listing_id` and `amenity_id` form this table's composite primary key.*
