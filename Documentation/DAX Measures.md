# Toronto Airbnb Analytics — Power BI DAX Measures

This document covers the DAX layer of the Phase 2 (2026) Power BI model: calculated columns added on top of the reporting schema (see [SQL Code.md](./SQL%20Code.md)), and every measure in the model, grouped by display folder, with the business question each one answers.

**Naming convention:** measures that calculate a dimension's share of a total are prefixed by that dimension — `PC` (Property Category), `LS` (Listing Size), `SL` (Stay Length Category), `NA` (Neighbourhood Area) — followed by what's being measured, e.g. `PC, % of Total Revenue`.

**Terminology note:** a few underlying columns carry different names than their Power BI display labels — see the terminology table in [Project Roadmap.md](./Project%20Roadmap.md) (Phase 2 → Power BI Model Extensions) for the full mapping.

---

## Calculated Columns

Added directly in the Power BI model, on top of the reporting schema built in SQL. Columns are grouped by the table they live on.

### dim_classifications

**`Segment`** — concatenates three classification fields into a single label (e.g. *"Apartment | Short-term | Group"*). This is what lets the Segment Matrix page compare specific combinations of characteristics rather than one dimension at a time.

```dax
Segment =
dim_classifications[Property Category]
& " | "
& dim_classifications[Stay Length Category]
& " | "
& dim_classifications[Listing Size]
```

### dim_hosts

**`Active Listings Count`** — the number of a host's listings that actually exist in the cleaned dataset. This replaces Airbnb's reported `total_listings_count` for grouping hosts. Airbnb's figure counts listings that are not in the cleaned dataset (for example, inactive listings excluded during data preparation), which made the original portfolio-size buckets inconsistent with the listings they contained.

```dax
Active Listings Count =
COUNTROWS(RELATEDTABLE(fact_listings))
```

**`Host Scale`** — buckets hosts by Active Listings Count for the portfolio-size analysis on the Host Performance page.

```dax
Host Scale =
VAR Listings = dim_hosts[Active Listings Count]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Listings), "Unknown",
        Listings <= 1, "1 Listing",
        Listings <= 3, "2-3 Listings",
        Listings <= 10, "4-10 Listings",
        "10+ Listings"
    )
```

**`Host Scale Sort`** — helper column that sorts `Host Scale` in logical order (1 Listing → 10+ Listings) instead of alphabetically. Applied with Column tools → Sort by column.

```dax
Host Scale Sort =
VAR Listings = dim_hosts[Active Listings Count]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Listings), 0,
        Listings <= 1, 1,
        Listings <= 3, 2,
        Listings <= 10, 3,
        4
    )
```

**`Superhost Status`** — converts the boolean Superhost flag into readable labels. Hosts with unknown status get their own "Unknown" value, so they can be excluded from the Superhost visual instead of being counted as non-Superhosts.

```dax
Superhost Status =
SWITCH(
    TRUE(),
    ISBLANK(dim_hosts[Superhost]), "Unknown",
    dim_hosts[Superhost] = TRUE(), "Superhost",
    "Not Superhost"
)
```

### fact_listings

**`Revenue bins`** — buckets Estimated Revenue into $10,000 brackets. It is the grouping key for the revenue-distribution histogram on the Executive Overview page.

**`Revenue Range`** — builds a readable label from `Revenue bins` (e.g. *"10K–20K"*), since the raw bin values alone aren't a useful axis label.

```dax
Revenue Range =
VAR Lower = [Revenue bins]
VAR Upper = Lower + 10000
RETURN
    IF(
        Lower = 0,
        "0–" & FORMAT(Upper / 1000, "0") & "K",
        FORMAT(Lower / 1000, "0") & "K–" &
        FORMAT(Upper / 1000, "0") & "K"
    )
```

**`Amenity Count`** — counts how many amenities are linked to each listing via the bridge table. It feeds the `Average Amenity Count` measure.

```dax
Amenity Count =
COUNTROWS(RELATEDTABLE(bridge_amenities))
```

**`Rating Band`** — groups the overall rating into four bands for the Host Performance page. Ratings below 4.0 are merged into one band because the lowest ratings have very few listings.

```dax
Rating Band =
VAR Rating = fact_listings[Overall Rating]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Rating), "No Rating",
        Rating < 4.0, "Below 4.0",
        Rating < 4.5, "4.0–4.5",
        Rating < 4.8, "4.5–4.8",
        "4.8–5.0"
    )
```

**`Rating Band Sort`** — helper column that sorts `Rating Band` from lowest to highest.

```dax
Rating Band Sort =
VAR Rating = fact_listings[Overall Rating]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Rating), 0,
        Rating < 4.0, 1,
        Rating < 4.5, 2,
        Rating < 4.8, 3,
        4
    )
```

## Measures

### Listings

| Measure | DAX | Purpose |
|---|---|---|
| Total Listings | `COUNT(fact_listings[listing_id])` | The baseline listing count for the current filter context. The denominator behind every "share of" measure in the model, and the headline listing-volume KPI. |
| % of Total Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(fact_listings[Revenue bins], fact_listings[Revenue Range])))` | What share of all listings fall into a given $10K revenue bracket — the measure behind the Executive Overview's revenue-distribution histogram. |
| PC, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Property Category])))` | Each Property Category's share of all listings — paired against its revenue share to reveal concentration (e.g. Apartments' listing share vs. revenue share). |
| LS, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Listing Size])))` | Same pattern, for Listing Size (Small/Medium/Large/Group). |
| SL, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Stay Length Category])))` | Same pattern, for Stay Length Category. |
| Segment, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Segment])))` | Same pattern, at the combined Segment level — used on the Segment Matrix page. |
| NA, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood Area])))` | Each broad geographic area's share of the overall market. |
| Neighbourhood, Listings % of Area | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood])))` | A finer-grained version of the above — a specific neighbourhood's share of listings *within its own area*, rather than citywide. |

### Revenue

| Measure | DAX | Purpose |
|---|---|---|
| Total Revenue | `SUM(fact_listings[Estimated Revenue (Last 365 Days)])` | Sum of estimated revenue across the current filter context — the headline revenue KPI and the base for every revenue measure below. |
| Average Revenue | `AVERAGE(fact_listings[Estimated Revenue (Last 365 Days)])` | Mean revenue per listing. Shown alongside the median since revenue is right-skewed and the two figures tell different stories about a segment. |
| Median Revenue | `MEDIAN(fact_listings[Estimated Revenue (Last 365 Days)])` | The primary revenue benchmark used throughout the report, since it's less distorted than the average by a small number of very high-revenue listings. |
| Median Revenue (min 50 listing count) | `IF([Total Listings] >= 50, [Median Revenue], BLANK())` | The same median, suppressed for any segment with fewer than 50 listings, so a tiny segment can't surface as a misleading "top performer." Used on the Segment Matrix page's Top 5 Segments chart. |
| PC, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_classifications[Property Category])))` | Each Property Category's share of total revenue — the revenue side of the listings-vs-revenue-share comparison. |
| LS, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_classifications[Listing Size])))` | Same pattern, for Listing Size. |
| SL, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_classifications[Stay Length Category])))` | Same pattern, for Stay Length Category. |
| NA, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood Area])))` | Same pattern, for Neighbourhood Area. |
| Neighbourhood, Revenue % of Area | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood])))` | A specific neighbourhood's share of revenue within its own area. |

### Occupancy

| Measure | DAX | Purpose |
|---|---|---|
| Total Occupied Nights | `SUM(fact_listings[Estimated Occupancy (Last 365 Days)])` | Sum of estimated occupied nights — the base figure behind every occupancy measure. |
| Average Occupied Nights | `AVERAGE(fact_listings[Estimated Occupancy (Last 365 Days)])` | Mean occupied nights per listing. |
| Median Occupied Nights | `MEDIAN(fact_listings[Estimated Occupancy (Last 365 Days)])` | The occupancy counterpart to Median Revenue — a less-skewed occupancy benchmark. |
| Average Occupancy % | `DIVIDE([Average Occupied Nights], 365)` | Converts average occupied nights into a percentage of the year — the figure shown on the Executive Overview KPI card and in every revenue/occupancy paired chart. |
| PC, % of Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_classifications[Property Category])))` | Each Property Category's share of total occupied nights. |
| LS, % of Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_classifications[Listing Size])))` | Same pattern, for Listing Size. |
| SL, % Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_classifications[Stay Length Category])))` | Same pattern, for Stay Length Category. |
| NA, % of Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood Area])))` | Same pattern, for Neighbourhood Area. |
| Neighbourhood, Occupied Nights % of Area | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood])))` | A specific neighbourhood's share of occupied nights within its own area. |
| Average Occupancy Rate *(work in progress)* | `AVERAGEX(VALUES(fact_listings[listing_id]), DIVIDE(SELECTEDVALUE(fact_listings[Estimated Occupancy (Last 365 Days)]), 365))` | An alternative to `Average Occupancy %`: rather than averaging occupied nights and converting the average to a rate, this calculates *each listing's* occupancy rate individually and then averages those rates. Left as a work in progress to preserve that modeling decision; `Average Occupancy %` is the version currently used in the report. |

### Amenities

All amenity performance measures are gated by a 2%-of-total-listings minimum, so an amenity offered on only a handful of listings can't produce a misleadingly high or low median.

| Measure | DAX | Purpose |
|---|---|---|
| Amenity Minimum Listings | `CALCULATE([Total Listings] * 0.02, REMOVEFILTERS(dim_amenities))` | Sets the 2%-of-total-listings floor used to gate the two measures below. |
| Amenity, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_amenities[Amenity])))` | What share of all listings offer a given amenity — the measure behind the Most Common Amenities chart. |
| Amenity, Revenue % | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_amenities[Amenity])))` | An amenity's share of total revenue, relative to all listings that offer any amenity. |
| Amenity, Occupancy % | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_amenities[Amenity])))` | Same pattern, for occupied nights. |
| Amenity Median Revenue | `IF([Total Listings] >= [Amenity Minimum Listings], [Median Revenue], BLANK())` | Median revenue specifically for listings offering a given amenity, blanked out below the prevalence floor — the core measure behind "Amenities Associated with Higher Revenue." |
| Amenity Median Occupied Nights | `IF([Total Listings] >= [Amenity Minimum Listings], [Median Occupied Nights], BLANK())` | Same pattern, for occupancy — behind "Amenities Associated with Higher Occupancy." |
| Average Amenity Count | `AVERAGE(fact_listings[Amenity Count])` | Average number of amenities per listing — a general indicator of how fully-featured a typical listing is, independent of which specific amenities are offered. |

### Price Metrics

| Measure | DAX | Purpose |
|---|---|---|
| Average Price | `AVERAGE(fact_listings[Price])` | Average nightly price across listings in the current filter context — a supporting pricing metric referenced alongside revenue and occupancy. |
