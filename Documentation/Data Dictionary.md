# Data Dictionary — Toronto Airbnb Market Analysis

This document defines every table and field in the Power BI reporting model, which follows a star-schema design: one fact table (`fact\_listings`) surrounded by four dimension tables (`dim\_hosts`, `dim\_neighbourhoods`, `dim\_classifications`, `dim\_amenities`) and one bridge table (`bridge\_amenities`) resolving the many-to-many relationship between listings and amenities. A disconnected `Measures List` table holds the DAX measures.

Each field is documented with:

* **Column Name** — the name as it appears in Power BI (and in the report visuals).
* **Source Column** — the column name in the PostgreSQL `reporting` schema (see [SQL Code.md](./SQL%20Code.md) and [Reporting Schema.md](./Reporting%20Schema.md)). Renaming happens in Power Query.
* **Data Type** — PostgreSQL type → Power BI type. Calculated columns show the Power BI type only.
* **Business Purpose** — how the field is actually used in analysis and reporting, not just what it technically stores.

> \*\*Note on naming:\*\* several fields carry different names at different stages of the project (e.g. `property\_group` in the original SQL model → `property\_category` in the reporting schema → \*\*Property Category\*\* in Power BI). Where relevant, the prior name is noted for cross-reference with \[SQL Code.md](./SQL%20Code.md).

> \*\*Note on hidden fields:\*\* surrogate keys and foreign keys (`host\_key`, `neighbourhood\_id`, `amenity\_id`, etc.) are hidden in Report view. They keep their snake\_case names because they are technical join fields, not analysis fields. The entire `bridge\_amenities` table is hidden.

**Power BI type key:** Whole Number = `int64` · Decimal Number = `double` · Fixed Decimal (Currency) = `decimal` · Text = `string` · True/False = `boolean` · Date = `dateTime`

\---

## dim\_hosts

**Description:** One row per unique Airbnb host. Supports host-level analysis — for example, comparing hosts who manage a single listing against multi-listing hosts or Superhosts, and understanding how a host's portfolio composition (entire homes vs. private/shared rooms) relates to their overall scale in the market.

### Source columns

|Column Name|Source Column|Data Type|Description|Business Purpose|
|-|-|-|-|-|
|`host\_key` *(hidden)*|`host\_key`|`BIGINT` → Whole Number|Surrogate primary key generated for the reporting model, replacing the original Airbnb `host\_id`.|Enables stable relationships between `fact\_listings` and this dimension, independent of any changes to source-system host identifiers.|
|**Host Name**|`host\_name`|`VARCHAR(50)` → Text|The host's display name as listed on Airbnb.|Used for listing-level detail views and host identification in drill-through analysis.|
|**Host Since**|`host\_since`|`DATE` → Date|The date the host's Airbnb account was created.|Enables tenure-based segmentation — e.g. comparing performance of long-tenured hosts against newer entrants to the Toronto market.|
|**Host Location**|`host\_location`|`VARCHAR(75)` → Text|The host's self-reported location.|Indicates whether a host is locally based in Toronto or managing listings remotely; relevant to understanding local vs. absentee/professional management patterns.|
|**Superhost**|`host\_is\_superhost`|`BOOLEAN` → True/False|Whether the host holds Airbnb's "Superhost" status.|A trust/quality signal; used to evaluate whether Superhost status is associated with stronger listing performance (revenue, occupancy, reviews).|

### 

### 

### Calculated columns (DAX)

|Column Name|Data Type|Logic|Business Purpose|
|-|-|-|-|
|**Active Listings Count**|Whole Number|`COUNTROWS(RELATEDTABLE(fact\_listings))` — the number of listings this host has **in this dataset**.|Unlike `Total Listings Count` (Airbnb's own snapshot figure), this is derived from the model itself, so it always reconciles with the listing-level data. It drives `Host Scale`.|
|**Host Scale**|Text|Buckets `Active Listings Count` into **1 Listing**, **2-3 Listings**, **4-10 Listings**, or **10+ Listings** (anything above 10); **Unknown** if blank. Sorted by `Host Scale Sort`.|Turns a continuous listing count into readable host-size tiers for comparing single-listing hosts against professional operators on revenue and occupancy.|
|**Host Scale Sort**|Whole Number|Numeric sort key for `Host Scale` (0 = Unknown, 1 = 1 Listing, 2 = 2-3, 3 = 4-10, 4 = 10+).|Ensures `Host Scale` displays in logical order rather than alphabetically. Helper column — not used directly in visuals.|
|**Superhost Status**|Text|Converts `Superhost` (True/False) to **Superhost**, **Not Superhost**, or **Unknown**.|Provides readable legend/axis labels for charts and slicers instead of raw TRUE/FALSE.|

\---

## dim\_neighbourhoods

**Description:** One row per Toronto neighbourhood, with both a granular neighbourhood name and a broader geographic grouping. Supports location-based analysis at two levels of detail — from citywide area comparisons down to individual neighbourhood rankings. Both text columns are tagged with the Power BI *Place* data category for map visuals.

|Column Name|Source Column|Data Type|Description|Business Purpose|
|-|-|-|-|-|
|`neighbourhood\_id` *(hidden)*|`neighbourhood\_id`|`BIGINT` → Whole Number|Surrogate primary key for the neighbourhood dimension.|Joins `fact\_listings` to this dimension for all location-based analysis and filtering.|
|**Neighbourhood**|`neighbourhood\_name`|`VARCHAR(50)` → Text|The specific Toronto neighbourhood name, as defined by the City of Toronto's official neighbourhood boundaries (via the source data's `neighbourhood\_cleansed` field).|Powers granular, neighbourhood-level analysis — e.g. the Top 10 Neighbourhoods by Median Revenue view, used to identify specific high-value micro-markets.|
|**Neighbourhood Area**|`neighbourhood\_group`|`VARCHAR(50)` → Text|A broader geographic cluster (e.g. Downtown Core, Midtown, North York, Scarborough, Etobicoke, East End, West End) that the neighbourhood belongs to.|Reduces \~140 individual neighbourhoods to a manageable set of analytical regions, making area-level comparisons and slicers usable without overwhelming the report with too many categories.|

**Hierarchy:** *Neighbourhood Area Hierarchy* — Neighbourhood Area → Neighbourhood.

*Assumption: the mapping of individual neighbourhoods to broader groups reflects Toronto's commonly recognized district boundaries (e.g. former pre-amalgamation municipalities), applied manually during data preparation rather than sourced from an official geographic dataset.*

\---

## dim\_classifications

**Description:** One row per unique combination of property and rental characteristics (208 combinations in the current dataset). This is a consolidated "junk dimension" — rather than maintaining four or five separate low-cardinality dimension tables, the classification attributes that are frequently filtered together are combined into a single dimension, which is both a common dimensional-modeling technique and a query-performance optimization.

### Source columns

|Column Name|Source Column|Data Type|Description|Business Purpose|
|-|-|-|-|-|
|`classification\_key` *(hidden)*|`classification\_key`|`BIGINT` → Whole Number|Surrogate primary key for this dimension.|Joins `fact\_listings` to a single set of classification attributes, avoiding repeated category text on every fact row.|
|**Rental Scope**|`rental\_scope`|`VARCHAR(20)` → Text|Whether the listing offers exclusive use of the property or shared/partial access: **Entire**, **Private Room**, or **Shared Room**.|A primary segmentation axis throughout the report; rental scope is one of the strongest differentiators of both price point and guest experience.|
|**Property Category**|`property\_category`|`VARCHAR(20)` → Text|A standardized property classification (Apartment, House, Hotel, Villa, Castle, Farm, Other), derived from dozens of raw Airbnb property-type values. Referred to as `property\_group` in the source/normalized schema.|Enables meaningful property-type comparisons that wouldn't be possible against the raw, highly fragmented `Property Type` field — the basis for the report's Property Category-level revenue and occupancy analysis.|
|**Listing Size**|`accommodation\_size`|`VARCHAR(20)` → Text|A guest-capacity band — **Small** (1–2), **Medium** (3–4), **Large** (5–7), or **Group** (8+) — derived from the listing's `Accommodates` value. Referred to as `accommodate\_group` in the source schema.|Groups listings into comparable size tiers for revenue/occupancy analysis, since raw guest-capacity values are too granular to analyze individually.|
|**Stay Length Category**|`stay\_length\_category`|`VARCHAR(20)` → Text|A minimum-stay classification — **Short-term** (<28 nights), **Long-term** (28–89), **Seasonal** (90–179), or **Lease-style** (180+) — derived from the listing's minimum night requirement.|Distinguishes classic short-term rentals from longer-term stays, which behave differently in both pricing and regulatory context (Toronto's short-term rental bylaws influence minimum-stay behavior).|
|**Property Type**|`property\_type`|`VARCHAR(45)` → Text|The original, unstandardized Airbnb property type text (e.g. "Entire rental unit," "Private room in bungalow").|Preserved alongside the standardized `Property Category` for transparency and to support drill-down into the most granular level of property detail when needed.|

**Hierarchy:** *Property Category Hierarchy* — Property Category → Property Type.

### Calculated columns (DAX)

|Column Name|Data Type|Logic|Business Purpose|
|-|-|-|-|
|**Segment**|Text|Concatenates `Property Category`, `Stay Length Category`, and `Listing Size` with " \| " separators (e.g. `Apartment \| Short-term \| Small`).|A single combined label for ranking and comparing the most granular market segments in one visual. Used by the `Segment, % of Listings` measure.|

\---

## dim\_amenities

**Description:** One row per unique, standardized amenity offered across all listings. Works together with `bridge\_amenities` to resolve the many-to-many relationship between listings and amenities — a single listing can offer dozens of amenities, and a single amenity (e.g. Wi-Fi) can appear on thousands of listings.

|Column Name|Source Column|Data Type|Description|Business Purpose|
|-|-|-|-|-|
|`amenity\_id` *(hidden)*|`amenity\_id`|`BIGINT` → Whole Number|Surrogate primary key for the amenity dimension.|Joins `bridge\_amenities` to a single, deduplicated amenity name.|
|**Amenity**|`amenity`|`VARCHAR(100)` → Text|The standardized amenity name (e.g. "Wifi," "Air Conditioning," "Lake View"), consolidated from dozens of raw wording variants in the source data.|Powers all amenity-level analysis in the report — most common amenities, amenities associated with higher revenue/occupancy, and amenity prevalence-vs-performance comparisons. Standardization is what makes this analysis possible at all; without it, wording variants would fragment the same amenity into dozens of near-duplicate categories.|

\---

## fact\_listings

**Description:** The central fact table, with one row per active Toronto Airbnb listing (\~11,177 rows). Each row represents a single listing's attributes, pricing, availability, review performance, and estimated financial performance over the trailing 365 days. This is the grain the entire report is built on — every measure in the Power BI model ultimately aggregates from this table.

### Source columns

|Column Name|Source Column|Data Type|Description|Business Purpose|
|-|-|-|-|-|
|`listing_id` *(hidden)*|`listing_id`|`BIGINT` → Whole Number|Surrogate primary key uniquely identifying each listing.|The fact table's grain; the join key for every measure and every dimension relationship in the model. Counted by `Total Listings`.|
|`host_key` *(hidden)*|`host_key`|`BIGINT` → Whole Number|Foreign key to `dim_hosts`.|Links each listing to its host, enabling host-level rollups and Superhost/tenure-based analysis.|
|`neighbourhood_id` *(hidden)*|`neighbourhood_id`|`BIGINT` → Whole Number|Foreign key to `dim\_neighbourhoods`.|Links each listing to its location for all geographic analysis.|
|`classification_key`<br />*(hidden)*|`classification_key`|`BIGINT` → Whole Number|Foreign key to `dim_classifications`.|Links each listing to its property category, listing size, stay length category, rental scope, and raw property type.|
|**Listing Name**|`listing_name`|`TEXT` → Text|The listing's title as displayed on Airbnb.|Used in listing-level detail views (e.g. the Top 15 Listings by Revenue table) for human-readable identification.|
|**Accommodates**|`accommodates`|`SMALLINT` → Whole Number|Maximum guest capacity of the listing.|The raw value underlying the `Listing Size` classification; also useful for precise, non-binned capacity analysis.|
|**Bathrooms**|`bathrooms`|`NUMERIC(3,1)` → Decimal Number|Number of bathrooms (supports half-bath values, e.g. 1.5).|A core property-size attribute used in listing-level detail and quality assessment.|
|**Bedrooms**|`bedrooms`|`SMALLINT` → Whole Number|Number of bedrooms.|Same purpose as `Bathrooms` — a standard property-size attribute for detail views.|
|**Beds**|`beds`|`SMALLINT` → Whole Number|Number of beds.|Same purpose as above; distinct from bedrooms since a single bedroom can contain multiple beds.|
|**Price**|`price`|`NUMERIC(7,2)` → Fixed Decimal (Currency)|Nightly listing price in CAD, as set by the host (with a small number of missing values imputed — see `Source\_Schema.md`).|The base pricing metric that, combined with occupancy, determines estimated revenue; also the basis for `Average Price`.|
|**Is Price Outlier**|`is_price_outlier`|`BOOLEAN` → True/False|Flags listings priced above the IQR-based upper fence ($434/night).|Allows analysis to include or exclude unusually high-priced listings, preventing a small number of luxury outliers from distorting average price or revenue figures.|
|**Minimum Nights**|`minimum_nights`|`INT` → Whole Number|The minimum number of nights a guest must book.|The raw value underlying `Stay Length Category`; also directly relevant to understanding regulatory-driven booking constraints.|
|**Maximum Nights**|`maximum_nights`|`INT` → Whole Number|The maximum number of nights a guest may book.|Complements `Minimum Nights` in describing a listing's booking-length policy.|
|**Total Reviews**|`number_of_reviews`|`INT` → Whole Number|Total number of reviews received over the listing's lifetime.|A proxy for a listing's overall booking history and guest engagement over time.|
|**Total Reviews (Last 12 Months)**|`number_of_reviews_ltm`|`SMALLINT` → Whole Number|Number of reviews received in the last twelve months.|A more current activity signal than lifetime review count, useful for assessing recent guest engagement.|
|**Total Reviews (Last Year)**|`number_of_reviews_ly`|`SMALLINT` → Whole Number|Number of reviews received in the prior calendar year.|Supports year-over-year review-volume comparison.|
|**Estimated Occupancy (Last 365 Days)**|`estimated_occupancy_l365d`|`SMALLINT` → Whole Number | Modelled estimate of nights booked over the trailing 365 days. Per Inside Airbnb's published methodology, it is derived from review counts and an assumed length of stay (not observed bookings) and capped at 70% of the year, i.e. 255 nights. 2,174 listings (19.5%) sit exactly at that cap. | One of the two primary target variables in this project. Because of the cap and the estimation method, occupancy comparisons should be read as approximate rather than exact. |
|**Estimated Revenue (Last 365 Days)**|`estimated_revenue_l365d`|`INT` → Fixed Decimal (Currency)|Estimated revenue over the trailing 365 days (derived from price × estimated occupancy where not directly available).|The primary target variable in this project — the central metric the entire analysis is built to explain. Basis for `Revenue bins`.|
|**First Review**|`first_review`|`DATE` → Date|Date of the listing's first review.|Serves as a practical proxy for how long a listing has been actively hosting guests.|
|**Last Review**|`last_review`|`DATE` → Date|Date of the listing's most recent review.|A recency signal indicating whether a listing is still actively receiving bookings.|
|**Rating**|`overall_rating`|`NUMERIC(3,2)` → Decimal Number|The listing's overall guest rating.|The headline quality metric surfaced alongside revenue rankings (e.g. in the Top 15 Listings by Revenue table), adding a quality lens to performance analysis. Basis for `Rating Band`.|
|**Accuracy Score**|`accuracy_score`|`NUMERIC(3,2)` → Decimal Number|Sub-rating for how accurately the listing was described.|One of Airbnb's six standard review sub-categories; supports more granular quality analysis than the overall rating alone.|
|**Cleanliness Score**|`cleanliness_score`|`NUMERIC(3,2)` → Decimal Number|Sub-rating for listing cleanliness.|Same purpose as above; cleanliness is typically one of the more revenue-sensitive sub-ratings.|
|**Check In Score**|`check_in_score`|`NUMERIC(3,2)` → Decimal Number|Sub-rating for the check-in experience.|Same purpose as above.|
|**Communication Score**|`communication_score`|`NUMERIC(3,2)` → Decimal Number|Sub-rating for host communication quality.|Same purpose as above; also reflects host responsiveness and service quality.|
|**Location Score**|`location_score`|`NUMERIC(3,2)` → Decimal Number|Sub-rating for the listing's location.|Same purpose as above; can be cross-referenced against neighbourhood-level revenue patterns.|
|**Value Score**|`value_score`|`NUMERIC(3,2)` → Decimal Number|Sub-rating for perceived value for money.|Same purpose as above; particularly relevant when analyzed alongside price and revenue.|
|**Reviews Per Month**|`reviews_per_month`|`NUMERIC(5,2)` → Decimal Number|Average number of reviews received per month since the listing's first review.|A normalized booking-velocity metric that accounts for how long a listing has been active, unlike raw review counts.|
|**Instantly Bookable**|`instant_bookable`|`BOOLEAN` → True/False|Whether the listing can be booked immediately without host approval.|A booking-friction indicator; relevant to understanding whether ease of booking correlates with occupancy performance.|
|**Latitude**|`latitude`|`NUMERIC(9,6)` → Decimal Number|Geographic latitude of the listing.|Supports map-based visualization and precise geographic analysis beyond neighbourhood-level grouping.|
|**Longitude**|`longitude`|`NUMERIC(9,6)` → Decimal Number|Geographic longitude of the listing.|Same purpose as `Latitude`.|

*Assumption: `Estimated Revenue (Last 365 Days)` reflects Airbnb's own trailing-365-day estimate where present, supplemented by a calculated value (price × estimated occupancy) for listings missing this figure directly — see the imputation methodology in* [*SQL Code.md*](./SQL%20Code.md) *(Section 2.3).*

*Not loaded into Power BI: the source column `room\_type` (Airbnb's native room-type classification: Entire home/apt, Private room, Shared room, Hotel room) is removed in Power Query. It remains available in the PostgreSQL `reporting.fact\_listings` table; the derived `Rental Scope` in `dim\_classifications` is used in the report instead.*

### Calculated columns (DAX)

|Column Name|Data Type|Logic|Business Purpose|
|-|-|-|-|
|**Amenity Count**|Whole Number|`COUNTROWS(RELATEDTABLE(bridge\_amenities))` — number of amenities offered by the listing.|Enables amenity-richness analysis at the listing level. Feeds `Average Amenity Count` and `Amenity Density Score`.|
|**Revenue bins**|Fixed Decimal (Currency)|Rounds `Estimated Revenue (Last 365 Days)` down to the nearest $10,000 (blank if revenue is blank). Set up as a Power BI numeric bin group.|Groups listings into $10K revenue bands for distribution charts. Numeric so the bands sort correctly.|
|**Revenue Range**|Text|Text label for each bin, e.g. `0–10K`, `10K–20K`, `20K–30K`. Sorted by `Revenue bins`.|Readable axis labels for the revenue distribution chart; used with `% of Total Listings`.|
|**Rating Band**|Text|Buckets `Rating` into **Below 4.0**, **4.0–4.5**, **4.5–4.8**, **4.8–5.0**, or **No Rating** (blank). Sorted by `Rating Band Sort`.|Groups continuous ratings into meaningful quality tiers for comparing revenue/occupancy across guest-satisfaction levels.|
|**Rating Band Sort**|Whole Number|Numeric sort key for `Rating Band` (0 = No Rating, 1 = Below 4.0, 2 = 4.0–4.5, 3 = 4.5–4.8, 4 = 4.8–5.0).|Ensures `Rating Band` displays in logical order. Helper column — not used directly in visuals.|

\---

## bridge\_amenities

**Description:** A hidden bridge (associative) table resolving the many-to-many relationship between listings and amenities. Each row represents one listing offering one amenity; a listing with 20 amenities produces 20 rows here. This structure is what allows amenity-level aggregation (e.g. "median revenue for listings with a Garage") without duplicating rows in `fact\_listings` itself. The relationship to `fact\_listings` uses **bi-directional** filtering so amenity selections filter listings.

|Column Name|Source Column|Data Type|Description|Business Purpose|
|-|-|-|-|-|
|`listing\_id` *(hidden)*|`listing\_id`|`BIGINT` → Whole Number|Foreign key to `fact\_listings`.|Identifies which listing offers the associated amenity.|
|`amenity\_id` *(hidden)*|`amenity\_id`|`BIGINT` → Whole Number|Foreign key to `dim\_amenities`.|Identifies which standardized amenity is being associated with the listing.|

*In PostgreSQL, `listing\_id` and `amenity\_id` together form this table's composite primary key. The Power BI model does not define a key on this table.*

\---

## Measures List

**Description:** A disconnected, empty calculated table (a single blank `Measure Name` column) that exists only to hold the model's DAX measures, organized into display folders. Abbreviation prefixes used in measure names: **LS** = Listing Size, **PC** = Property Category, **SL** = Stay Length Category, **NA** = Neighbourhood Area.

### Revenue

|Measure|Logic|Business Purpose|
|-|-|-|
|**Total Revenue**|Sum of `Estimated Revenue (Last 365 Days)`.|Headline market-size metric. Base for all revenue-share measures.|
|**Average Revenue**|Average of `Estimated Revenue (Last 365 Days)`.|Mean revenue per listing; sensitive to high earners.|
|**Median Revenue**|Median of `Estimated Revenue (Last 365 Days)`.|The primary "typical listing" revenue metric; robust to outliers.|
|**Median Revenue (min 50 listing count)**|`Median Revenue`, blank if the current filter context has fewer than 50 listings.|Prevents small-sample segments from appearing in rankings (e.g. Top 10 Neighbourhoods by Median Revenue).|
|**LS, % of Total Revenue**|`Total Revenue` ÷ total with `Listing Size` filter removed.|Share of revenue by listing size.|
|**PC, % of Total Revenue**|Same, removing the `Property Category` filter.|Share of revenue by property category.|
|**SL, % of Total Revenue**|Same, removing the `Stay Length Category` filter.|Share of revenue by stay length category.|
|**NA, % of Total Revenue**|Same, removing the `Neighbourhood Area` filter.|Share of revenue by neighbourhood area.|
|**Neighbourhood, Revenue % of Area**|`Total Revenue` ÷ total with the `Neighbourhood` filter removed.|Each neighbourhood's share of revenue within its selected area (or citywide if no area is selected).|

### Occupancy

|Measure|Logic|Business Purpose|
|-|-|-|
|**Total Occupied Nights**|Sum of `Estimated Occupancy (Last 365 Days)`.|Total booked nights; base for occupancy-share measures.|
|**Average Occupied Nights**|Average of `Estimated Occupancy (Last 365 Days)`.|Mean booked nights per listing.|
|**Median Occupied Nights**|Median of `Estimated Occupancy (Last 365 Days)`.|Typical booked nights per listing; robust to outliers.|
|**Average Occupancy %**|`Average Occupied Nights` ÷ 365.|Converts average nights to an occupancy rate.|
|**LS, % of Occupied Nights**|`Total Occupied Nights` ÷ total with `Listing Size` filter removed.|Share of booked nights by listing size.|
|**PC, % of Occupied Nights**|Same, removing the `Property Category` filter.|Share of booked nights by property category.|
|**SL, % Occupied Nights**|Same, removing the `Stay Length Category` filter.|Share of booked nights by stay length category.|
|**NA, % of Occupied Nights**|Same, removing the `Neighbourhood Area` filter.|Share of booked nights by neighbourhood area.|
|**Neighbourhood, Occupied Nights % of Area**|Same, removing the `Neighbourhood` filter.|Each neighbourhood's share of booked nights within its area.|

### Listings

|Measure|Logic|Business Purpose|
|-|-|-|
|**Total Listings**|Count of `listing\_id`.|Core supply metric; base for listing-share measures and sample-size thresholds.|
|**LS, % of Listings**|`Total Listings` ÷ total with `Listing Size` filter removed.|Listing-count mix by listing size.|
|**PC, % of Listings**|Same, removing the `Property Category` filter.|Listing-count mix by property category.|
|**SL, % of Listings**|Same, removing the `Stay Length Category` filter.|Listing-count mix by stay length category.|
|**NA, % of Listings**|Same, removing the `Neighbourhood Area` filter.|Listing-count mix by neighbourhood area.|
|**Neighbourhood, Listings % of Area**|Same, removing the `Neighbourhood` filter.|Each neighbourhood's share of listings within its area.|
|**Segment, % of Listings**|Same, removing the `Segment` filter.|Share of listings by combined Property Category / Stay Length / Listing Size segment.|
|**% of Total Listings**|`Total Listings` ÷ total with `Revenue bins` and `Revenue Range` filters removed.|Share of listings in each revenue band (revenue distribution chart).|

### Amenities

|Measure|Logic|Business Purpose|
|-|-|-|
|**Average Amenity Count**|Average of `Amenity Count`.|Typical amenity richness of a listing.|
|**Amenity Density Score**|`Average Amenity Count` ÷ the same measure with the `Neighbourhood` filter removed.|Indexes a neighbourhood's amenity richness relative to its wider area (1.0 = same as the area).|
|**Amenity, Revenue %**|`Total Revenue` ÷ total with the `Amenity` filter removed.|Share of total revenue earned by listings offering the selected amenity.|
|**Amenity, Occupancy %**|`Total Occupied Nights` ÷ total with the `Amenity` filter removed.|Share of total booked nights captured by listings offering the selected amenity.|
|**Amenity, % of Listings**|`Total Listings` ÷ total with the `Amenity` filter removed.|Amenity prevalence — share of listings offering it.|
|**Amenity Minimum Listings**|2% of total listings (ignoring the amenity filter).|Threshold for suppressing rarely-offered amenities in performance comparisons.|
|**Amenity Median Revenue**|`Median Revenue`, blank if the amenity appears on fewer listings than `Amenity Minimum Listings`.|Amenity-vs-revenue comparison without noise from rare amenities.|
|**Amenity Median Occupied Nights**|`Median Occupied Nights`, blank if below `Amenity Minimum Listings`.|Amenity-vs-occupancy comparison without noise from rare amenities.|

### Price Metrics

|Measure|Logic|Business Purpose|
|-|-|-|
|**Average Price**|Average of `Price`.|Average nightly rate; can be filtered with `Is Price Outlier` to exclude luxury outliers.|



