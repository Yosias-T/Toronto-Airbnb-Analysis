# Toronto Airbnb Analytics — Project Roadmap

## Project Overview

**Business question:** What factors are associated with higher earnings for Airbnb hosts in Toronto?

**Target variables:** Estimated Revenue (primary), Estimated Occupancy (secondary)

**Grouping factors analyzed:** Property Group, Neighbourhood, Stay Length Category, Listing Size

**Tools used:** pgAdmin (PostgreSQL), Excel, Power BI, PowerPoint

**Dataset source:** [Inside Airbnb](http://insideairbnb.com/) — Toronto listings

---

## Project Development History

This project was completed in two phases, roughly a year apart.

It began in **2025** as an exploratory analytics project: raw Airbnb listings data was cleaned, transformed, and relationally modeled in PostgreSQL, then analyzed in Excel, culminating in an Excel dashboard.

In **2026**, after completing the Microsoft Power BI Data Analyst (PL-300) certification, the project was revisited to apply newly developed Power BI, DAX, and data-modeling skills to the same underlying question. This second phase expanded the original SQL work into a dedicated reporting schema and a Power BI star-schema data model.

The Power BI work is a **continuation and expansion** of the original project — not a separate effort. SQL, Excel, and Power BI were not used simultaneously; the phases below reflect the actual chronology.

---

## Phase 1 — Initial Analysis (2025)

### 1. Data Acquisition & Preparation (Excel)

- Removed columns not needed for analysis: `listing_url`, `source`, `scrape_id`, `last_scraped`, `picture_url`, `host_url`, `host_thumbnail_url`, `host_picture_url`, `neighborhood` (mostly empty; `neighbourhood_cleansed` used instead), `neighborhood_group_cleansed` (empty), `calendar_last_scraped`, `calendar_updated` (empty), `license`.
- Standardized `host_verifications` and `amenities` columns from Python-style list notation (`['abc', 'def']`) to PostgreSQL array notation (`{abc, def}`) via find-and-replace, enabling both a proper `ARRAY` column type and later analysis of semi-structured data.
- Cleaned "N/A" text values to true empty cells for reliable null handling in SQL.
- Reformatted percentage and currency cells to plain numeric values, corrected scientific-notation values, and standardized formatting across columns.
- Added a surrogate `listing_id` column, since the original listing IDs were inconsistent — generated via an Excel fill sequence, then reflected in a recreated raw table before import.
- Created the PostgreSQL database and schema (`Airbnb` database, `toronto` schema) and imported the prepared data into `raw_listings`.

### 2. Data Cleaning (SQL)

**Missing values**
- `bathrooms`: extracted numeric values from `bathrooms_text` using a regex match where `bathrooms` was null.
- Half-bath text values (`'Private half-bath'`, `'Shared half-bath'`, `'Half-bath'`) manually mapped to `0.5`.
- `has_availability` nulls set to `false`.
- 11 remaining rows with both `bathrooms` and `bathrooms_text` null were removed.

**Inactive listings**
- 11,203 rows were missing `estimated_revenue_l365d`; of those, 9,895 also had no `estimated_occupancy_l365d`.
- These 9,895 rows were flagged `inactive = true` via a new boolean column, since they don't reflect current market activity and are missing the analysis's target variable.

**Imputed price**
- 1,308 active rows still had no price. Missing prices were imputed using the median price for the listing's `room_type` + `neighbourhood_cleansed` combination.
- One listing had a unique `room_type`/`neighbourhood` combination with no comparison group; its price was imputed from the median for its `room_type` alone.
- An `imputed_price` boolean column tracks which rows were estimated.
- For rows still missing `estimated_revenue_l365d`, revenue was derived as `price × estimated_occupancy_l365d` — an approach validated first by confirming the equation held true on rows where both values already existed.

### 3. Feature Engineering

**Stay Length Category** — based on the distribution of `minimum_nights`:
| Category | Minimum nights |
|---|---|
| Short-term | < 28 |
| Long-term | 28–89 |
| Seasonal | 90–179 |
| Lease-style | 180+ |

**Listing Size** — based on `accommodates`:
| Category | Guests |
|---|---|
| Small | 1–2 |
| Medium | 3–4 |
| Large | 5–7 |
| Group | 8+ |

**Property Group** — standardized the many raw `property_type` values into consistent categories (House, Apartment, Hotel, Villa, Castle, Farm, Other), e.g. grouping "townhouse," "bungalow," and "guesthouse" under House, and "rental unit," "condo," and "suite" under Apartment. Additional passes resolved remaining edge cases later in the project (see *Notes & Flags* below).

**Rental Scope** — derived from `property_type` text into Entire / Private room / Shared room.

**Amenities** — `amenities` was already stored as a PostgreSQL array; unnested into a normalized table with one row per listing–amenity pair, then standardized to reduce wording redundancy (e.g. merging variants of the same amenity) that would otherwise skew amenity-level analysis. This cleaning continued in more depth during the Phase 2 reporting-schema work.

**Price outliers** — flagged using the IQR method: Q1 = 79, Q3 = 221, IQR = 142, giving an upper fence of 221 + 1.5 × 142 = **434**. Listings priced above this were flagged via `is_price_outlier`.

**False duplicates** — identified listings sharing name, latitude, and longitude but differing in price and description (not true duplicates). Resolved during normalization by retaining only the highest-priced row per (name, latitude, longitude) group.

**Neighbourhood Group** — Toronto's ~140 raw neighbourhoods were too granular for meaningful analysis and were grouped into broader geographic clusters (Etobicoke, Scarborough, North York, Downtown Core, Midtown, East End, West End). *(See Notes & Flags — this grouping was revised once after the initial pass.)*

### 4. Relational Modeling (Normalization)

Once the data was clean, `active_listings` (deduplicated, `inactive = false`) was normalized into a set of logical child tables, each in a 1:1 relationship with `listings` except `hosts` and `neighbourhoods` (1:many) and `amenities` (1:many via listing ID):

- **`neighbourhoods`** — `neighbourhood_id` (PK), neighbourhood name
- **`listings`** — `listing_id` (PK), `host_id` (FK), `neighbourhood_id` (FK), Property Group, Listing Size, Stay Length Category, Estimated Occupancy, Estimated Revenue
- **`price_info`** — price, imputed-price flag, price-outlier flag
- **`listing_details`** — description, neighbourhood overview, accommodates, bathrooms, bedrooms, beds, availability/booking flags
- **`hosts`** — deduplicated via `DISTINCT ON (host_id)`, ordered by `host_since DESC`
- **`reviews`** — review counts and review-category scores
- **`booking_length`** — minimum/maximum night constraints
- **`availability`** — 30/60/90/365-day and end-of-year availability (`has_availability` was dropped here as redundant — only 11 rows were `false`)
- **`amenities`** — recreated to reference `active_listings` rather than the raw table, since inactive listings weren't needed downstream

Primary and foreign key constraints were added throughout. `raw_listings` and `active_listings` were later renamed to `raw_toronto_listings` and `active_toronto_listings` and moved to the `public` schema.

### 5. SQL → Excel Analysis

1. **Confirmed analysis-ready fields**: target variables (Estimated Revenue, Estimated Occupancy) and grouping factors (Property Group, Neighbourhood, Stay Length Category, Listing Size) were verified clean and categorized.
2. **Built grouped summary tables in SQL** (average, median revenue and occupancy) for each factor and exported them to Excel.
3. **Built Excel visuals**: bar charts (average revenue by factor), box plots (revenue distribution and outliers), a neighbourhood heat map, and scatterplots (occupancy vs. revenue).
4. **Cross-factor analysis in SQL**: combinations such as Rental Scope × Neighbourhood Group, Rental Scope × Listing Size, Rental Scope × Property Group, and Rental Scope × Listing Size × Stay Length Category.
5. **Structured findings** around Business Question → Method → Findings → Insights/Recommendations, translating summary statistics into narrative observations about which listing characteristics were associated with higher revenue and occupancy.

**Output:** an Excel dashboard — the original analytical deliverable of the project.

---

## Phase 2 — Power BI Analysis (2026)

### 1. Project Revisit

After completing the Microsoft Power BI Data Analyst (PL-300) certification, the project was revisited to apply newly developed Power BI, DAX, data-modeling, and visualization skills to the existing dataset and business question.

### 2. Reporting-Schema Development

- Denormalized the Phase 1 relational structure back into a single wide reporting table (`listings_reporting`), reuniting fields that share a genuine 1:1 relationship with each listing.
- Reattached latitude/longitude, neighbourhood ID, and classification fields.
- Built a `host_info` summary table capturing each host's listing counts by room type.
- Created a dedicated `reporting` schema to house the Power BI-facing model.

### 3. Power BI Data Model (Star Schema)

- **`fact_listings`** — one row per listing (PK `listing_id`)
- **`dim_hosts`** — deduplicated host attributes (a duplicate host record was found and resolved by rebuilding the table with `DISTINCT`)
- **`dim_neighbourhoods`**
- **`dim_listing_classification`** — unique combinations of Property Group, Listing Size, Stay Length Category, and Rental Scope, stored once (208 rows) instead of repeating on every fact row (~11,177 rows). Property Type was added to this dimension at a later point in the project, requiring the table to be recreated.
- **`dim_amenities`** + **`bridge_amenities`** — a dedicated amenity dimension and a many-to-many bridge table linking listings to amenities, with further amenity-text cleaning applied at this stage to make the dimension analysis-ready
- A **surrogate key** (`host_key`) was introduced for hosts, replacing the original `host_id` as the model's primary/foreign key
- Primary keys, foreign keys, and indexes were added throughout the model

**Final structure:** 1 fact table, 3 dimension tables, and 1 bridge table — a standard star schema.

### 4. Power BI Model Extensions

A few fields were added directly in the Power BI model (calculated columns), on top of the reporting schema built in SQL:

- **`Segment`** (`dim_classifications`) — a concatenated label combining Property Category, Stay Length Category, and Listing Size (e.g. *"Apartment | Short-term | Group"*), used to drive the segment-level analysis on the Segment Matrix page.
- **`Revenue bins`** and **`Revenue Range`** (`fact_listings`) — Estimated Revenue grouped into $10K brackets, with a second column built on top of the first to generate a readable label (e.g. *"10K–20K"*) for the revenue-distribution histogram on the Executive Overview page.
- **`Amenity Count`** (`fact_listings`) — a count of amenities per listing via the bridge table, used for the `Average Amenity Count` measure.

Full DAX for the model's measures and calculated columns is documented in **[`DAX_MEASURES.md`](./DAX_MEASURES.md)**.

**Terminology note:** a few fields carry different names at different stages of the project. For reference:

| Phase 1 (SQL) | Phase 2 reporting schema | Power BI display name |
|---|---|---|
| `property_group` | `property_category` | **Property Category** |
| `accommodate_group` | `accommodation_size` | **Listing Size** |
| `listing_category` / `length_category` | `stay_length_category` | **Stay Length Category** |
| `neighbourhood_group` | — | **Neighbourhood Area** |
| `neighbourhood_cleansed` / `neighbourhood_name` | — | **Neighbourhood** |
| `rental_scope` | `rental_scope` | **Rental Scope** |

### 5. Report Design

The report comprises five analytical pages plus an About & Definitions page, navigated via a left sidebar.

**Visual style:** a compact, professional palette — dark teal (`#087E8B`) as the primary analytical color, coral (`#FF5A5F`) reserved for highlights and key findings, dark gray for text, light gray for the canvas, with mauve as an occasional third categorical color. Header banner in a darker accent color, white content cards on a light neutral background, and compact KPI cards (large number, small label) rather than verbose card text.

**Pages:**

| Page | Purpose | Key visuals |
|---|---|---|
| **Executive Overview** | How is the Toronto market performing overall? | KPI strip (Total Listings, Total Revenue, Average Revenue, Median Revenue, Average Occupancy %); revenue distribution histogram (by $10K bracket); listings & median revenue by Property Category; listings & average revenue by Listing Size; revenue share vs. listing share by Property Category |
| **Revenue Drivers** | Which characteristics are associated with higher revenue? | Revenue & occupancy by Stay Length Category; revenue & occupancy by Listing Size; median revenue by Property Category; top 10 neighbourhoods by median revenue |
| **Listing Performance** | Which listings and characteristics stand out? | Revenue-vs-occupied-nights scatter plot (by Property Category); top 15 listings by revenue table |
| **Segment Matrix** | Which listing segments perform best? | Listing segment performance matrix (Property Category × totals, medians, averages); top 5 segments by median revenue; a written Key Findings panel |
| **Amenities** | Are certain amenities associated with listing performance? | Most common amenities; amenities associated with higher revenue; amenities associated with higher occupancy (all gated at a 2% listing-prevalence minimum); amenity prevalence vs. performance scatter |
| **About & Definitions** | Methodology and terminology reference | Business question, data source, data collection period (July 2024 – June 2025), scrape date (June 14, 2025), scope, term definitions, and an explicit association-vs-causation disclaimer |

An earlier version of the report plan explored a Decomposition Tree and Key Influencers visual for deeper drill-down on the Listing Performance page; the final build uses the scatter plot and top-listings table instead.

### 6. Segmentation & Key Findings

The `Segment` field (Property Category × Stay Length Category × Listing Size) powers a dedicated performance matrix, surfacing patterns not visible from any single classification alone. Documented findings from the Segment Matrix page:

1. **Property mix:** Apartments account for 54% of listings, followed by houses at 42%.
2. **Revenue concentration:** Apartments generate 66% of total estimated revenue, despite representing 54% of listings.
3. **Listing size:** all five top-performing segments (by median revenue) consist of Large or Group listings — 5+ guests.
4. **Stay length:** short-term listings appear in 3 of the top 5 segments, though long-term segments also rank highly.

These are associations observed in the dataset, not causal claims — consistent with the disclaimer on the About & Definitions page.

---

## Notes & Flags for Review

*(Documented per project instructions — observations below are noted rather than silently corrected, since the goal is to preserve the project's actual history and logic.)*

- **Neighbourhood Group logic was revised once.** An initial `CASE` grouping was later replaced by a final, more complete version; the roadmap and reporting reflect the final version as authoritative.
- **Property Group classification was built in three passes** — an initial rule set, a small patch for edge cases (e.g. "casa particular," "hostel"), and a later pass to resolve any remaining "Other" rows. The logic in that final pass (mapping `'Private room'` → Apartment and `'Entire place'` → House) doesn't clearly follow the reasoning used in the earlier, more granular passes and may be worth revisiting.
- **Amenity text cleaning happened in two rounds** — once shortly after the amenities table was first created, and again, more extensively, during the Phase 2 reporting-schema build. Both are legitimate parts of the project's history.
- **An early, apparently superseded `listings` table definition** appears at the very start of the SQL file, before `raw_listings` is created. It looks like an initial draft schema that predates the final approach and was not carried forward.

