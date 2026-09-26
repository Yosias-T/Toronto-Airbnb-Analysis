# Toronto Airbnb Market Analysis: An End-to-End Business Intelligence Case Study

**Yosias Teshome | Data Analyst**

*Toronto Airbnb Market Analysis — from exploratory SQL/Excel analysis to a Power BI reporting solution*

---

## Project Overview

Short-term rental hosts and prospective hosts in Toronto operate with limited visibility into what actually distinguishes a high-performing listing from an average one. Listing characteristics, pricing, location, and amenities all interact in ways that are difficult to evaluate intuitively across a market of over ten thousand active listings.

This project set out to answer a practical question for that audience: **what listing characteristics are associated with higher Airbnb revenue and occupancy in Toronto?** The objective was not to produce a single static answer, but a reusable analytical asset — a data model and reporting layer that a host, analyst, or investor could filter and explore to evaluate their own listing profile against market patterns.

The project was completed in two phases roughly a year apart. It began in 2025 as an exploratory SQL and Excel analysis: raw listings data was cleaned, modeled relationally in PostgreSQL, and analyzed manually in Excel. In 2026, after completing the Microsoft Power BI Data Analyst (PL-300) certification, the project was revisited and expanded into a fully interactive Power BI reporting solution — a dedicated reporting schema, a star-schema data model, a library of DAX measures, and a six-page report. The result is a project that documents genuine skill growth: the same business question, answered twice, with two very different toolsets and levels of analytical maturity.

## Business Context

Short-term rental platforms like Airbnb have become a significant and durable part of urban housing and tourism economies. For hosts, the difference between an average listing and a strong one can mean thousands of dollars in annual revenue — but that difference is rarely obvious from a single Airbnb listing page. For property managers and investors evaluating acquisition opportunities, understanding which characteristics (location, size, amenities, rental structure) tend to correlate with stronger performance is directly relevant to underwriting decisions. For municipal policymakers, patterns in stay length and rental scope carry implications for housing supply and short-term rental regulation.

Toronto is a useful market for this kind of analysis: it's large and diverse enough to produce meaningful segment-level patterns (across more than 140 neighbourhoods), and — as a market with active short-term rental regulation — it includes real regulatory signal in the data, such as the minimum-stay behavior visible in the Stay Length Category findings below.

This project treats the publicly available Inside Airbnb dataset as a stand-in for the kind of internal performance data a host, property manager, or analytics team would actually want to interrogate, and builds the kind of tooling — a clean data model plus a self-service report — that a business intelligence function would be expected to deliver.

## Project Goals

The project was structured around a core business question and a set of supporting analytical questions:

- **Core question:** What factors are associated with higher Airbnb revenue in Toronto?
- Which **Property Categories** (Apartment, House, Hotel, Villa, Farm) show the strongest revenue and occupancy performance?
- How does **Listing Size** (guest capacity) relate to revenue and occupancy?
- How does **Stay Length Category** (Short-term, Long-term, Seasonal, Lease-style) relate to revenue and occupancy — and does it reflect regulatory dynamics in the Toronto market?
- Which **neighbourhoods** command a revenue premium?
- Which **amenities** are associated with stronger revenue or occupancy, once small-sample noise is controlled for?
- Which combinations of characteristics (**segments**) outperform others, and which individual listings stand out?

Throughout, the project distinguishes between **documented associations** and **causal claims** — a distinction made explicit in the final Power BI report itself, which carries the disclaimer: *"Observed relationships represent associations within the dataset and should not be interpreted as evidence of causation."*

## Data Acquisition

The dataset is sourced from **Inside Airbnb**, an independent, publicly available data project that publishes scraped Airbnb listing data for major cities. This project uses the Toronto listings dataset, covering a data collection period of **July 2024 – June 2025**, with a scrape date of **June 14, 2025** — roughly 11,000 active listings after cleaning, each with property attributes, host attributes, pricing, availability, review data, and estimated revenue/occupancy figures.

Initial preparation happened in Excel, before the data ever reached the database: unnecessary columns (URLs, scrape metadata, largely empty fields) were removed, list-formatted fields (`amenities`, `host_verifications`) were reformatted from Python-style list notation into PostgreSQL array notation, currency and percentage fields were normalized to plain numbers, and a surrogate `listing_id` was generated to replace inconsistent source IDs. The prepared file was then imported into a PostgreSQL database for the analytical work that followed.

## Data Preparation

Data preparation followed a standard, defensible cleaning pipeline, all performed in SQL and documented in full in [SQL Code.md](./SQL%20Code.md).

**Missing values.** Missing `bathrooms` values were recovered from the free-text `bathrooms_text` field via regex extraction; a small number of rows with no usable data in either field were removed. Rows with no revenue or occupancy data over the trailing 365 days (9,895 of them) were flagged as **inactive** rather than deleted outright, since they don't reflect current market activity but still carry other useful attributes. Among the remaining active listings, 1,308 had no price. Before imputing anything, these rows were checked against occupancy, revenue, and review-count fields to confirm they were genuinely active and to test whether price could simply be calculated from revenue and occupancy directly — it couldn't, since revenue was missing for the same rows. Only then were prices imputed, using the **median price for the listing's room type and neighbourhood**, with a fallback to room-type-only median for the one listing with no matching comparison group. Every imputed row is tracked with a boolean flag, so any downstream analysis can include or exclude imputed values as needed.

**Feature engineering.** Several analytical fields were derived directly from raw attributes:
- **Stay Length Category** — binned from `minimum_nights` into Short-term (<28 days), Long-term (28–89), Seasonal (90–179), and Lease-style (180+), a categorization that turns out to carry real signal about Toronto's short-term rental regulatory environment.
- **Listing Size** — binned from `accommodates` into Small (1–2), Medium (3–4), Large (5–7), and Group (8+).
- **Property Category** — standardized dozens of raw `property_type` strings into a consistent set of categories (Apartment, House, Hotel, Villa, Castle, Farm, Other), applied iteratively across several passes as edge cases surfaced.
- **Rental Scope** — derived from `property_type` into Entire / Private room / Shared room.
- **Amenities** — unnested from a PostgreSQL array into a normalized one-row-per-amenity structure, then standardized to collapse wording variants (e.g., multiple ways of describing the same air conditioning or Wi-Fi amenity) that would otherwise fragment the amenity-level analysis.

**Outliers and data quality flags.** Prices were evaluated using the IQR method (upper fence of $434), and unusually high prices were flagged — not removed — so they remain visible but excludable. Listings that appeared to be duplicates (same name, latitude, and longitude, but differing price and description) were resolved by retaining the highest-priced row per group, treating it as the more complete/current record.

## Database Design

The cleaned data was normalized into a relational model with `listings` as the core table and a set of single-purpose child tables in a 1:1 relationship with it (`price_info`, `listing_details`, `reviews`, `booking_length`, `availability`), plus `hosts` and `neighbourhoods` in a many-to-one relationship, and `amenities` in a one-to-many relationship. Primary and foreign key constraints were enforced throughout.

This normalized structure served two purposes in Phase 1: it kept each table focused on a single concern (pricing separate from availability separate from review metrics), and it supported the SQL-driven grouped-summary analysis that fed the original Excel dashboard — building `GROUP BY` views by Property Category, Neighbourhood Group, Stay Length Category, and Listing Size, then cross-tabulating them against Rental Scope.

## Reporting Model

When the project was revisited for Power BI, the normalized structure was deliberately **denormalized** into a star schema — a different design goal than Phase 1's. Power BI's DAX engine and report visuals perform best against a small number of wide, purpose-built tables rather than a fully normalized relational structure, so the reporting layer prioritizes query performance and usability over storage efficiency.

The final model consists of:
- **`fact_listings`** — one row per listing, the grain of the entire model.
- **`dim_hosts`** — deduplicated host attributes, keyed on a surrogate `host_key` rather than the original `host_id`, since the surrogate key insulates the model from any future changes to the source ID.
- **`dim_neighbourhoods`** — both the raw neighbourhood name and the broader "Neighbourhood Area" grouping.
- **`dim_classifications`** — a single dimension holding the unique combinations of Property Category, Listing Size, Stay Length Category, Rental Scope, and Property Type (208 rows), rather than repeating those attributes on every one of the ~11,177 fact rows. This is a standard **role-playing/junk-dimension pattern**: several low-cardinality categorical attributes that are frequently filtered together are combined into one dimension instead of four or five separate ones.
- **`dim_amenities`** + **`bridge_amenities`** — because a listing can have many amenities and an amenity applies to many listings, this many-to-many relationship can't be modeled with a simple foreign key. A **bridge table** resolves it: `bridge_amenities` holds one row per (listing, amenity) pair, connecting `fact_listings` to `dim_amenities` without duplicating fact rows.

This structure is what makes measures like "amenities associated with higher revenue" possible in the first place — without the bridge table, amenity-level aggregation against a one-row-per-listing fact table wouldn't be expressible.

## Power BI Development

**Power Query and calculated columns.** Beyond the SQL-built reporting schema, a few fields were added directly in the Power BI model: a `Segment` calculated column that concatenates Property Category, Stay Length Category, and Listing Size into a single label for segment-level analysis; a `Revenue bins` / `Revenue Range` pair that buckets Estimated Revenue into readable $10K brackets for the Executive Overview histogram; and an `Amenity Count` column that counts each listing's amenities via the bridge table.

**Relationships.** `fact_listings` connects to `dim_hosts` via `host_key`, to `dim_neighbourhoods` via `neighbourhood_id`, and to `dim_classifications` via `classification_key` — each a standard one-to-many relationship from dimension to fact. The amenity relationship runs `dim_amenities` → `bridge_amenities` → `fact_listings`, the many-to-many pattern described above.

**Report architecture.** The report is organized as five analytical pages plus a reference page, navigated through a persistent left sidebar, with a consistent set of slicers (Neighbourhood Area, Property Category, Listing Size, Stay Length Category, Rental Scope) available on every analytical page so filtering carries across the whole report rather than resetting per page. The visual style uses a restrained palette — dark teal as the primary analytical color, coral reserved for highlights and key findings, with a dark header banner and light neutral canvas — chosen deliberately to read as a professional analytical tool rather than a decorative dashboard.

An earlier version of the report plan explored a Decomposition Tree and Key Influencers visual for the Listing Performance page; the finished report uses a scatter plot and a ranked table instead, a simplification made during development.

## DAX Measures

The DAX layer (full detail in [DAX Measures.md](./DAX%20Measures.md)) follows a consistent, extensible pattern rather than one-off calculations for each visual.

**Base measures** — `Total Listings`, `Total Revenue`, `Total Occupied Nights`, and their average/median variants — form the foundation every other measure builds on.

**"Share of total" measures** — a family of measures prefixed by the dimension they slice on (`PC` for Property Category, `LS` for Listing Size, `SL` for Stay Length Category, `NA` for Neighbourhood Area) that use `CALCULATE` with `REMOVEFILTERS` to compute each category's share of listings, revenue, or occupied nights independent of other active filters. This consistent naming and pattern makes the measure library predictable and easy to extend.

**Threshold-gated measures** — a deliberate data-quality control. `Amenity Minimum Listings` sets a 2%-of-total-listings floor, and `Amenity Median Revenue` / `Amenity Median Occupied Nights` return blank rather than a number for any amenity below that floor — preventing an amenity that appears on three listings from producing a headline-grabbing but statistically meaningless average. `Median Revenue (min 50 listing count)` applies the same logic to segment-level analysis, using an absolute floor rather than a percentage.

**KPI categories** covered by the measure library: listing volume, revenue (total, average, median, and share-of-total by every major dimension), occupancy (mirroring the revenue measures), price, and amenity performance. One measure, `Average Occupancy Rate`, is documented as a work in progress — a deliberate choice to preserve the reasoning behind an unfinished calculation rather than remove it, since the difference between "average of each listing's occupancy rate" and "average occupied nights converted to a rate" is a genuine modeling decision worth showing.

## Dashboard Design

The report comprises six pages, each answering a distinct question:

**Executive Overview** — *"Understanding factors associated with higher revenue."* A KPI strip (Total Listings, Total Revenue, Average Revenue, Median Revenue, Average Occupancy %) sets market-level context, followed by a revenue-distribution histogram, listings and median revenue by Property Category, listings and average revenue by Listing Size, and a paired chart comparing each Property Category's share of listings against its share of total revenue.

**Revenue Drivers** — *"Which characteristics are associated with higher revenue?"* Combination bar/line visuals pair median revenue with average occupancy across Stay Length Category and Listing Size, a horizontal bar chart ranks median revenue by Property Category, and a Top 10 Neighbourhoods chart surfaces the highest-median-revenue areas.

**Listing Performance** — *"Which listings and characteristics stand out?"* A revenue-vs-occupied-nights scatter plot, colored by Property Category, shows the relationship between the two target variables at the individual-listing level, alongside a Top 15 Listings by Revenue table for a ground-level view of the strongest performers.

**Segment Matrix** — *"Which listing segments perform the best?"* A performance matrix breaks out total listings, median and average occupied nights, and median, average, and total revenue by Property Category, paired with a Top 5 Segments (Property Category × Stay Length Category × Listing Size) chart and a written Key Findings panel — the most synthesized page in the report.

**Amenities** — *"Which amenities are associated with listing performance?"* Most Common Amenities, Amenities Associated with Higher Revenue, and Amenities Associated with Higher Occupancy (all gated at the 2% prevalence minimum described above), plus a prevalence-vs-performance scatter that makes clear that the most common amenities are not necessarily the ones associated with the strongest performance.

**About & Definitions** — the report's methodology page: business question, data source, collection period, scrape date, scope, term definitions for every derived field, and the association-versus-causation disclaimer.

## Key Insights

The following findings are drawn directly from the Segment Matrix page's Key Findings panel and the Revenue Drivers / Amenities pages. All describe **associations observed in the dataset**, not causal relationships.

- **Property mix and revenue concentration.** Apartments make up 54% of all listings and generate 66% of total estimated revenue — a disproportionate share relative to their listing count. Houses, at 42% of listings, generate a smaller proportion of revenue by comparison.
- **Size is associated with revenue.** All five top-performing segments by median revenue consist of Large or Group listings (5+ guests) — larger listings are consistently associated with higher revenue, though the Revenue Drivers page also shows average occupancy trending lower for Group-sized listings than for Medium or Large ones.
- **Stay length shows a similar revenue/occupancy trade-off.** Lease-style and Seasonal listings show higher median revenue than Short-term and Long-term listings, while Short-term listings show notably higher average occupancy — a pattern plausibly connected to Toronto's short-term rental regulatory environment, which the project's own Stay Length Category definitions flag as a likely factor behind the Long-term category's existence.
- **Location carries a meaningful revenue premium.** The Top 10 Neighbourhoods by Median Revenue span a wide range (from roughly $36K down to $20K), indicating that neighbourhood is a strong differentiator independent of property characteristics.
- **Certain amenities are associated with higher revenue**, notably ones signaling either a larger or higher-end property (Garage, Lake View, Building Staff, City Skyline View, Waterfront) or added guest amenities (Shared Sauna, Elevator, Treadmill, Pool) — while the most *common* amenities (Smoke Alarm, Heating, Air Conditioning, Wifi) are largely baseline expectations rather than differentiators, as the prevalence-vs-performance scatter plot on the Amenities page makes visually clear.

## Exception Analysis

A meaningful part of this project's rigor lies in how it identified and handled data exceptions — records that didn't fit cleanly into the standard analytical path — rather than silently discarding them.

**Inactive listings.** 9,895 listings had no revenue and no occupancy recorded over the trailing 365 days. Rather than deleting these rows, they were retained in the raw table and flagged with an `inactive` boolean, preserving the option to reference them later (for instance, to compare active vs. inactive listing characteristics) while excluding them from the core revenue/occupancy analysis, since they don't reflect current market conditions.

**Imputed pricing.** 1,308 active listings had no recorded price. The first step wasn't imputation — it was a diagnostic check against occupancy, revenue, and review counts to confirm these listings were genuinely active (rather than simply under-flagged inactive listings) and to test whether price could be derived mathematically from revenue and occupancy (`price = revenue ÷ occupancy`). That check showed these rows had occupancy and/or review activity but not revenue, so both figures needed for a direct calculation weren't available together. Only once math was ruled out were prices estimated, using the median price for each listing's specific room type and neighbourhood combination — a locally-contextualized imputation that respects the fact that price varies substantially by both factors. The one listing with a unique room-type/neighbourhood combination and no comparison group was imputed using a room-type-only fallback. (For the smaller number of rows still missing *revenue* after imputation, revenue was calculated directly as `price × occupancy` — a relationship validated against existing rows where all three values were already known.) Every imputed row carries a boolean flag, so the choice to include or exclude imputed values in any downstream analysis remains available and auditable rather than hidden.

**Price outliers.** Using the IQR method (Q1 = $79, Q3 = $221, IQR = $142), an upper fence of $434 was established, and listings priced above it were flagged rather than removed. This preserves legitimate high-end listings in the dataset for reporting purposes while giving any analysis the option to exclude them when outliers would distort an average.

**False duplicates.** A number of listings shared identical name, latitude, and longitude but differed in price and description — evidence of the same physical property re-listed or updated over time rather than true duplicates. These were resolved by retaining only the highest-priced row per group, treated as the more complete or current record, rather than arbitrarily keeping the first or last row encountered.

Across all four categories, the consistent pattern is **flag and preserve rather than silently delete** wherever the excluded data still had potential analytical value — a data-governance instinct that keeps the pipeline auditable and defensible when presented to stakeholders.

## Technical Skills Demonstrated

- **SQL** — DDL and DML across a multi-table schema, constraint design (primary/foreign keys, cascading deletes), aggregate and window functions (`PERCENTILE_CONT`, `DISTINCT ON`), array handling and unnesting (PostgreSQL `ARRAY`, `unnest`), and view-based reporting layers.
- **Data Modeling** — entity-relationship design for a normalized relational schema, with tables scoped to single analytical concerns and referential integrity enforced throughout.
- **Dimensional Modeling** — star-schema design for the Power BI reporting layer, including fact/dimension/bridge table patterns, a junk-dimension approach for low-cardinality classification attributes, and surrogate key design to decouple the model from source-system identifiers.
- **Power Query / Data Modeling in Power BI** — calculated column design (concatenated segment labels, binned ranges, related-table counts) and relationship configuration across a multi-table model.
- **DAX** — context manipulation with `CALCULATE` and `REMOVEFILTERS`, iterator functions (`AVERAGEX`), and a deliberately consistent, extensible measure-naming pattern.
- **Power BI Report Design** — multi-page report architecture with persistent cross-page filtering, KPI card design, and a considered visual/color system.
- **Data Visualization** — visual-type selection matched to analytical intent (histograms for distribution, scatter plots for relationship, matrices for segment comparison, ranked bar charts for comparison across categories).
- **GitHub / Technical Documentation** — structured, cross-referenced project documentation (roadmap, SQL reference, DAX reference, and this case study) written for reviewer comprehension rather than just personal notes.

## Lessons Learned

**Technical.** The most instructive technical lesson of this project is the deliberate shift from a normalized relational model (Phase 1) to a denormalized star schema (Phase 2) — the same underlying data, modeled two different ways for two different purposes. Normalization served SQL-based analysis and data integrity well; it would have been a poor fit for Power BI's DAX engine and report performance. Building both, in sequence, made that trade-off concrete rather than theoretical. A second lesson came from the amenity bridge table: many-to-many relationships can't be modeled away, and resolving one properly (rather than working around it with string concatenation or repeated fact rows) was a necessary step to make amenity-level analysis possible at all. A third: introducing surrogate keys (`host_key`, `classification_key`, `amenity_id`) partway through the project — after the model had already changed shape once — reinforced why surrogate keys are standard practice in dimensional modeling: they insulate the model from exactly the kind of schema evolution this project went through.

**Business.** The revenue-concentration finding — Apartments generating 66% of revenue from 54% of listings — is a reminder that headline averages can obscure how unevenly a market's revenue is actually distributed, which matters directly for how a stakeholder should weight "average revenue by category" figures. The threshold-gating decisions (2% amenity prevalence, 50-listing segment minimum) were themselves a business lesson: a dashboard that reports every statistic regardless of sample size will eventually mislead someone, and building that judgment into the DAX layer — rather than relying on every report viewer to apply it manually — is a more reliable way to keep the analysis honest.

## Conclusion

This project delivers a complete analytics pipeline: raw, messy source data taken through documented cleaning and feature engineering, modeled relationally for analytical integrity, then remodeled dimensionally for reporting performance, and surfaced through an interactive, six-page Power BI report backed by a disciplined, consistently-patterned DAX layer. It answers its founding business question — what's associated with higher Airbnb revenue in Toronto — with specific, segment-level, statistically-guarded findings rather than vague generalities, while being explicit throughout about the line between association and causation.

Just as importantly, the project's two-phase structure — a 2025 SQL/Excel analysis revisited and substantially expanded after 2026 PL-300 certification — documents genuine analytical growth: from ad hoc SQL querying and static Excel charts to a governed data model, a reusable measure library, and a self-service report a stakeholder could actually explore on their own. That trajectory, as much as any single finding in the dashboard, is the point of including this project in a data analyst portfolio.

---

*Full technical documentation: [Project Roadmap.md](./Project%20Roadmap.md) (project history and development steps) · [SQL Code.md](./SQL%20Code.md) (database design and transformation logic) · [DAX Measures.md](./DAX%20Measures.md) (Power BI measure and calculated-column reference) · [Data Dictionary.md](./Data%20Dictionary.md) (field-level definitions) · [Reporting Schema.md](./Reporting%20Schema.md) (star-schema design)*
