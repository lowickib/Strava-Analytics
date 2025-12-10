## Data Model Overview

The model is built around the **fact table** **`gold fact_activities`**, which stores individual **Strava activities** (runs, rides etc.).
It is connected to several **dimension tables** that describe **time**, **sport type**, **gear**, **location**, and **workout type**, plus additional fact tables for laps, best efforts, segments, zones and kudos.

There are also **helper / calculated tables** (**`Measures Table`**, **`SummaryType`**, **`Training Zones Time Intelligence`**) used mainly for DAX measures, slicers and advanced visuals.

![Data Model](png/Data_Model.png)

### Relationships

| **From table**               | **To table**            | **From column**                      | **To column**                  | **Active**        | **Description**                                                                                                                                         |
| ---------------------------- | ----------------------- | ------------------------------------ | ------------------------------ | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gold fact_activities`       | `gold dim_calendar`     | `fact_activities[date]`              | `dim_calendar[date]`           | **Yes**           | Used for **date**, **month**, **year**, **week**, **day of week** analysis and **time intelligence**.                                                   |
| `gold fact_activities`       | `gold dim_time`         | `fact_activities[time]`              | `dim_time[time]`               | **Yes**           | Used for **hour-of-day** and **day part** analysis (**morning/evening** etc.).                                                                          |
| `gold fact_activities`       | `gold dim_gear`         | `fact_activities[gear_id]`           | `dim_gear[id]`                 | **Yes**           | Connects each activity to **shoes**, **bikes** or other **gear**.                                                                                       |
| `gold fact_activities`       | `gold dim_location`     | `fact_activities[location_id]`       | `dim_location[id]`             | **Yes**           | Used to analyze activities by **country**, **region**, **city**.                                                                                        |
| `gold fact_activities`       | `gold dim_device`       | `fact_activities[device_id]`         | `dim_device[id]`               | **Yes**           | Shows which **device** (**watch / bike computer / phone**) was used.                                                                                    |
| `gold fact_activities`       | `gold dim_workout_type` | `fact_activities[workout_type_id]`   | `dim_workout_type[id]`         | **Yes**           | Distinguishes **workout types** (e.g. **Run – Race**, training runs, rides etc.).                                                                       |
| `gold fact_activities`       | `gold dim_sport_type`   | `fact_activities[sport_type_id]`     | `dim_sport_type[id]`           | **Yes**           | Groups activities into **sport types** (Run, Ride, etc.) and **summary categories**.                                                                    |
| `gold fact_best_efforts`     | `gold dim_effort_type`  | `fact_best_efforts[effort_type_id]`  | `dim_effort_type[id]`          | **Yes**           | Links **best efforts** (e.g. **1 km**, **5 km**, **10 km**) to their **effort type**.                                                                   |
| `gold fact_best_efforts`     | `gold fact_activities`  | `fact_best_efforts[activity_id]`     | `fact_activities[id]`          | **Yes**           | Connects each **best effort** to the **underlying activity**.                                                                                           |
| `gold fact_kudos`            | `gold fact_activities`  | `fact_kudos[activity_id]`            | `fact_activities[id]`          | **Yes**           | Enables analysis of **who gave kudos** to which activity.                                                                                               |
| `gold fact_laps`             | `gold fact_activities`  | `fact_laps[activity_id]`             | `fact_activities[id]`          | **Yes**           | Stores **laps / splits** for each activity.                                                                                                             |
| `gold fact_segments_efforts` | `gold dim_segment`      | `fact_segments_efforts[segment_id]`  | `dim_segment[id]`              | **Yes**           | Links individual **segment efforts** to **segment definitions**.                                                                                        |
| `gold fact_segments_efforts` | `gold fact_activities`  | `fact_segments_efforts[activity_id]` | `fact_activities[id]`          | **Yes**           | Connects **segment efforts** to the **activity** they belong to.                                                                                        |
| `gold fact_zones`            | `gold fact_activities`  | `fact_zones[activity_id]`            | `fact_activities[id]`          | **Yes**           | Provides **time in zones** (**pace / heart rate**) per activity.                                                                                        |
| `gold fact_maps_activities`  | `gold fact_activities`  | `gold fact_maps_activities[map_id]`  | `gold fact_activities[map_id]` | **Yes**           | Stores **polyline / coordinates** for **entire activities**.                                                                                            |
| `gold fact_maps_segments`    | `gold dim_segment`      | `gold fact_maps_segments[map_id]`    | `gold dim_segment[map_id]`     | **Yes**           | Stores **polyline / coordinates** for **segments**.                                                                                                     |
| `gold dim_segment`           | `gold dim_location`     | `gold dim_segment[location_id]`      | `gold dim_location[id]`        | **No (inactive)** | **Helper relationship** – segment location is typically resolved via **LOOKUPVALUE** into calculated columns (**Country, Region, Locality, Location**). |

Możesz to wkleić bezpośrednio do **README.md** – GitHub ładnie wyrenderuje tabelę.




### Dimension Tables

#### `gold dim_calendar`

**Purpose:** Standard **date dimension** for all time-based analysis.

* One row per **date**.
* Columns for **year**, **month**, **week**, **day**, **day of week**, **day of year**, **is_weekend**.
* Pre-calculated **hierarchy**: **Year → Month → Week → Date**.
* Additional **presentation columns**:

  * **`month_name`**, **`month_name_short`**, **`month_year`**, **`month_start_date`**
  * **`day_of_week_name`**, **`Day Of Week Name Short`**
  * **`week_start_date`**, **`year_start_date`**
* Used as the main **date table** for all measures and time intelligence.

---

#### `gold dim_time`

**Purpose:** **Time-of-day dimension** used with `fact_activities[time]`.

* One row per **time** (hour + minute + second).
* Columns: **hour**, **minute**, **second**, **hour_minute**, **hour_label**.
* **day_part** (e.g. morning / afternoon / evening) and **day_part_number** for ordered grouping.
* Used to analyze **when during the day** activities are performed.

---

#### `gold dim_device`

**Purpose:** Devices used to record activities.

* **`id`** – device key.
* **`device_name`** – e.g. specific watch / bike computer model.
* Used to analyze **activities by device** and for **tooltips** or labels.

---

#### `gold dim_effort_type`

**Purpose:** Types of **best efforts**.

* **`id`** – effort type key.
* **`name`** – e.g. “1 km”, “5 km”, “10 km”, “Half marathon”.
* Used by **`fact_best_efforts`** to interpret **effort distance/type**.

---

#### `gold dim_gear`

**Purpose:** **Gear dimension** for shoes, bikes etc.

* **`id`** – unique gear ID (string, as in Strava API).
* **`name`** – user-friendly gear name.
* **`distance_m`**, **`distance_km`** – total distance covered with the gear.
* **`gear_type`** – type, e.g. **Shoes**, **Bike**.
* **`status`** – e.g. **active**, **retired**.
* Used to analyze **mileage by gear** and **gear performance**.

---

#### `gold dim_location`

**Purpose:** **Location dimension** for activities.

* **`id`** – location key.
* **`country`**, **`region`**, **`locality` (city)** with proper **data categories** (Country/State/City).
* **`Location Hierarchy`**: **Country → Region → Locality**.
* Used for **maps**, **geographical analysis**, and **heatmaps by location**.

---

#### `gold dim_segment`

**Purpose:** Metadata for **Strava segments**.

* Basic attributes:

  * **`id`**, **`name`**, **`activity_type`**.
  * **`distance`** (in meters), **`Distance Km`** (formatted), **`distance_km_str`** (text).
* Elevation and difficulty:

  * **`average_grade`**, **`maximum_grade`**, **`elevation_high`**, **`elevation_low`**, **`elevation_difference`**, **`total_elevation_gain`**, **`climb_category`**.
* Location:

  * **`location_id`** (FK to `dim_location`), plus calculated **`Country`**, **`Region`**, **`Locality`**, **`Location`** text, and **Location Hierarchy**.
* Visuals and stats:

  * **`elevation_profile`** (image URL for elevation chart).
  * Athlete stats: **`athlete_segment_stats_pr_elapsed_time`**, **`..._pr_date`**, **`..._pr_activity_id`**, **`..._effort_count`**.
  * **KOM/QOM/XOMS** fields: **`xoms_kom`**, **`xoms_qom`**, **`xoms_overall`**, and derived **pace**, **time differences**, **combined results**.
* Local legend:

  * **`local_legend_athlete_id`**, **`local_legend_title`**, **`local_legend_profile`** (image), **`local_legend_effort_description`**, **`local_legend_effort_count`**.
  * Measures/columns like **`Efforts Last 90 Days`**, **`Efforts To Local Legend`** to show how many more efforts are needed.
* Metadata:

  * **`created_date`**, **`updated_date`**.
  * **`map_id`** – link to **segment map geometry**.

---

#### `gold dim_sport_type`

**Purpose:** Classification of **sport types**.

* **`id`**, **`sport_type`** – detailed sport name (Run, Ride, Walk, etc.).
* **`sport_type_summary`**, **`sport_type_summary_number`** – grouping for summary logic (e.g. **distance-based** vs **time-based** sports).
* Calculated column **`Sport Icon`** – URL for sport icon used in slicers.

---

#### `gold dim_workout_type`

**Purpose:** Types of **workouts** within a sport.

* **`id`** – workout type ID.
* **`type`** – name of the workout (e.g. **Run – Race**, **Run – Long Run** etc.).
* Used to distinguish **races**, **workouts**, **commutes**, etc. in analysis.

---

### Fact Tables

#### `gold fact_activities`

**Purpose:** Central **fact table** containing **one row per activity**.

Key columns:

* Identification & time:

  * **`id`**, **`name`**, **`description`**.
  * **`date`**, **`time`**, **`datetime`** (combined).
* Distance & time:

  * **`distance`** (meters), **`distance_km`** (calculated).
  * **`moving_time_s`**, **`elapsed_time_s`**, plus technical ***_td** columns.
  * **`moving_time_hms`**, **`elapsed_time_hms`** (formatted time).
* Elevation & calories:

  * **`total_elevation_gain`**, **`calories`**.
* Social & metadata:

  * **`achievement_count`**, **`kudos_count`**, **`comment_count`**, **`athlete_count`**, **`photo_count`**.
  * **`commute`**, **`manual`**, **`visibility`**.
* Pace & speed:

  * **`average_speed`**, **`max_speed`**, **`avg_pace_str`**, **`avg_pace_float`**, **`max_pace_str`**, **`max_pace_float`**.
  * **`avg_speed_pace_str`** – calculated, uses **sport type** to decide whether to show pace or speed.
* Cadence, power, heart rate:

  * **`average_cadence`**.
  * **`average_watts`**, **`max_watts`**, **`weighted_average_watts`** (mostly hidden).
  * **`has_heartrate`**, **`average_heartrate`**, **`max_heartrate`**.
* Effort:

  * **`relative_effort`**, **`pr_count`**, **`suffer_score`**.
* Foreign keys:

  * **`gear_id`**, **`location_id`**, **`sport_type_id`**, **`device_id`**, **`workout_type_id`**, **`map_id`**.

This table powers most **totals**, **averages**, **heatmaps** and **trend visualizations**.

---

#### `gold fact_best_efforts`

**Purpose:** **Best effort performances** (e.g. best 1 km, 5 km, 10 km) tied to activities.

* Time & ID:

  * **`id`**, **`date`**, **`time`**.
* Duration:

  * **`moving_time`**, **`elapsed_time`**, plus hidden ***_td** variants.
  * **`moving_time_hms`** – formatted time.
* Metadata:

  * **`rank`** – PR rank (1st, 2nd, 3rd etc.).
  * **`type`**, **`effort_type_id`**, **`activity_id`**.
* Calculations:

  * **`rank_icon`** – PR icon image URL.
  * **`pace`** – pace based on **effort type** and **time**.

---

#### `gold fact_kudos`

**Purpose:** Who gave **kudos** to which activity.

* **`first_name`**, **`last_name`**, **`full_name`** – kudo giver.
* **`activity_id`** – activity that received kudos.
* Used for **“Top Kudoers”** analysis and **social interactions**.

---

#### `gold fact_laps`

**Purpose:** **Laps / splits** within activities.

* Identification & time:

  * **`id`**, **`name`**, **`lap_index`**, **`split`**, **`activity_id`**.
  * **`date`**, **`time`**.
* Distance & time:

  * **`distance`**, **`moving_time`**, **`elapsed_time`**, plus ***_td** variants.
  * **`distance_km`**, **`moving_time_hms`**.
* Pace & speed:

  * **`average_speed`**, **`max_speed`**, **`avg_pace_str`**, **`avg_pace_float`**, **`max_pace_str`**, **`max_pace_float`**, **`avg_speed_pace_str`**, **`max_speed_pace_str`**.
* Elevation & HR:

  * **`total_elevation_gain`**, **`average_cadence`**, **`average_heartrate`**, **`max_heartrate`**.
* Derived positioning:

  * **`lap_start_distance`**, **`lap_end_distance`** – cumulative distance in the activity.
* Zone coloring:

  * **`pace_zones_colors`**, **`heartrate_zones_colors`** – hex colors based on **pace / HR zones** from `fact_zones`.

---

#### `gold fact_segments_efforts`

**Purpose:** Individual **efforts on segments**.

* Identification & time:

  * **`id`**, **`segment_id`**, **`activity_id`**.
  * **`date`**, **`time`**, **`datetime`** (combined).
* Duration & performance:

  * **`moving_time`**, **`elapsed_time`**, **`moving_time_hms`**.
  * **`average_cadence`**, **`average_heartrate`**, **`max_heartrate`**.
  * **`average_watts`**, **`device_watts`**.
* Rankings & visibility:

  * **`pr_rank`**, **`kom_rank`**, **`rank`**, **`visibility`**.
  * **`rank_icon`** – combined KOM/PR icon.
* Pace:

  * **`avg_speed_pace_str`** – uses segment distance and time to calculate pace.
* Helper measures:

  * **`Latest Segment Efforts`**, **`Latest Segment Efforts Colors`**, **`Top Effort Time`** – used to highlight recent and best segment efforts in charts.

---

#### `gold fact_zones`

**Purpose:** Time spent in **training zones** (pace or HR) per activity.

* **`activity_id`**, **`type`** (Pace/Heartrate), **`zone_number`**, **`zone_name`**.
* **`time`** (seconds), **`time_hms`**, **`min`**, **`max`** (zone boundaries).
* **`zone_name_short`** – short label (first two letters).
* Drives **time in zone** charts and **zone intensity** metrics.

---

#### `gold fact_maps_activities` & `gold fact_maps_segments`

**Purpose:** **Geometry** (points) for maps.

* Common columns:

  * **`map_id`**, **`point_id`**, **`lat`**, **`lng`**.
  * **`lat`** and **`lng`** are tagged as **Latitude/Longitude**.
* `fact_maps_activities`: map points for **activities**.
* `fact_maps_segments`: map points for **segments**.
* Used for **route visualizations**, **maps** and **top segment map**.

---

### Measures & Helper Tables

#### `Measures Table`

**Purpose:** Central table holding **all DAX measures**, no physical rows.

Key measure groups (simplified):

* **Totals (All)**:

  * **`Total Distance (All)`**, **`Total Elevation Gain (All)`**, **`Total Moving Time (All)`**, **`Total Days Active (All)`**, **`Total Calories (All)`**.
* **Totals (Current filters)**:

  * **`Total Time (Seconds/Hours)`**, **`Total Distance (Meters/Kilometers)`**, **`Total Effort`**, **`Total Activities`**, **`Total Time Zone (Seconds/Hours)`**, etc.
* **Readable summaries**:

  * **`Total Distance Summary`**, **`Total Moving Time Summary`**, **`Total Calories Summary`**, **`Total Days Active Summary`** – human-friendly comparisons (e.g. distance ≈ trips Wrocław–Paris).
* **Weekly summary**:

  * **`Weekly Distance (Meters/Kilometers)`**, **`Weekly Time (Seconds/Hours)`**, **`Weekly Time/Distance`**, **`Weekly Time/Distance Str`**, titles.
* **Monthly/Yearly heatmaps & averages**:

  * **`Total X Monthly Heatmap`**, **`Avg Total X Monthly`**, **`Avg Total X Daily`**, with label & title measures.
* **Gear metrics**:

  * **`Gear Avg Pace/Speed`**, **`Running Total Distance`**, **`Races Completed`**, **gear button icons**.
* **Relative effort & intensity**:

  * **`Total Relative Effort`**, **`3-Weeks Relative Effort Avg`** and low/high range, **week-level color & labels**.
  * **`Daily Effort Level`**, **filters for sports/years with effort**.
* **Training zones time intelligence**:

  * **`Total Time Zone 7D / 1M / 3M / 6M / 1Y / YTD`**, plus % in Zone 2, previous period, differences and date labels.
* **Yearly summary & comparisons**:

  * **`Total Monthly - Chart`**, **`Total Monthly - Chart YTD`**, **`Total Monthly - Table Curr/Prev Year`**, **Diff measures**, plus titles.
* **Rewind / storytelling measures**:

  * **`Yearly Time/Distance`**, **`Days Active`**, **`Days Active %`**, **`Longest Streak`**, **`Top Day of Week`**, **`Top Part Of Day`**, **`Top Segment`**, **`Top Kudoers` helpers**, **`Location Rank`**, etc.
* **User info & icons**:

  * **`User Name`**, **`User Profile Picture`**, **`Icon KOM`**, **`Icon Local Legend`**, etc.
* **Pop-up texts**:

  * **`Local Legend Info`**, **`Segment Info`**, **`Relative Effort Info`** – explanatory texts for tooltips/popups.
* **Activity-level helpers**:

  * **`Activity Gear Name`**, **`Activity Device Name`**, **`Activity Workout Name`**, **`Activity Location`**.

This table is **measure-only** and drives the majority of **KPIs, cards, tooltips, heatmaps and advanced visuals**.

---

#### `SummaryType`

**Purpose:** Small helper table used as **summary-type slicer**.

* Columns:

  * **`SummaryType`** – display label (**Time**, **Distance**, **Effort**, **Count**).
  * **`SummaryType Fields`** – internal reference to the underlying numeric column.
  * **`SummaryType Order`** – numeric order, used for sorting and SWITCH logic.
* Populated as a **calculated table** with four rows:

  * Time → `moving_time_s`
  * Distance → `distance`
  * Effort → `relative_effort`
  * Count → `id`

This drives dynamic selection of **which metric** to show in many measures (heatmaps, yearly totals, etc.).

---

#### `Training Zones Time Intelligence`

**Purpose:** Helper table for **training zones time-intelligence slicer**.

* Columns:

  * **`Training Zones Time Intelligence`** – label (e.g. **Total Time Zone 7D**, **1M**, **3M**, **6M**, **YTD**, **1Y**).
  * **`Training Zones Time Intelligence Fields`** – linked to specific measures.
  * **`Training Zones Time Intelligence Order`** – numeric order.
* Used to drive SWITCH logic in **time-zone measures** (e.g. total time in zone for the last 7 days vs last month).

