# Toronto Airbnb Analytics — SQL Code

This document contains the SQL used throughout the project, organized by workflow stage rather than strict chronological order, so the logic reads as a coherent pipeline. It spans both project phases:

- **Phase 1 (2025):** raw data preparation, cleaning, feature engineering, relational modeling, and SQL-driven Excel analysis.
- **Phase 2 (2026):** the reporting-schema and Power BI star-schema build.

SQL logic is unchanged from the original project. Formatting (keyword case, indentation, section headers, light commenting) has been standardized for readability. Anything that looked questionable, redundant, or worth a second look is flagged inline as `-- NOTE:` rather than silently changed — see also the *Notes & Flags* section in the project roadmap.

**Environment:** PostgreSQL, database `airbnb`.

---

## 1. Source / Raw Data Preparation

Initial table definition and import setup. `raw_listings` was created with a surrogate `listing_id` primary key (the original Airbnb `id` values were inconsistent), and a handful of column types were corrected after the first import pass.

```sql
CREATE DATABASE airbnb;

CREATE SCHEMA toronto;

ALTER DATABASE airbnb SET search_path TO toronto;

CREATE TABLE raw_listings (
    listing_id                                     BIGINT PRIMARY KEY,
    id                                              BIGINT,
    name                                            TEXT,
    description                                     TEXT,
    neighborhood_overview                           TEXT,
    host_id                                         INT,
    host_name                                       VARCHAR(50),
    host_since                                      DATE,
    host_location                                   VARCHAR(75),
    host_about                                      TEXT,
    host_response_time                              VARCHAR(35),
    host_response_rate                              NUMERIC(5, 2),
    host_acceptance_rate                            NUMERIC(5, 2),
    host_is_superhost                               BOOLEAN,
    host_neighborhood                               VARCHAR(50),
    host_listing_count                              SMALLINT,
    host_total_listing_count                        SMALLINT,
    host_verifications                              TEXT[],
    host_has_profile_pic                            BOOLEAN,
    host_identity_verified                          BOOLEAN,
    neighbourhood_cleansed                          VARCHAR(50),
    latitude                                        NUMERIC(9, 6),
    longitude                                       NUMERIC(9, 6),
    property_type                                   VARCHAR(45),
    room_type                                       VARCHAR(20),
    accommodates                                    SMALLINT,
    bathrooms                                       NUMERIC(3, 1),
    bathrooms_text                                  VARCHAR(50),
    bedrooms                                        SMALLINT,
    beds                                             SMALLINT,
    amenities                                       TEXT[],
    price                                            NUMERIC(7, 2),
    minimum_nights                                  INT,
    maximum_nights                                  INT,
    minimum_minimum_nights                          INT,
    maximum_minimum_nights                          INT,
    minimum_maximum_nights                          INT,
    maximum_maximum_nights                          INT,
    minimum_nights_avg_ntm                          DECIMAL(8, 2),
    maximum_nights_avg_ntm                          DECIMAL(12, 2),
    has_availability                                BOOLEAN,
    availability_30                                 SMALLINT,
    availability_60                                 SMALLINT,
    availability_90                                 SMALLINT,
    availability_365                                SMALLINT,
    number_of_reviews                               INT,
    number_of_reviews_ltm                           SMALLINT,
    number_of_reviews_l30d                          SMALLINT,
    availability_eoy                                SMALLINT,
    number_of_reviews_ly                            SMALLINT,
    estimated_occupancy_l365d                       SMALLINT,
    estimated_revenue_l365d                         INT,
    first_review                                    DATE,
    last_review                                     DATE,
    review_scores_rating                            NUMERIC(3, 2),
    review_scores_accuracy                          NUMERIC(3, 2),
    review_scores_cleanliness                       NUMERIC(3, 2),
    review_scores_checkin                           NUMERIC(3, 2),
    review_scores_communication                     NUMERIC(3, 2),
    review_scores_location                          NUMERIC(3, 2),
    review_scores_value                             NUMERIC(3, 2),
    instant_bookable                                BOOLEAN,
    calculated_host_listings_count                  SMALLINT,
    calculated_host_listings_count_entire_homes     SMALLINT,
    calculated_host_listings_count_private_rooms    SMALLINT,
    calculated_host_listings_count_shared_rooms     SMALLINT,
    reviews_per_month                               NUMERIC(5, 2)
);

-- NOTE: an earlier draft table definition (named `listings`, without `listing_id`)
-- preceded this one in the original file and appears to have been superseded before
-- raw_listings was finalized. It is not carried forward here — see roadmap Notes & Flags.

-- Data type corrections made after the initial import:
ALTER TABLE raw_listings
    ALTER COLUMN minimum_nights_avg_ntm TYPE DECIMAL(8, 2),
    ALTER COLUMN maximum_nights_avg_ntm TYPE DECIMAL(12, 2);

ALTER TABLE raw_listings
    ALTER COLUMN host_location TYPE VARCHAR(75);

ALTER TABLE raw_listings
    ALTER COLUMN id TYPE BIGINT;

-- Repointing the primary key to the new surrogate listing_id column:
ALTER TABLE raw_listings DROP CONSTRAINT raw_listings_pkey;
ALTER TABLE raw_listings ADD COLUMN listing_id SERIAL PRIMARY KEY;
```

---

## 2. Data Cleaning

### 2.1 Availability and bathrooms

```sql
UPDATE raw_listings
SET has_availability = false
WHERE has_availability IS NULL;

UPDATE raw_listings
SET bathrooms = 0.5
WHERE bathrooms_text IN ('Private half-bath', 'Shared half-bath', 'Half-bath');

-- 11 rows with no bathrooms and no bathrooms_text remained and were removed:
DELETE FROM raw_listings
WHERE bathrooms IS NULL;
```

### 2.2 Price and inactive listings

```sql
-- Scoping the problem: rows missing price, revenue, or both
SELECT COUNT(*) FROM raw_listings
WHERE price IS NULL;

SELECT *
FROM raw_listings
WHERE price IS NULL
  AND estimated_occupancy_l365d > 0
  AND number_of_reviews > 0
ORDER BY minimum_nights;

SELECT price, estimated_revenue_l365d, estimated_occupancy_l365d
FROM raw_listings
WHERE price IS NULL
  AND estimated_revenue_l365d IS NULL
  AND estimated_occupancy_l365d > 0;

-- Checking whether any listings are active (has occupancy) but missing revenue:
SELECT *
FROM raw_listings
WHERE price IS NULL
  AND estimated_occupancy_l365d > 0
  AND estimated_revenue_l365d IS NULL
  AND number_of_reviews > 0;

ALTER TABLE raw_listings
    ADD COLUMN inactive BOOLEAN;

-- Identifying obviously inactive listings:
SELECT *
FROM raw_listings
WHERE (last_review IS NULL OR last_review < CURRENT_DATE - INTERVAL '12 months')
  AND availability_365 = 0
  AND price IS NULL
  AND estimated_occupancy_l365d = 0
  AND estimated_revenue_l365d IS NULL
  AND has_availability = false
  AND calculated_host_listings_count = 0;

SELECT *
FROM raw_listings
WHERE availability_365 = 0
  AND (estimated_occupancy_l365d > 0 OR estimated_revenue_l365d IS NOT NULL);

SELECT COUNT(*)
FROM raw_listings
WHERE (estimated_revenue_l365d = 0 OR estimated_revenue_l365d IS NULL)
  AND estimated_occupancy_l365d = 0;

-- 9,895 rows had no revenue and no occupancy in the trailing 365 days — flagged inactive,
-- since they don't reflect current market conditions and lack the analysis's target variable.
UPDATE raw_listings
SET inactive = false;

UPDATE raw_listings
SET inactive = true
WHERE (estimated_revenue_l365d = 0 OR estimated_revenue_l365d IS NULL)
  AND estimated_occupancy_l365d = 0;
```

### 2.3 Imputed price

```sql
ALTER TABLE raw_listings
    ADD COLUMN imputed_price BOOLEAN;

UPDATE raw_listings
SET imputed_price = false;

UPDATE raw_listings
SET imputed_price = true
WHERE price IS NULL
  AND inactive = false;

-- 1,308 active rows remained with no price. Imputed using the median price for the
-- listing's room_type + neighbourhood combination:
UPDATE raw_listings rl
SET price = sub.median_price
FROM (
    SELECT room_type, neighbourhood_cleansed,
           PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY price) AS median_price
    FROM raw_listings
    WHERE price IS NOT NULL
    GROUP BY 1, 2
) sub
WHERE rl.price IS NULL
  AND rl.inactive = false
  AND rl.room_type = sub.room_type
  AND rl.neighbourhood_cleansed = sub.neighbourhood_cleansed;

-- 1 listing had a unique room_type + neighbourhood combination with no match;
-- imputed using the median price for its room_type alone:
UPDATE raw_listings rl
SET price = sub.median_price
FROM (
    SELECT room_type,
           PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY price) AS median_price
    FROM raw_listings
    WHERE price IS NOT NULL
    GROUP BY room_type
) sub
WHERE rl.price IS NULL
  AND rl.inactive = false
  AND rl.room_type = sub.room_type;

-- Revenue derived for active rows still missing it, using price and occupancy
-- (both now available, since price has just been imputed above). This relationship
-- was validated against existing non-null price/revenue/occupancy rows before being
-- applied here; the validation queries themselves aren't part of this documented script.
UPDATE raw_listings
SET estimated_revenue_l365d = price * estimated_occupancy_l365d
WHERE estimated_revenue_l365d IS NULL
  AND inactive = false;

SELECT DISTINCT room_type FROM raw_listings
WHERE inactive = false;
```

---

## 3. Feature Engineering

### 3.1 Stay Length Category (`listing_category`)

```sql
SELECT MAX(minimum_nights) FROM raw_listings
WHERE inactive = false;

-- Distribution of minimum_nights values, to inform the bin boundaries:
SELECT minimum_nights, COUNT(*)
FROM raw_listings
WHERE inactive = false
GROUP BY 1
ORDER BY 1 DESC;

ALTER TABLE raw_listings
    ADD COLUMN listing_category VARCHAR(20);

UPDATE raw_listings
SET listing_category = CASE
    WHEN minimum_nights < 28               THEN 'Short-term'
    WHEN minimum_nights BETWEEN 28 AND 89  THEN 'Long-term'
    WHEN minimum_nights BETWEEN 90 AND 179 THEN 'Seasonal'
    WHEN minimum_nights >= 180              THEN 'Lease-style'
END;
```

### 3.2 Listing Size (`accommodate_group`)

```sql
SELECT DISTINCT accommodates FROM raw_listings;

SELECT accommodates, COUNT(*)
FROM raw_listings
WHERE inactive = false
GROUP BY 1
ORDER BY 1 DESC;

ALTER TABLE raw_listings
    ADD COLUMN accommodate_group VARCHAR(20);

UPDATE raw_listings
SET accommodate_group = CASE
    WHEN accommodates <= 2               THEN 'Small'
    WHEN accommodates BETWEEN 3 AND 4    THEN 'Medium'
    WHEN accommodates BETWEEN 5 AND 7    THEN 'Large'
    WHEN accommodates >= 8                THEN 'Group'
END;
```

### 3.3 Property Group (`property_group`)

```sql
SELECT property_type, COUNT(*)
FROM raw_listings
GROUP BY 1
ORDER BY 1 DESC;

ALTER TABLE raw_listings
    ADD COLUMN property_group TEXT;

SELECT *
FROM raw_listings
WHERE property_type ILIKE 'private room' OR property_type ILIKE '%home/apt%';

UPDATE raw_listings
SET property_group = CASE
    WHEN property_type ILIKE '%house%' OR property_type ILIKE '%home%'
      OR property_type ILIKE '%townhouse%' OR property_type ILIKE '%bungalow%'
      OR property_type ILIKE '%guesthouse%' OR property_type ILIKE '%floor%'
        THEN 'House'

    WHEN property_type ILIKE '%rental unit%' OR property_type ILIKE '%loft%'
      OR property_type ILIKE '%condo%' OR property_type ILIKE '%apartment%'
      OR property_type ILIKE '%suite%'
        THEN 'Apartment'

    WHEN property_type ILIKE '%bed and breakfast%' OR property_type ILIKE '%aparthotel%'
        THEN 'Hotel'

    WHEN property_type ILIKE '%villa%' THEN 'Villa'

    WHEN property_type ILIKE '%castle%' THEN 'Castle'

    WHEN property_type ILIKE '%cottage%' OR property_type ILIKE '%barn%'
      OR property_type ILIKE '%farm%'
        THEN 'Farm'

    ELSE 'Other'
END;
```

> Two further refinement passes were applied to `property_group` after normalization (Section 5.2) and during the Phase 2 reporting-schema build (Section 7.1).

### 3.4 Amenities (initial pass)

```sql
CREATE TABLE amenities AS
SELECT
    listing_id,
    CASE
        WHEN amenity ILIKE '%wi-fi%'                                        THEN 'wifi'
        WHEN amenity ILIKE '%fridge%' OR amenity ILIKE '%refrigerator%'     THEN 'fridge'
        WHEN amenity ILIKE '%tv%'                                          THEN 'tv'
        WHEN amenity ILIKE '%air conditioning%' OR amenity ILIKE '%a/c%'
          OR amenity ILIKE '%ac %'                                          THEN 'air conditioning'
        WHEN amenity ILIKE '%washer%'                                      THEN 'washer'
        WHEN amenity ILIKE '%dryer%'                                       THEN 'dryer'
        WHEN amenity ILIKE '%heating%' OR amenity ILIKE '%heated%'         THEN 'heating'
        WHEN amenity ILIKE '%grill%'                                       THEN 'grill'
        WHEN amenity ILIKE '%stove%'                                       THEN 'stove'
        WHEN amenity ILIKE '%paid%' AND amenity ILIKE '%parking%'          THEN 'paid parking'
        WHEN amenity ILIKE '%free%' AND amenity ILIKE '%parking%'          THEN 'free parking'
        WHEN amenity ILIKE '%coffee maker%' OR amenity ILIKE '%coffe machine%' THEN 'coffee maker'
        WHEN amenity ILIKE '%oven%'                                        THEN 'oven'
        WHEN amenity ILIKE '%crib%'                                        THEN 'crib'
        WHEN amenity ILIKE '%fireplace%'                                   THEN 'fireplace'
        WHEN amenity ILIKE '%pool%'                                        THEN 'pool'
        WHEN amenity ILIKE '%gym%'                                         THEN 'gym'
        WHEN amenity ILIKE '%shampoo%'                                     THEN 'shampoo'
        WHEN amenity ILIKE '%backyard%'                                    THEN 'backyard'
        WHEN amenity ILIKE '%hot tub%'                                     THEN 'hot tub'
        ELSE amenity
    END AS amenity
FROM (
    SELECT listing_id, TRIM(LOWER(unnest(amenities))) AS amenity
    FROM raw_listings
) AS sub;

CREATE INDEX idx_amenity_listing_id
    ON amenities (amenity, listing_id);

ALTER TABLE amenities
    ADD CONSTRAINT fk_listing
    FOREIGN KEY (listing_id)
    REFERENCES raw_listings(listing_id)
    ON DELETE CASCADE;
```

> Amenity cleaning continues, more extensively, in Section 6.1 as part of the Phase 2 reporting-schema build.

### 3.5 Price outliers

```sql
SELECT
    MIN(beds),
    MAX(beds),
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY beds) AS q1,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY beds) AS q3
FROM raw_listings
WHERE beds IS NOT NULL;

SELECT COUNT(*) FROM raw_listings
WHERE price > 434;

-- IQR method: Q1 = 79, Q3 = 221, IQR = 142
-- Upper fence = 221 + 1.5 x 142 = 434
ALTER TABLE raw_listings
    ADD COLUMN is_price_outlier BOOLEAN;

UPDATE raw_listings
SET is_price_outlier = price > 434;
```

### 3.6 False duplicates

```sql
-- Rows sharing name/latitude/longitude but differing in price and description
-- (not true duplicates — resolved during normalization, see Section 4.2):
SELECT name, latitude, longitude, COUNT(*)
FROM raw_listings
GROUP BY 1, 2, 3
HAVING COUNT(*) > 1;

SELECT
    name, latitude, longitude,
    COUNT(*) AS count,
    COUNT(DISTINCT name) AS distinct_names,
    COUNT(DISTINCT price) AS price_variety,
    COUNT(DISTINCT description) AS description_variety
FROM raw_listings
GROUP BY name, latitude, longitude
HAVING COUNT(*) > 1;
```

> Neighbourhood grouping (binning ~140 raw neighbourhoods into broader geographic clusters) is covered in Section 5.1, since it's applied to the `neighbourhoods` table created during normalization.

---

## 4. Normalization / Relational Modeling

Once `raw_listings` was clean, the data was split into logical child tables. Design principle: `listings` has a 1:1 relationship with every child table except `hosts` and `neighbourhoods` (1:many, referenced by FK) and `amenities` (1:many via `listing_id`).

### 4.1 Neighbourhoods

```sql
CREATE TABLE neighbourhoods AS
SELECT DISTINCT neighbourhood_cleansed AS neighbourhood
FROM raw_listings
WHERE neighbourhood_cleansed IS NOT NULL;

ALTER TABLE neighbourhoods
    ADD COLUMN neighbourhood_id SERIAL PRIMARY KEY;

ALTER TABLE neighbourhoods
    ADD COLUMN neighbourhood_name VARCHAR(50);

UPDATE neighbourhoods
SET neighbourhood_name = neighbourhood_cleansed;

ALTER TABLE neighbourhoods
    DROP COLUMN neighbourhood_cleansed;
```

### 4.2 Active listings (deduplication + inactive filter)

```sql
CREATE TABLE active_listings AS
SELECT DISTINCT ON (name, latitude, longitude) *
FROM raw_listings
WHERE inactive = false
ORDER BY name, latitude, longitude, price DESC;

ALTER TABLE active_listings
    ADD CONSTRAINT pk_listing PRIMARY KEY (listing_id);
```

`DISTINCT ON (name, latitude, longitude)` combined with `ORDER BY ... price DESC` resolves the false-duplicate rows identified in Section 3.6 by keeping only the highest-priced row per group.

### 4.3 Amenities (recreated against active listings)

```sql
-- The amenities table from Section 3.4 was dropped and recreated to reference
-- active_listings, since inactive listings weren't needed downstream:
DROP TABLE amenities;

-- (amenities table recreated here — see Section 3.4 for the unnest/standardization logic)

ALTER TABLE amenities
    ADD CONSTRAINT fk_listing
    FOREIGN KEY (listing_id)
    REFERENCES active_listings(listing_id)
    ON DELETE CASCADE;

CREATE INDEX idx_amenity_listing_id
    ON amenities (amenity, listing_id);

-- Foreign key repointed once more, to the final `listings` table (Section 4.4):
ALTER TABLE amenities DROP CONSTRAINT fk_listing;

ALTER TABLE amenities
    ADD CONSTRAINT fk_listing
    FOREIGN KEY (listing_id)
    REFERENCES listings(listing_id)
    ON DELETE CASCADE;
```

### 4.4 Listings (core table)

```sql
CREATE TABLE listings AS
SELECT
    listing_id, name, host_id, neighbourhood_id, property_group,
    accommodate_group, listing_category AS length_category,
    estimated_occupancy_l365d AS occupancy_l365d,
    estimated_revenue_l365d AS revenue_l365d
FROM active_listings a
JOIN neighbourhoods n ON a.neighbourhood_cleansed = n.neighbourhood_cleansed;

ALTER TABLE listings
    ADD CONSTRAINT pk_listings PRIMARY KEY (listing_id);

ALTER TABLE listings
    ADD CONSTRAINT fk_neighbourhood
    FOREIGN KEY (neighbourhood_id)
    REFERENCES neighbourhoods(neighbourhood_id);
```

### 4.5 Price info

```sql
CREATE TABLE price_info AS
SELECT listing_id, price, imputed_price, is_price_outlier
FROM active_listings;

ALTER TABLE price_info
    ADD CONSTRAINT pk_price_info PRIMARY KEY (listing_id);

ALTER TABLE price_info
    ADD CONSTRAINT fk_price_info
    FOREIGN KEY (listing_id)
    REFERENCES listings(listing_id)
    ON DELETE CASCADE;
```

### 4.6 Listing details

```sql
CREATE TABLE listing_details AS
SELECT
    listing_id, description, neighborhood_overview AS neighbourhood_overview,
    accommodates, bathrooms, bedrooms, beds, has_availability, instant_bookable
FROM active_listings;

ALTER TABLE listing_details
    ADD CONSTRAINT pk_listing_details PRIMARY KEY (listing_id);

ALTER TABLE listing_details
    ADD CONSTRAINT fk_listing_details
    FOREIGN KEY (listing_id)
    REFERENCES listings(listing_id)
    ON DELETE CASCADE;
```

### 4.7 Hosts

```sql
CREATE TABLE hosts AS
SELECT DISTINCT ON (host_id)
    host_id, host_name, host_since, host_location, host_about,
    host_response_time, host_acceptance_rate, host_is_superhost,
    host_neighborhood, host_listing_count, host_total_listing_count,
    host_verifications, host_has_profile_pic, host_identity_verified
FROM active_listings
ORDER BY host_id, host_since DESC;

ALTER TABLE hosts
    ADD CONSTRAINT pk_hosts PRIMARY KEY (host_id);

ALTER TABLE listings
    ADD CONSTRAINT fk_listings_host
    FOREIGN KEY (host_id)
    REFERENCES hosts(host_id)
    ON DELETE CASCADE;
```

### 4.8 Reviews

```sql
CREATE TABLE reviews AS
SELECT
    listing_id, number_of_reviews, reviews_per_month, number_of_reviews_l30d,
    number_of_reviews_ltm AS number_of_reviews_l12m, number_of_reviews_ly,
    first_review, last_review, review_scores_rating, review_scores_accuracy,
    review_scores_cleanliness, review_scores_checkin, review_scores_communication,
    review_scores_location, review_scores_value
FROM active_listings;

ALTER TABLE reviews
    ADD CONSTRAINT pk_listing_reviews PRIMARY KEY (listing_id);

ALTER TABLE reviews
    ADD CONSTRAINT fk_listing_reviews
    FOREIGN KEY (listing_id)
    REFERENCES listings(listing_id)
    ON DELETE CASCADE;
```

### 4.9 Booking length

```sql
CREATE TABLE booking_length AS
SELECT
    listing_id, minimum_nights, maximum_nights,
    minimum_minimum_nights, maximum_minimum_nights,
    minimum_maximum_nights, maximum_maximum_nights,
    minimum_nights_avg_ntm AS minimum_nights_avg_n12m,
    maximum_nights_avg_ntm AS maximum_nights_avg_n12m
FROM active_listings;

ALTER TABLE booking_length
    ADD CONSTRAINT pk_booking_length PRIMARY KEY (listing_id);

ALTER TABLE booking_length
    ADD CONSTRAINT fk_booking_length_listings
    FOREIGN KEY (listing_id) REFERENCES listings(listing_id)
    ON DELETE CASCADE;
```

### 4.10 Availability

```sql
CREATE TABLE availability AS
SELECT
    listing_id, availability_30, availability_60,
    availability_90, availability_365,
    availability_eoy AS availability_until_eoy
FROM active_listings;
-- has_availability was not carried over here: only 11 rows were false, making it redundant.

ALTER TABLE availability
    ADD CONSTRAINT pk_availability PRIMARY KEY (listing_id);

ALTER TABLE availability
    ADD CONSTRAINT fk_availability_listings
    FOREIGN KEY (listing_id)
    REFERENCES listings(listing_id)
    ON DELETE CASCADE;
```

### 4.11 Renaming and schema cleanup

```sql
ALTER TABLE raw_listings
    RENAME TO raw_toronto_listings;

ALTER TABLE active_listings
    RENAME TO active_toronto_listings;

ALTER TABLE raw_toronto_listings
    SET SCHEMA public;

ALTER TABLE active_toronto_listings
    SET SCHEMA public;

CREATE INDEX idx_listings_property_group    ON listings(property_group);
CREATE INDEX idx_listings_accommodate_group ON listings(accommodate_group);
CREATE INDEX idx_listings_length_category   ON listings(length_category);
```

**Resulting schema** (Phase 1 relational model):

```
neighbourhoods ──< listings >── hosts
                     │
     ┌───────┬───────┼───────┬─────────────┐
     │       │       │       │             │
price_info listing_details reviews booking_length availability
                                                        │
                                                    amenities (1:many via listing_id)
```

---

## 5. Post-Normalization Refinements

A few classification fields were refined further after the core tables existed, since they depend on `listings`, `listing_details`, or `neighbourhoods` being in place.

### 5.1 Neighbourhood Group

Toronto's raw neighbourhood list (~140 values) was too granular for meaningful analysis and was grouped into broader geographic clusters. This was done in two passes — an initial version, followed by a final, more complete revision. **The final version (below) is authoritative**; the first pass is included for completeness since it was part of the actual project history.

```sql
ALTER TABLE neighbourhoods
    ADD COLUMN neighbourhood_group VARCHAR(50);

-- Initial pass
UPDATE neighbourhoods
SET neighbourhood_group = CASE
    -- Etobicoke
    WHEN neighbourhood_name IN (
        'Alderwood', 'Long Branch', 'Mimico-Queensway', 'New Toronto',
        'Rexdale-Kipling', 'Thistletown-Beaumond Heights', 'Islington-City Centre West',
        'Kingsway South', 'Humber Heights-Westmount', 'Humber Summit'
    ) THEN 'Etobicoke'

    -- Scarborough
    WHEN neighbourhood_name IN (
        'Agincourt South-Malvern West', 'Agincourt North', 'Guildwood', 'Woburn',
        'Malvern', 'Milliken', 'Dorset Park', 'Kennedy Park', 'LAmoreaux',
        'Clairlea-Birchmount', 'Highland Creek', 'Centennial Scarborough',
        'Cliffside', 'Birch Cliff', 'Eglinton East', 'Ionview', 'Tam OShanter-Sullivan',
        'Scarborough Village West', 'Rouge'
    ) THEN 'Scarborough'

    -- North York
    WHEN neighbourhood_name IN (
        'Banbury-Don Mills', 'Bayview Village', 'Bedford Park-Nortown', 'Black Creek',
        'Don Valley Village', 'Englemount-Lawrence', 'Glenfield-Jane Heights', 'Glen Park',
        'Lawrence Park', 'Lawrence Manor', 'Leaside-Bennington', 'Leaside', 'Little Portugal',
        'Mount Dennis', 'Oakwood-Vaughan', 'Oakridge', 'OConnor-Parkview',
        'Parkwoods-Donalda', 'Pleasant View', 'Rockcliffe-Smythe', 'Rosedale-Moore Park',
        'Steeles', 'Victoria Village', 'Yorkdale-Glen Park', 'York Mills', 'York University Heights'
    ) THEN 'North York'

    -- Downtown Core
    WHEN neighbourhood_name IN (
        'Bay Street Corridor', 'Bloor-Yorkville', 'Church-Yonge Corridor',
        'Danforth Village-East York', 'Danforth Village-Leslieville', 'Don Mills',
        'Dufferin Grove', 'East End-Danforth', 'East York', 'Forest Hill South',
        'Forest Hill North', 'Niagara', 'Annex', 'Moss Park', 'Trinity-Bellwoods',
        'University', 'Kensington-Chinatown', 'Cabbagetown-South St. James Town',
        'Regent Park', 'High Park-North', 'High Park-Swansea', 'South Riverdale',
        'St. Andrew-Windfields', 'St. James Town', 'St. Lawrence', 'Swansea',
        'Palmerston-Little Italy', 'Playter Estates-Danforth', 'South Parkdale'
    ) THEN 'Downtown Core'

    -- Midtown
    WHEN neighbourhood_name IN (
        'Mount Pleasant East', 'Mount Pleasant West', 'Forest Hill South',
        'Forest Hill North', 'Yonge-Eglinton', 'Leaside-Bennington', 'Leaside'
    ) THEN 'Midtown'

    -- York & East York
    WHEN neighbourhood_name IN (
        'East York', 'Danforth Village-East York', 'Danforth Village-Leslieville',
        'Rockcliffe-Smythe'
    ) THEN 'York & East York'

    ELSE 'Other'
END;

CREATE INDEX idx_neighbourhood_group
    ON neighbourhoods(neighbourhood_group);

-- Final revision
UPDATE neighbourhoods
SET neighbourhood_group = CASE
    -- Downtown Core
    WHEN neighbourhood_name IN (
        'Blake-Jones', 'Broadview North', 'Cabbagetown-South St.James Town',
        'Casa Loma', 'Danforth', 'Dovercourt-Wallace Emerson-Junction',
        'Junction Area', 'North Riverdale', 'North St.James Town',
        'Roncesvalles', 'Runnymede-Bloor West Village',
        'Waterfront Communities-The Island', 'Wychwood', 'Yonge-St.Clair',
        'Bay Street Corridor', 'Bloor-Yorkville', 'Church-Yonge Corridor',
        'Dufferin Grove', 'Niagara', 'Annex', 'Moss Park',
        'Trinity-Bellwoods', 'University', 'Kensington-Chinatown',
        'Regent Park', 'High Park-Swansea', 'South Riverdale',
        'St. James Town', 'St. Lawrence', 'Swansea',
        'Palmerston-Little Italy', 'Playter Estates-Danforth',
        'South Parkdale', 'Little Portugal'
    ) THEN 'Downtown Core'

    -- Midtown
    WHEN neighbourhood_name IN (
        'Briar Hill-Belgravia', 'Corso Italia-Davenport', 'Hillcrest Village',
        'Humewood-Cedarvale', 'Keelesdale-Eglinton West',
        'Lawrence Park North', 'Lawrence Park South', 'Oakwood Village',
        'Mount Pleasant East', 'Mount Pleasant West', 'Yonge-Eglinton',
        'Forest Hill North', 'Forest Hill South', 'Rosedale-Moore Park',
        'Leaside-Bennington'
    ) THEN 'Midtown'

    -- North York
    WHEN neighbourhood_name IN (
        'Bathurst Manor', 'Bayview Woods-Steeles', 'Beechborough-Greenbrook',
        'Bridle Path-Sunnybrook-York Mills', 'Clanton Park',
        'Downsview-Roding-CFB', 'Henry Farm', 'Humbermede',
        'Lansing-Westgate', 'Maple Leaf', 'Newtonbrook East',
        'Newtonbrook West', 'Rustic', 'St.Andrew-Windfields',
        'Westminster-Branson', 'Willowdale East', 'Willowdale West',
        'Banbury-Don Mills', 'Bayview Village', 'Bedford Park-Nortown',
        'Black Creek', 'Don Valley Village', 'Englemount-Lawrence',
        'Glenfield-Jane Heights', 'Glen Park', 'Lawrence Manor',
        'Parkwoods-Donalda', 'Pleasant View', 'Steeles',
        'Victoria Village', 'York Mills', 'York University Heights',
        'Brookhaven-Amesbury', 'Yorkdale-Glen Park'
    ) THEN 'North York'

    -- Scarborough
    WHEN neighbourhood_name IN (
        'Bendale', 'Birchcliffe-Cliffside', 'Cliffcrest', 'Morningside',
        'Scarborough Village', 'West Hill', 'Wexford/Maryvale',
        'Agincourt South-Malvern West', 'Agincourt North', 'Guildwood',
        'Woburn', 'Malvern', 'Milliken', 'Dorset Park', 'Kennedy Park',
        'LAmoreaux', 'Clairlea-Birchmount', 'Highland Creek',
        'Centennial Scarborough', 'Eglinton East', 'Ionview',
        'Tam OShanter-Sullivan', 'Rouge', 'Oakridge'
    ) THEN 'Scarborough'

    -- Etobicoke
    WHEN neighbourhood_name IN (
        'Edenbridge-Humber Valley', 'Elms-Old Rexdale',
        'Eringate-Centennial-West Deane', 'Etobicoke West Mall',
        'Kingsview Village-The Westway', 'Markland Wood',
        'Mimico (includes Humber Bay Shores)', 'Mount Olive-Silverstone-Jamestown',
        'Pelmo Park-Humberlea', 'Princess-Rosethorn', 'Stonegate-Queensway',
        'West Humber-Clairville', 'Weston', 'Willowridge-Martingrove-Richview',
        'Alderwood', 'Long Branch', 'Mimico-Queensway', 'New Toronto',
        'Rexdale-Kipling', 'Thistletown-Beaumond Heights',
        'Islington-City Centre West', 'Kingsway South',
        'Humber Heights-Westmount', 'Humber Summit'
    ) THEN 'Etobicoke'

    -- East End
    WHEN neighbourhood_name IN (
        'Danforth East York', 'Flemingdon Park', 'Greenwood-Coxwell',
        'Old East York', 'Taylor-Massey', 'The Beaches',
        'Thorncliffe Park', 'Woodbine-Lumsden', 'Woodbine Corridor',
        'East End-Danforth', 'Danforth Village-Leslieville', 'East York',
        'OConnor-Parkview'
    ) THEN 'East End'

    -- West End
    WHEN neighbourhood_name IN (
        'Caledonia-Fairbank', 'Lambton Baby Point', 'Weston-Pellam Park',
        'Rockcliffe-Smythe', 'Mount Dennis', 'High Park North'
    ) THEN 'West End'

    ELSE 'Other'
END;
```

### 5.2 Property Group (further refinements)

Two more passes resolved edge cases the initial `property_group` logic (Section 3.3) didn't cover.

```sql
-- Pass 2: casa particular / hostel weren't captured by the initial rule set
UPDATE listings l
SET property_group = CASE
    WHEN property_type ILIKE '%casa particular%' THEN 'House'
    WHEN property_type ILIKE '%hostel%'          THEN 'Hotel'
    ELSE property_group
END
FROM listing_details ld
WHERE l.listing_id = ld.listing_id;

-- Pass 3: resolving any remaining 'Other' rows
-- NOTE: this mapping doesn't clearly follow the reasoning used in Sections 3.3 and 5.2
-- above (e.g. 'Private room' -> Apartment, 'Entire place' -> House) and is flagged here
-- for review rather than silently changed — see roadmap Notes & Flags.
UPDATE listings l
SET property_group = CASE
    WHEN property_type IN ('Room in hotel', 'Room in boutique hotel') THEN 'Hotel'
    WHEN property_type IN ('Entire place')                            THEN 'House'
    WHEN property_type IN ('Private room')                            THEN 'Apartment'
    ELSE 'Other'
END
FROM public.active_toronto_listings l1
WHERE l.listing_id = l1.listing_id
  AND l.property_group = 'Other';

-- property_type was also copied onto listing_details for reference:
ALTER TABLE listing_details
    ADD COLUMN property_type VARCHAR(45);

UPDATE listing_details ld
SET property_type = l1.property_type
FROM public.active_toronto_listings l1
WHERE ld.listing_id = l1.listing_id;
```

### 5.3 Rental Scope

```sql
SELECT DISTINCT property_type, l.rental_scope
FROM listing_details ld
JOIN listings l ON ld.listing_id = l.listing_id;

ALTER TABLE listings
    ADD COLUMN rental_scope VARCHAR(20);

UPDATE listings l
SET rental_scope = CASE
    WHEN ld.property_type ILIKE '%shared room%' THEN 'Shared room'
    WHEN ld.property_type ILIKE '%room%'         THEN 'Private room'
    WHEN ld.property_type ILIKE '%entire%'       THEN 'Entire'
    ELSE 'Other'
END
FROM listing_details ld
WHERE l.listing_id = ld.listing_id;

UPDATE listings
SET rental_scope = 'Entire'
WHERE rental_scope ILIKE 'other';
```

---

## 6. Analysis Views (Phase 1 — SQL → Excel)

Grouped summary views built in SQL and exported to Excel for visualization. All figures describe **association**, not causation, between listing characteristics and Estimated Revenue / Estimated Occupancy.

### 6.1 Property Group

```sql
CREATE VIEW property_group_revenues AS
SELECT
    property_group,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100, 2) AS share_of_properties,
    ROUND(AVG(revenue_l365d), 2) AS average_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS revenue_median
FROM listings l
GROUP BY property_group
ORDER BY num_of_listings DESC;

CREATE VIEW property_group_occupancies AS
SELECT
    property_group,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100, 2) AS share_of_properties,
    ROUND(AVG(occupancy_l365d), 2) AS average_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS occupancy_median
FROM listings l
GROUP BY property_group
ORDER BY num_of_listings DESC;

-- Combined summary (supersedes the two views above)
CREATE VIEW property_group_summary AS
SELECT
    property_group,
    COUNT(*) AS num_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100, 2) AS share_of_properties,

    -- Revenue stats
    ROUND(AVG(revenue_l365d), 2) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,

    -- Occupancy stats
    ROUND(AVG(occupancy_l365d), 2) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d

FROM listings l
GROUP BY property_group
ORDER BY num_listings DESC;
```

### 6.2 Neighbourhood Group

```sql
CREATE VIEW neighbourhood_group_summary AS
SELECT
    neighbourhood_group,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_properties,
    ROUND(AVG(revenue_l365d), 2) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 2) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM neighbourhoods n
JOIN listings l ON n.neighbourhood_id = l.neighbourhood_id
GROUP BY neighbourhood_group
ORDER BY num_of_listings DESC;
```

### 6.3 Stay Length Category

```sql
CREATE VIEW length_category_summary AS
SELECT
    length_category,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_listings,
    ROUND(AVG(revenue_l365d), 2) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 2) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM listings
GROUP BY length_category
ORDER BY num_of_listings DESC;
```

### 6.4 Listing Size

```sql
CREATE VIEW accommodate_group_summary AS
SELECT
    accommodate_group,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_listings,
    ROUND(AVG(revenue_l365d), 2) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 2) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM listings
GROUP BY accommodate_group
ORDER BY num_of_listings DESC;
```

### 6.5 Cross-Factor Analysis

```sql
CREATE VIEW rental_scope_neighbourhood_summary AS
SELECT
    rental_scope,
    neighbourhood_group,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_listings,
    ROUND(AVG(revenue_l365d), 2) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 2) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM listings l
JOIN neighbourhoods n ON l.neighbourhood_id = n.neighbourhood_id
GROUP BY rental_scope, neighbourhood_group
ORDER BY rental_scope, neighbourhood_group;

CREATE VIEW rental_scope_accommodate_group_summary AS
SELECT
    rental_scope,
    accommodate_group,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_listings,
    ROUND(AVG(revenue_l365d), 1) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 1) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM listings
GROUP BY rental_scope, accommodate_group
ORDER BY avg_revenue_l365d DESC;

CREATE VIEW rental_scope_length_category_summary AS
SELECT
    rental_scope,
    length_category,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_listings,
    ROUND(AVG(revenue_l365d), 1) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 1) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM listings
GROUP BY rental_scope, length_category
ORDER BY avg_revenue_l365d DESC;

CREATE VIEW rental_scope_property_group_summary AS
SELECT
    rental_scope,
    property_group,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_listings,
    ROUND(AVG(revenue_l365d), 1) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 1) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM listings
GROUP BY rental_scope, property_group
ORDER BY avg_revenue_l365d DESC;

CREATE VIEW rental_scope_accommodate_length_summary AS
SELECT
    rental_scope,
    accommodate_group,
    length_category,
    COUNT(*) AS num_of_listings,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM listings) * 100.0, 2) AS share_of_listings,
    ROUND(AVG(revenue_l365d), 1) AS avg_revenue_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue_l365d) AS median_revenue_l365d,
    ROUND(AVG(occupancy_l365d), 1) AS avg_occupancy_l365d,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY occupancy_l365d) AS median_occupancy_l365d
FROM listings
GROUP BY rental_scope, accommodate_group, length_category
ORDER BY avg_revenue_l365d DESC;
```

---

## 7. Reporting Schema (Phase 2 — Power BI, 2026)

After completing the PL-300 certification, the project was revisited and the normalized Phase 1 tables were rebuilt into a dedicated reporting schema, denormalized into a fact/dimension/bridge structure suited to a Power BI star schema.

### 7.1 Reporting table (denormalized)

```sql
CREATE TABLE listings_reporting AS
SELECT
    listing_id, name AS listing_name, host_id, property_type, room_type, accommodates,
    bathrooms, bedrooms, beds, price, minimum_nights, maximum_nights, number_of_reviews, number_of_reviews_ltm,
    number_of_reviews_ly, estimated_occupancy_l365d, estimated_revenue_l365d,
    first_review, last_review, review_scores_rating AS overall_rating,
    review_scores_accuracy AS accuracy_score, review_scores_cleanliness AS cleanliness_score,
    review_scores_checkin AS check_in_score, review_scores_communication AS communication_score,
    review_scores_location AS location_score, review_scores_value AS value_score,
    reviews_per_month, instant_bookable, is_price_outlier
FROM public.active_toronto_listings;

ALTER TABLE listings_reporting
    ADD COLUMN property_group VARCHAR(20),
    ADD COLUMN length_category VARCHAR(20),
    ADD COLUMN accommodation_group VARCHAR(20),
    ADD COLUMN rental_scope VARCHAR(20);

UPDATE listings_reporting lr
SET property_group = l.property_group,
    length_category = l.length_category,
    accommodation_group = l.accommodate_group,
    rental_scope = l.rental_scope
FROM listings l
WHERE lr.listing_id = l.listing_id;

CREATE TABLE lat_long_listings AS
SELECT listing_id, latitude, longitude FROM public.active_toronto_listings;

ALTER TABLE listings_reporting
    ADD COLUMN latitude  NUMERIC(9, 6),
    ADD COLUMN longitude NUMERIC(9, 6);

UPDATE listings_reporting lr
SET latitude = lll.latitude,
    longitude = lll.longitude
FROM lat_long_listings lll
WHERE lr.listing_id = lll.listing_id;

ALTER TABLE listings_reporting
    ADD COLUMN neighbourhood_id BIGINT;

UPDATE listings_reporting lr
SET neighbourhood_id = l.neighbourhood_id
FROM listings l
WHERE lr.listing_id = l.listing_id;

CREATE TABLE host_info AS
SELECT
    host_id,
    calculated_host_listings_count AS total_listings_count,
    calculated_host_listings_count_entire_homes AS entire_home_listings,
    calculated_host_listings_count_private_rooms AS private_room_listings,
    calculated_host_listings_count_shared_rooms AS shared_room_listings
FROM public.active_toronto_listings;
```

### 7.2 Reporting schema — fact and dimension tables

```sql
CREATE SCHEMA reporting;

CREATE TABLE reporting.fact_listings AS
SELECT * FROM listings_reporting;

ALTER TABLE reporting.fact_listings
    ADD CONSTRAINT pk_fact_listings
    PRIMARY KEY (listing_id);

CREATE TABLE reporting.dim_hosts AS
SELECT DISTINCT
    h.host_id,
    host_name,
    host_since,
    host_location,
    host_is_superhost
FROM hosts h
LEFT JOIN host_info hf ON h.host_id = hf.host_id;

ALTER TABLE reporting.dim_hosts
    ADD CONSTRAINT pk_dim_hosts
    PRIMARY KEY (host_id);
-- 1 duplicate host record was found; table was dropped and recreated using DISTINCT.
-- TODO (noted in original project): rebuild dim_hosts to pull host attributes directly
-- from active listings instead of via host_info, for a cleaner source.

CREATE TABLE reporting.dim_neighbourhoods AS
SELECT * FROM neighbourhoods;

ALTER TABLE reporting.dim_neighbourhoods
    ADD CONSTRAINT pk_dim_neighbourhoods
    PRIMARY KEY (neighbourhood_id);

ALTER TABLE reporting.fact_listings
    ADD CONSTRAINT fk_fact_host
    FOREIGN KEY (host_id)
    REFERENCES reporting.dim_hosts(host_id);

ALTER TABLE reporting.fact_listings
    ADD CONSTRAINT fk_fact_neighbourhoods
    FOREIGN KEY (neighbourhood_id)
    REFERENCES reporting.dim_neighbourhoods(neighbourhood_id);
```

### 7.3 Classification dimension

Stores unique combinations of Stay Length Category, Listing Size, Property Group, and Rental Scope once (208 rows) instead of repeating them on every fact row (~11,177 rows).

```sql
CREATE TABLE reporting.dim_listing_classification AS
SELECT DISTINCT
    length_category,
    accommodation_group,
    property_group,
    rental_scope
FROM reporting.fact_listings;

ALTER TABLE reporting.dim_listing_classification
    ADD COLUMN classification_id BIGINT
    GENERATED ALWAYS AS IDENTITY
    PRIMARY KEY;

ALTER TABLE reporting.fact_listings
    ADD COLUMN classification_id BIGINT;

UPDATE reporting.fact_listings f
SET classification_id = c.classification_id
FROM reporting.dim_listing_classification c
WHERE f.length_category = c.length_category
  AND f.accommodation_group = c.accommodation_group
  AND f.property_group = c.property_group
  AND f.rental_scope = c.rental_scope;

ALTER TABLE reporting.fact_listings
    ADD CONSTRAINT fk_fact_listing_classification
    FOREIGN KEY (classification_id)
    REFERENCES reporting.dim_listing_classification(classification_id);

ALTER DATABASE airbnb
SET search_path TO reporting;

ALTER TABLE fact_listings
    DROP COLUMN length_category,
    DROP COLUMN accommodation_group,
    DROP COLUMN property_group,
    DROP COLUMN rental_scope;
```

**Adding Property Type to the classification dimension** — decided at a later point in the project, requiring the dimension to be rebuilt:

```sql
CREATE TABLE public.property_types AS
SELECT listing_id, property_type, classification_id
FROM fact_listings
ORDER BY 1 ASC;

CREATE TABLE public.classifications AS
SELECT * FROM dim_listing_classifications;

CREATE TABLE public.property_classifications AS
SELECT
    listing_id, property_type, pt.classification_id,
    length_category, accommodation_group, property_group, rental_scope
FROM public.property_types pt
LEFT JOIN public.classifications c ON pt.classification_id = c.classification_id;

CREATE TABLE public.new_classifications AS
SELECT DISTINCT property_type, length_category, accommodation_group, property_group, rental_scope
FROM public.property_classifications;

ALTER TABLE public.new_classifications
    ADD COLUMN classification_key BIGINT
    GENERATED ALWAYS AS IDENTITY
    PRIMARY KEY;

ALTER TABLE public.property_classifications
    ADD COLUMN classification_key BIGINT;

UPDATE public.property_classifications pc
SET classification_key = nc.classification_key
FROM public.new_classifications nc
WHERE pc.property_type = nc.property_type
  AND pc.length_category = nc.length_category
  AND pc.accommodation_group = nc.accommodation_group
  AND pc.property_group = nc.property_group
  AND pc.rental_scope = nc.rental_scope;

-- Final classification dimension, with display-friendly column names
-- (property_group -> property_category, accommodation_group -> accommodation_size):
CREATE TABLE dim_classifications AS
SELECT
    classification_key,
    rental_scope,
    property_group AS property_category,
    accommodation_group AS accommodation_size,
    length_category AS stay_length_category,
    property_type
FROM public.new_classifications;

ALTER TABLE dim_classifications
    ADD CONSTRAINT pk_classifications
    PRIMARY KEY (classification_key);

ALTER TABLE fact_listings
    ADD COLUMN classification_key BIGINT;

UPDATE fact_listings fl
SET classification_key = ppc.classification_key
FROM public.property_classifications ppc
WHERE fl.listing_id = ppc.listing_id;

ALTER TABLE fact_listings
    ADD CONSTRAINT fk_fact_classifications
    FOREIGN KEY (classification_key)
    REFERENCES dim_classifications(classification_key);

ALTER TABLE fact_listings
    DROP CONSTRAINT fk_fact_listing_classification;

DROP TABLE dim_listing_classifications;

ALTER TABLE fact_listings
    DROP COLUMN property_type,
    DROP COLUMN classification_id;

CREATE INDEX idx_fact_listings_classification_key
    ON fact_listings (classification_key);
```

> **Note:** `dim_classifications` renames these fields for the reporting layer: `property_group` → `property_category`, and `accommodate_group` → `accommodation_size`. In the finished Power BI report these are displayed as **Property Category** and **Listing Size** — the "Property Group" naming used earlier in this document was the Phase 1 (2025) working name for the same field. See [DAX Measures.md](./DAX%20Measures.md) for the full terminology mapping used in the report.

### 7.4 Amenity dimension and bridge table

```sql
CREATE TABLE reporting.dim_amenities AS
SELECT * FROM amenities;

ALTER DATABASE airbnb
    SET search_path TO reporting, public;
```

Further amenity-text cleaning applied at this stage, beyond the initial pass in Section 3.4:

```sql
UPDATE dim_amenities
SET amenity = CASE
    WHEN amenity ILIKE '%housekeeping%'                THEN 'Housekeeping'
    WHEN amenity ILIKE '%conditioner%'                  THEN 'Conditioner'
    WHEN amenity ILIKE '%body soap%'                    THEN 'Body soap'
    WHEN amenity ILIKE '%sound system%'                 THEN 'Sound system'
    WHEN amenity ILIKE '%carport%'                      THEN 'Carport'
    WHEN amenity ILIKE 'free parking'                   THEN 'Free parking'
    WHEN amenity ILIKE 'paid parking'                   THEN 'Paid parking'
    WHEN amenity ILIKE '%garage%'                       THEN 'Garage'
    WHEN amenity ILIKE '%books and toys%'               THEN 'Books and toys'
    WHEN amenity ILIKE '%books and reading material%'   THEN 'Books and reading material'
    WHEN amenity ILIKE '%fast wifi%'                     THEN 'Fast Wifi'
    WHEN amenity ILIKE '%wifi%'                          THEN 'Wifi'
    WHEN amenity ILIKE '%game console%'                  THEN 'Game console'
    WHEN amenity ILIKE '%clothing storage%'              THEN 'Clothing storage'
    WHEN amenity ILIKE '%wardrobe%'                      THEN 'Clothing storage'
    WHEN amenity ILIKE '%dresser%'                       THEN 'Clothing storage'
    WHEN amenity ILIKE '%free weights%'                  THEN 'Free weights'
    ELSE amenity
END;

DELETE FROM dim_amenities
WHERE amenity ILIKE '%available%'
   OR amenity ILIKE '%10+ years old%'
   OR amenity ILIKE '%5-10 years old%'
   OR amenity ILIKE '%Olympic sized%';

DELETE FROM dim_amenities
WHERE amenity ILIKE '%free%'
  AND amenity NOT ILIKE '%free parking%'
  AND amenity NOT ILIKE '%free weights%';

-- Further cleaning
SELECT DISTINCT amenity FROM dim_amenities
WHERE amenity ILIKE '%electric%';

SELECT amenity, COUNT(*) FROM dim_amenities
GROUP BY 1
ORDER BY 1 ASC;

UPDATE dim_amenities
SET amenity = CASE
    WHEN amenity ILIKE '%body wash%' OR amenity ILIKE '%shower gel%'
      OR amenity ILIKE '%body soap%'                                  THEN 'Body wash / Shower Gel'
    WHEN amenity ILIKE '%head and shoulders%' OR amenity ILIKE '%head&shoulders%'
                                                                        THEN 'shampoo'
    WHEN amenity ILIKE '%baby bath%'                                   THEN 'baby bath'
    WHEN amenity ILIKE '%baby monitor%'                                THEN 'baby monitor'
    WHEN amenity ILIKE '%exercise equipment%'                          THEN 'exercise equipment'
    WHEN amenity ILIKE '%high chair%' AND amenity NOT ILIKE '%paid high chair%'
                                                                        THEN 'high chair'
    WHEN amenity ILIKE '%coffee machine%'                              THEN 'coffee maker'
    WHEN amenity ILIKE '%outdoor kitchen%'                             THEN 'outdoor kitchen'
    WHEN amenity ILIKE '%ev charger%'                                  THEN 'ev charger'
    WHEN amenity ILIKE '%breakfast bar%'                               THEN 'breakfast bar'
    WHEN amenity ILIKE '%changing table%'                              THEN 'changing table'
    WHEN amenity ILIKE '%closet%'                                      THEN 'Clothing storage'
    ELSE amenity
END;

-- Removing noise values that aren't real amenities (stray brand names, schedules, etc.)
DELETE FROM dim_amenities
WHERE amenity ILIKE '%1 day a week%'    OR amenity ILIKE '%2-5 years old%'
   OR amenity ILIKE '%3 days a week%'   OR amenity ILIKE '%dove%'
   OR amenity ILIKE '%tresseme%'        OR amenity ILIKE '%electric%'
   OR amenity ILIKE '%every day%'       OR amenity ILIKE '%friday%'
   OR amenity ILIKE '%infinity%'        OR amenity ILIKE '%kirkland%'
   OR amenity ILIKE '%liquid%'          OR amenity ILIKE '%monday%'
   OR amenity ILIKE '%natural%'         OR amenity ILIKE '%nivea%'
   OR amenity ILIKE '%nivea for men%'   OR amenity ILIKE '%nourishing%'
   OR amenity ILIKE '%old spice%'       OR amenity ILIKE '%24 hours%'
   OR amenity ILIKE '%specific hours%'  OR amenity ILIKE '%pantene%'
   OR amenity ILIKE '%press and hold%'  OR amenity ILIKE '%prosilk%'
   OR (amenity ILIKE '%safe%' AND amenity NOT ILIKE '%safety%')
   OR amenity ILIKE '%sink%'            OR amenity ILIKE '%sonos%'
   OR amenity ILIKE '%tesla%'           OR amenity ILIKE '%unscented%'
   OR amenity ILIKE '%varies%'          OR amenity ILIKE '%various%'
   OR amenity ILIKE '%wednesday%'       OR amenity ILIKE '%windex%'
   OR amenity ILIKE '%wood-burning%'    OR amenity ILIKE '%plain%'
   OR amenity ILIKE '%thursday%';

-- Removing one-off amenity values (appear for only a single listing)
DELETE FROM dim_amenities
WHERE amenity IN (
    SELECT amenity FROM dim_amenities
    GROUP BY amenity
    HAVING COUNT(*) = 1
);

UPDATE dim_amenities
SET amenity = REPLACE(INITCAP(amenity), ' And ', ' and ');

-- Deduplicating (listing_id, amenity) pairs
DELETE FROM dim_amenities a
WHERE a.ctid NOT IN (
    SELECT MIN(ctid)
    FROM dim_amenities
    GROUP BY listing_id, amenity
);
```

**Splitting into a true dimension + bridge table:**

```sql
ALTER TABLE dim_amenities RENAME TO bridge_amenities;

CREATE TABLE dim_amenities AS
SELECT DISTINCT amenity FROM bridge_amenities
ORDER BY amenity ASC;

ALTER TABLE dim_amenities
    ADD COLUMN amenity_id BIGINT
    GENERATED ALWAYS AS IDENTITY
    PRIMARY KEY;

ALTER TABLE bridge_amenities
    ADD COLUMN amenity_id BIGINT;

UPDATE bridge_amenities ba
SET amenity_id = da.amenity_id
FROM dim_amenities da
WHERE ba.amenity = da.amenity;

-- Checking for any unmatched rows:
SELECT * FROM bridge_amenities
WHERE amenity_id IS NULL;

ALTER TABLE bridge_amenities
    DROP COLUMN amenity;

ALTER TABLE bridge_amenities
    ADD CONSTRAINT fk_bridge_listing
    FOREIGN KEY (listing_id)
    REFERENCES fact_listings(listing_id);

ALTER TABLE bridge_amenities
    ADD CONSTRAINT fk_bridge_amenity
    FOREIGN KEY (amenity_id)
    REFERENCES dim_amenities(amenity_id);

ALTER TABLE bridge_amenities
    ADD CONSTRAINT pk_bridge_amenities
    PRIMARY KEY (listing_id, amenity_id);

CREATE INDEX idx_bridge_amenities_listing_id  ON bridge_amenities (listing_id);
CREATE INDEX idx_bridge_amenities_amenity_id  ON bridge_amenities (amenity_id);
CREATE INDEX idx_bridge_amenities_listing_amenity ON bridge_amenities (listing_id, amenity_id);
```

### 7.5 Host surrogate key

The model was still keying off the original Airbnb `host_id`; a surrogate key (`host_key`) was introduced so the dimension didn't depend on an external ID.

```sql
ALTER TABLE reporting.dim_neighbourhoods
    ALTER COLUMN neighbourhood_id TYPE BIGINT;

ALTER TABLE reporting.dim_hosts
    ADD COLUMN host_key BIGSERIAL;

ALTER TABLE reporting.fact_listings
    DROP CONSTRAINT fk_fact_host;

ALTER TABLE reporting.dim_hosts
    DROP CONSTRAINT pk_dim_hosts;

ALTER TABLE reporting.dim_hosts
    ADD PRIMARY KEY (host_key);

ALTER TABLE reporting.fact_listings
    ADD COLUMN host_key BIGINT;

UPDATE fact_listings fl
SET host_key = dh.host_key
FROM dim_hosts dh
WHERE fl.host_id = dh.host_id;

ALTER TABLE toronto.hosts
    ADD COLUMN host_key BIGINT;

UPDATE toronto.hosts th
SET host_key = dh.host_key
FROM dim_hosts dh
WHERE th.host_id = dh.host_id;

ALTER TABLE fact_listings
    DROP COLUMN host_id;

ALTER TABLE dim_hosts
    DROP COLUMN host_id;

ALTER TABLE fact_listings
    ADD CONSTRAINT fk_fact_hosts
    FOREIGN KEY (host_key)
    REFERENCES dim_hosts(host_key);
```

**Final Power BI reporting model:** 1 fact table (`fact_listings`), 3 dimension tables (`dim_hosts`, `dim_neighbourhoods`, `dim_classifications`), and 1 bridge table (`bridge_amenities` ↔ `dim_amenities`) — a standard star schema.

