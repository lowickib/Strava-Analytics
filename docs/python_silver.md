## Silver Layer — Curated Strava Tables

### **Overview**

This notebook builds the **Silver Layer** for the Strava analytics pipeline by transforming **raw Bronze API payloads** into **clean, relational, analysis-ready tables** in **PostgreSQL**.

Silver focuses on:

* **flattening & normalizing** nested Strava structures (arrays + objects),
* creating **consistent grains & keys** across tables,
* adding **light enrichment** (e.g., **reverse geocoding**, **map point extraction**),
* preparing the dataset for the downstream **Gold layer** and BI consumption (e.g., Power BI).

---

### **Objective**

Build a **repeatable transformation workflow** that:

* Reads data from **Bronze** (minimal-transformation tables)
* Produces curated **Silver tables** with clear **row grain**
* Normalizes nested fields into separate tables (**laps**, **best efforts**, **segment efforts**, **zones buckets**)
* Adds reusable enrichments:

  * **location dimension** via reverse geocoding,
  * **maps** as point-level geometry derived from polylines
* Persists outputs into PostgreSQL (typically schema: **`silver`**) using **full refresh (`replace`)**

---

### What the Silver Layer Produces

The notebook generates the following curated domains:

* **Activities (flat)** — core activity attributes used by most downstream models
* **Relative Effort** — isolate effort signals per activity (e.g., suffer score)
* **Gear** — extracted gear metadata associated with activities
* **Segments + Segment Efforts** — segment dictionary + per-activity efforts
* **Laps** — per-lap breakdown (normalized from nested arrays)
* **Best Efforts** — PR-style best effort objects per activity
* **Zones** — zone distributions (flattened to bucket-level rows)
* **Locations** — reverse-geocoded start coordinates (cached & deduplicated)
* **Maps** — polyline decoded into **point-level latitude/longitude** rows

---

### Data sources (Bronze → Silver)

Silver reads from the Bronze schema (parameterized via `.env`), primarily:

* `bronze.activities` (activity discovery snapshot)
* `bronze.activities_details` (full activity payload, including nested structures)
* `bronze.activities_zones` (zones payload with bucket distributions)
* `bronze.kudos` (kudos payload)
* (optionally) `bronze.segments_details` (segment metadata, if available)

---

### Key Transformations (high level)

#### 1) Normalization of nested JSON

The notebook extracts and normalizes nested arrays/objects from the detailed activity payload into **separate, relational tables**, including:

* `segment_efforts` → **`silver.segment_efforts`**
* `laps` → **`silver.laps`**
* `best_efforts` → **`silver.best_efforts`**
* segment dictionaries extracted from segment efforts → **`silver.segments`**

#### 2) Zone buckets to table rows

Zones are transformed from a nested payload into a **bucket-level** structure by:

* filtering to relevant zone types (e.g., **heartrate**, **pace**),
* sorting bucket ranges,
* exploding `distribution_buckets` into rows with bucket boundaries and time spent.

#### 3) Reverse geocoding locations (dimension-like table)

For activities with GPS start coordinates, the notebook:

* builds a unique coordinate list,
* uses reverse geocoding (rate-limited) to resolve **city/region/country**,
* stores results in a **deduplicated locations table** for reuse.

**Why reverse geocoding was required (design rationale):**

* Strava **removed location information from activity-level API responses**, so the pipeline could no longer rely on official `city/state/country` fields for activities.
* For **segments**, location attributes were **not standardized** (sometimes provided in **Polish**, sometimes in **English**), which made cleaning and consistent downstream modeling difficult.
* To solve both issues, the notebook uses **available coordinates** (e.g., activity start lat/lng) and resolves them into a **standardized location** representation (city/region/country) that is consistent across the dataset.

#### 4) Map polylines → point-level geometry

For each activity polyline, the notebook:

* decodes the polyline into a list of lat/lng points,
* explodes into **1 row per point** with ordering (`point_id`),
* persists into a maps table for downstream spatial visuals.

---

### Persistence Layer (PostgreSQL — Silver Tables)

All outputs are stored in a schema defined by `TARGET_S_SCHEMA` (typically `silver`), using **full refresh** (`if_exists="replace"`).

| Table (env var)            | Grain / Rows                          | Primary key (as implemented) | Notes                               |
| -------------------------- | ------------------------------------- | ---------------------------- | ----------------------------------- |
| `ACTIVITIES_S_TABLE`       | 1 row per activity                    | `id`                         | Curated, flat activity table        |
| `RELATIVE_EFFORT_S_TABLE`  | 1 row per activity                    | `activity_id`                | Effort signals (e.g., suffer score) |
| `GEAR_S_TABLE`             | 1 row per gear item                   | `id`                         | Extracted from activity payload     |
| `SEGMENTS_S_TABLE`         | 1 row per segment                     | `id`                         | Segment dictionary (from efforts)   |
| `SEGMENTS_EFFORTS_S_TABLE` | many rows per activity                | `id`                         | One row per segment effort          |
| `LAPS_S_TABLE`             | many rows per activity                | `id`                         | One row per lap                     |
| `BEST_EFFORTS_S_TABLE`     | many rows per activity                | `id`                         | One row per best-effort object      |
| `ZONES_S_TABLE`            | many rows per activity (bucket-level) | *(composite-like)*           | Exploded buckets from zones payload |
| `KUDOS_S_TABLE`            | many rows per activity                | `id`                         | Curated kudos table                 |
| `LOCATIONS_S_TABLE`        | 1 row per unique coordinate           | *(hash/id)*                  | Reverse-geocoded location cache     |
| `MAPS_S_TABLE`             | many rows per activity (point-level)  | *(composite-like)*           | Decoded polyline → points           |

> **Load behavior:** all tables are rebuilt each run to keep the Silver layer deterministic and easy to reproduce.

---

### Tech stack (as used in the notebook)

* **Python**
* **pandas** (normalize/explode, data quality guards)
* **SQLAlchemy** (engine, `to_sql`, constraints)
* **PostgreSQL**
* **geopy** (reverse geocoding for locations)
* **polyline** (decode Strava polylines into lat/lng points)
* **dotenv**, **logging**
* Standard libs: **time**, **datetime**, **json**, **os**

---

### Engineering Practices Demonstrated

* **Clear grains & keys** per table (ready for star-schema modeling in Gold)
* **Deterministic transforms** (full refresh strategy, reproducible outputs)
* **Safe JSON handling** (normalize + explode patterns with empty guards)
* **Rate-limited enrichment** for geocoding (practical & pipeline-friendly)
* **Separation of concerns**: Bronze = raw truth, Silver = curated structure
