## Gold Layer — Analytics-Ready Star Schema

### **Overview**

This notebook builds the **Gold Layer** for the Strava analytics pipeline by transforming **curated Silver tables** into a **Power BI–friendly star schema** in **PostgreSQL**.

Gold focuses on:

* modeling data into **facts + dimensions** (clear grains, stable keys),
* creating **report-ready tables** for slicing (date/time, sport/workout, gear, location, device, segments),
* standardizing relationships across domains (activities, laps, efforts, zones, kudos, maps),
* ensuring the dataset is easy to consume by BI tools (especially **Power BI**).

---

### **Why Gold exists (what it solves)**

Silver tables are already clean and normalized, but Gold turns them into a **semantic layer**:

* consistent **surrogate keys** and **dimension tables** for filtering,
* multiple **fact tables** aligned to BI usage patterns,
* dedicated **mapping tables** for route/segment geometry,
* predictable **schema** that stays stable even as raw Strava payloads evolve.

---

### **Data sources (Silver → Gold)**

Gold reads from the **Silver layer** (schema/table names parameterized via `.env`), including domains like:

* **activities (flat)**
* **laps / best_efforts / segment_efforts**
* **zones (bucket-level)**
* **kudos**
* **gear**
* **locations** (from reverse geocoding)
* **maps** (point-level polyline output)

---

### **What the Gold Layer Produces**

#### **Dimensions**

Gold creates a set of reusable dimensions used across facts:

* **`gold.dim_calendar`** – calendar attributes derived from activity dates (for time intelligence).
* **`gold.dim_time`** – time-of-day dimension (hour/minute/time bucket slicing).
* **`gold.dim_sport_type`** – sport taxonomy (Run / Ride / Walk…).
* **`gold.dim_workout_type`** – workout classification (when available).
* **`gold.dim_device`** – device metadata (Garmin / Strava / etc., depending on the dataset).
* **`gold.dim_gear`** – shoes/bikes and their attributes.
* **`gold.dim_location`** – standardized location records (built from the **reverse-geocoded Silver locations**).
* **`gold.dim_segment`** – segment dictionary used for segment analytics.
* **`gold.dim_effort_type`** – “best effort” types (generated from the best-effort names, with a surrogate `id`).

#### **Facts**

Gold builds multiple fact tables with stable grains:

* **`gold.fact_activities`** – **1 row per activity** (the core fact table for most reporting).
* **`gold.fact_laps`** – **1 row per lap** (lap-level analysis).
* **`gold.fact_best_efforts`** – **1 row per best-effort object** (PR-style performance objects).
* **`gold.fact_segments_efforts`** – **1 row per segment effort** (segment analytics + Local Legend use cases).
* **`gold.fact_zones`** – **bucket-level zone rows** (time in HR/pace/power zones).
* **`gold.fact_kudos`** – **1 row per kudo** (social engagement analysis).
* **`gold.fact_maps_activities`** – **point-level geometry per activity** (decoded polylines → map visuals).
* **`gold.fact_maps_segments`** – **point-level geometry per segment** (segment polylines → map visuals).

---

### **Key Transformations (high level)**

1. **Star-schema modeling**

* Gold aligns all outputs to a **fact-centered** model, with consistent dimension joins.
* The main hub is **`fact_activities`**, referenced by the detailed facts (laps, zones, efforts, kudos, maps).

2. **Surrogate keys & BI-friendly joins**

* Dimensions use **stable IDs**, and fact tables carry **foreign keys** (date/time, gear, location, segment, effort type).
* **`dim_effort_type`** is created by deduplicating best-effort names and assigning IDs.

3. **Location standardization**

* Gold relies on **Silver’s reverse geocoding** output (created because activity-level location fields were removed from the official API, and segment locations were inconsistent across languages).
* This produces a clean **location dimension** suitable for filtering and grouping.

4. **Maps as point-level facts**

* Polylines are represented as **ordered point rows**, enabling:

  * route rendering,
  * segment geometry visuals,
  * spatial analysis in BI.

---

### **Persistence Layer (PostgreSQL — Gold Tables)**

* All outputs are written to a schema defined by `TARGET_G_SCHEMA` (typically **`gold`**).
* Tables are loaded using **full refresh** to keep runs **deterministic** and reproducible.
* After loading, the notebook applies **database constraints** (e.g., **primary keys**) where applicable to enforce model integrity.

---

### **Tech stack (as used in the notebook)**

* **Python**
* **pandas / numpy** (joins, deduplication, shaping, surrogate IDs)
* **SQLAlchemy** (`to_sql`, DDL execution for constraints)
* **PostgreSQL**
* **dotenv**, **logging**

---

### **Engineering Practices Demonstrated**

* **Semantic modeling**: clean separation into dimensions and facts.
* **Deterministic rebuilds**: full refresh strategy for reproducibility.
* **Stable grains**: explicit row-level meaning per table.
* **Constraint-aware loading**: enforcing PKs after bulk loads.
* **BI readiness**: schema designed for Power BI (filters, drillthrough, map visuals, time intelligence).
