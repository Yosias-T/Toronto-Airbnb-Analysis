# Toronto Airbnb Analytics — Power BI DAX Measures

This document covers the DAX layer of the Phase 2 (2026) Power BI model: calculated columns added on top of the reporting schema (see [`SQL_CODE.md`](./SQL_CODE.md), Section 7), and all measures, grouped by their display folder in the model.

**Naming convention:** measures that calculate a dimension's share of a total are prefixed by that dimension — `PC` (Property Category), `LS` (Listing Size), `SL` (Stay Length Category), `NA` (Neighbourhood Area) — followed by what's being measured, e.g. `PC, % of Total Revenue`.

**Terminology note:** a few underlying columns carry different names than their Power BI display labels. See the terminology table in the project roadmap (Phase 2 → Power BI Model Extensions) for the full mapping.

---

## Calculated Columns

Added directly in the Power BI model, beyond the reporting schema built in SQL.

**`Segment`** (`dim_classifications`) — concatenates three classification fields into a single label for segment-level analysis (e.g. *"Apartment | Short-term | Group"*):

```dax
Segment =
dim_classifications[Property Category]
& " | "
& dim_classifications[Stay Length Category]
& " | "
& dim_classifications[Listing Size]
```

**`Revenue bins`** (`fact_listings`) — buckets Estimated Revenue into $10,000 brackets.

**`Revenue Range`** (`fact_listings`) — builds a readable label from `Revenue bins` (e.g. *"10K–20K"*) for the revenue-distribution histogram on the Executive Overview page:

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

**`Amenity Count`** (`fact_listings`) — number of amenities linked to each listing via the bridge table:

```dax
Amenity Count =
COUNTROWS(RELATEDTABLE(bridge_amenities))
```

---

## Measures

### Listings

| Measure | DAX |
|---|---|
| Total Listings | `COUNT(fact_listings[listing_id])` |
| % of Total Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(fact_listings[Revenue bins], fact_listings[Revenue Range])))` |
| PC, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Property Category])))` |
| LS, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Listing Size])))` |
| SL, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Stay Length Category])))` |
| Segment, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_classifications[Segment])))` |
| NA, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood Area])))` |
| Neighbourhood, Listings % of Area | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood])))` |

### Revenue

| Measure | DAX |
|---|---|
| Total Revenue | `SUM(fact_listings[Estimated Revenue (Last 365 Days)])` |
| Average Revenue | `AVERAGE(fact_listings[Estimated Revenue (Last 365 Days)])` |
| Median Revenue | `MEDIAN(fact_listings[Estimated Revenue (Last 365 Days)])` |
| Median Revenue (min 50 listing count) | `IF([Total Listings] >= 50, [Median Revenue], BLANK())` |
| PC, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_classifications[Property Category])))` |
| LS, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_classifications[Listing Size])))` |
| SL, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_classifications[Stay Length Category])))` |
| NA, % of Total Revenue | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood Area])))` |
| Neighbourhood, Revenue % of Area | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood])))` |

### Occupancy

| Measure | DAX |
|---|---|
| Total Occupied Nights | `SUM(fact_listings[Estimated Occupancy (Last 365 Days)])` |
| Average Occupied Nights | `AVERAGE(fact_listings[Estimated Occupancy (Last 365 Days)])` |
| Median Occupied Nights | `MEDIAN(fact_listings[Estimated Occupancy (Last 365 Days)])` |
| Average Occupancy % | `DIVIDE([Average Occupied Nights], 365)` |
| PC, % of Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_classifications[Property Category])))` |
| LS, % of Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_classifications[Listing Size])))` |
| SL, % Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_classifications[Stay Length Category])))` |
| NA, % of Occupied Nights | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood Area])))` |
| Neighbourhood, Occupied Nights % of Area | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_neighbourhoods[Neighbourhood])))` |
| Average Occupancy Rate *(work in progress)* | `AVERAGEX(VALUES(fact_listings[listing_id]), DIVIDE(SELECTEDVALUE(fact_listings[Estimated Occupancy (Last 365 Days)]), 365))` — the idea was to calculate each listing's occupancy rate individually and then average those rates, rather than averaging occupied nights and converting the average to a rate. Left as a work in progress; `Average Occupancy %` above is the version currently used in the report. |

### Amenities

All amenity performance measures are gated by a 2%-of-total-listings minimum, so an amenity with only a handful of listings can't produce a misleadingly high or low median.

| Measure | DAX |
|---|---|
| Amenity Minimum Listings | `CALCULATE([Total Listings] * 0.02, REMOVEFILTERS(dim_amenities))` |
| Amenity, % of Listings | `DIVIDE([Total Listings], CALCULATE([Total Listings], REMOVEFILTERS(dim_amenities[Amenity])))` |
| Amenity, Revenue % | `DIVIDE([Total Revenue], CALCULATE([Total Revenue], REMOVEFILTERS(dim_amenities[Amenity])))` |
| Amenity, Occupancy % | `DIVIDE([Total Occupied Nights], CALCULATE([Total Occupied Nights], REMOVEFILTERS(dim_amenities[Amenity])))` |
| Amenity Median Revenue | `IF([Total Listings] >= [Amenity Minimum Listings], [Median Revenue], BLANK())` |
| Amenity Median Occupied Nights | `IF([Total Listings] >= [Amenity Minimum Listings], [Median Occupied Nights], BLANK())` |
| Average Amenity Count | `AVERAGE(fact_listings[Amenity Count])` |

### Price Metrics

| Measure | DAX |
|---|---|
| Average Price | `AVERAGE(fact_listings[Price])` |
