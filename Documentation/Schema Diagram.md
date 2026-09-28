# Schema ERD — Toronto Airbnb Market Analysis

This diagram shows the **Power BI semantic model** (star schema) as it currently exists in the report, including calculated columns.

**Reading the diagram**

* Column names match the Power BI model. Mermaid does not allow spaces or special characters in attribute names, so spaces are replaced with underscores and parentheses are dropped (e.g. `Estimated Revenue (Last 365 Days)` appears as `Estimated\_Revenue\_Last\_365\_Days`). Exact Power BI names, and the matching PostgreSQL source columns, are listed in the [Data Dictionary](./Data_Dictionary.md).
* Data types are Power BI types (`int64`, `decimal`, `double`, `string`, `boolean`, `dateTime`). PostgreSQL types are listed in the Data Dictionary.
* `"hidden"` = hidden from Report view. `"calculated"` = DAX calculated column (not present in the PostgreSQL source).
* Key columns (`\*\_key`, `\*\_id`) keep their snake\_case source names because they are hidden technical fields.

```mermaid
erDiagram

    dim\_hosts ||--o{ fact\_listings : "hosts"
    dim\_neighbourhoods ||--o{ fact\_listings : "located in"
    dim\_classifications ||--o{ fact\_listings : "classifies"

    fact\_listings ||--o{ bridge\_amenities : "has"
    dim\_amenities ||--o{ bridge\_amenities : "maps"

    dim\_hosts {
        int64 host\_key PK "hidden"
        string Host\_Name
        dateTime Host\_Since
        string Host\_Location
        boolean Superhost
        int64 Active\_Listings\_Count "calculated"
        string Host\_Scale "calculated"
        int64 Host\_Scale\_Sort "calculated"
        string Superhost\_Status "calculated"
    }

    dim\_neighbourhoods {
        int64 neighbourhood\_id PK "hidden"
        string Neighbourhood
        string Neighbourhood\_Area
    }

    dim\_classifications {
        int64 classification\_key PK "hidden"
        string Rental\_Scope
        string Property\_Category
        string Listing\_Size
        string Stay\_Length\_Category
        string Property\_Type
        string Segment "calculated"
    }

    dim\_amenities {
        int64 amenity\_id PK "hidden"
        string Amenity
    }

    fact\_listings {
        int64 listing\_id PK "hidden"

        int64 host\_key FK "hidden"
        int64 neighbourhood\_id FK "hidden"
        int64 classification\_key FK "hidden"

        string Listing\_Name

        int64 Accommodates
        double Bathrooms
        int64 Bedrooms
        int64 Beds

        decimal Price
        boolean Is\_Price\_Outlier

        int64 Minimum\_Nights
        int64 Maximum\_Nights

        int64 Availability\_Last\_30\_Days
        int64 Availability\_Last\_365\_Days
        int64 Availability\_End\_Of\_Fiscal\_Year

        int64 Total\_Reviews
        int64 Total\_Reviews\_Last\_12\_Months
        int64 Total\_Reviews\_Last\_Year

        int64 Estimated\_Occupancy\_Last\_365\_Days
        decimal Estimated\_Revenue\_Last\_365\_Days

        dateTime First\_Review
        dateTime Last\_Review

        double Rating
        double Accuracy\_Score
        double Cleanliness\_Score
        double Check\_In\_Score
        double Communication\_Score
        double Location\_Score
        double Value\_Score

        double Reviews\_Per\_Month

        boolean Instantly\_Bookable

        double Latitude
        double Longitude

        int64 Amenity\_Count "calculated"
        decimal Revenue\_bins "calculated"
        string Revenue\_Range "calculated"
        string Rating\_Band "calculated"
        int64 Rating\_Band\_Sort "calculated"
    }

    bridge\_amenities {
        int64 listing\_id FK "hidden"
        int64 amenity\_id FK "hidden"
    }
```

## Relationships

All relationships are one-to-many, with the filter flowing from the "one" side to the "many" side unless noted.

|From (many side)|To (one side)|Filter direction|Notes|
|-|-|-|-|
|`fact\_listings\[host\_key]`|`dim\_hosts\[host\_key]`|Single||
|`fact\_listings\[neighbourhood\_id]`|`dim\_neighbourhoods\[neighbourhood\_id]`|Single||
|`fact\_listings\[classification\_key]`|`dim\_classifications\[classification\_key]`|Single||
|`bridge\_amenities\[listing\_id]`|`fact\_listings\[listing\_id]`|**Both**|Bi-directional so that an amenity selection filters `fact\_listings`, which is what every `Amenity, …` measure relies on.|
|`bridge\_amenities\[amenity\_id]`|`dim\_amenities\[amenity\_id]`|Single||

## Hierarchies

|Table|Hierarchy|Levels|
|-|-|-|
|`dim\_neighbourhoods`|Neighbourhood Area Hierarchy|Neighbourhood Area → Neighbourhood|
|`dim\_classifications`|Property Category Hierarchy|Property Category → Property Type|

## Other Model Objects

* **`bridge\_amenities`** is a hidden table; users interact with amenities only through `dim\_amenities`.
* **`Measures List`** is a disconnected, empty calculated table used only as a home for DAX measures (35 measures organized into the display folders *Revenue*, *Occupancy*, *Listings*, *Amenities*, and *Price Metrics*). It has no relationships, so it is not shown in the diagram. See the Data Dictionary for the full measure list.
* **`room\_type`** exists in the PostgreSQL `reporting.fact\_listings` table but is removed in Power Query and is **not loaded** into the Power BI model.

## Reporting Layer Design

The reporting model was transformed from a highly normalized source structure into a dimensional model optimized for analytics and Power BI reporting.

### Fact Table

* `fact\_listings`

  * One row per Airbnb listing.
  * Stores occupancy, revenue, pricing, review, and availability metrics.

### Dimensions

* `dim\_hosts`
* `dim\_neighbourhoods`
* `dim\_classifications`
* `dim\_amenities`

### Bridge Table

* `bridge\_amenities`

  * Resolves the many-to-many relationship between listings and amenities.

### Benefits

* Simplified DAX calculations
* Improved query performance
* Reduced model complexity
* Consistent business classifications

