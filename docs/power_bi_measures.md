## DAX Measures Overview

This project uses a **large DAX layer**, but most measures are built from a **reusable set of standard DAX functions**.
Below is a summary of the main **function groups** and how they are used across the model.

---

### 1. Aggregations & basic math

Core aggregation and numeric functions:

* **`SUM`**, **`AVERAGE`**, **`DISTINCTCOUNT`**, **`DISTINCT`**
  Used for all base totals and averages, for example:

  * **Total Distance (Meters)**, **Total Time (Seconds)**, **Total Effort**
  * **Avg Distance (Meters)**, **Avg Time (Seconds)**
  * **Days Active**, **Total Activities**, **Races Completed**

* **`DIVIDE`**
  Safe division used for unit conversions and ratios, e.g.:

  * **Total Distance (Kilometers)**, **Total Time (Hours)**
  * **3-Weeks Relative Effort Avg**
  * Daily / monthly average *Time / Distance / Effort* measures

* **`COALESCE`**
  Ensures visuals show **0 instead of BLANK**, e.g.:

  * **Total Effort**, **Days Active**, **Sport Active Last 12 Weeks**
  * **Avg Total Time Daily**, **Avg Total Effort Daily**, etc.

* **`FORMAT`**
  Builds user-friendly KPI labels such as:

  * **Total Distance Summary**, **Total Moving Time Summary**
  * **Total Calories Summary**, **Total Days Active Summary**
  * Formatted pace/speed strings in **Gear Avg Pace/Speed**

* **`INT`**, **`MOD`**, **`UNICHAR`**
  Used for:

  * Splitting seconds into minutes/seconds for pace strings
  * Building arrows and other symbols in comparison labels
  * Converting large second totals into full hours

---

### 2. Time intelligence & date logic

Time-window and rolling-period logic is central to the model:

* **`MAX`**, **`MIN`** (on dates)
  Pick current or boundary dates, for example:

  * Latest date in **Total Time Zone Time Intelligence – Dates**
  * Current context date in **Calculate Efforts Last 90 Days**
  * Current week or year boundaries on *Yearly* and *Weekly* views

* **`DATESINPERIOD`**
  Creates rolling windows such as **7D, 1M, 3M, 6M, 1Y, YTD** and custom 21/90-day ranges:

  * **3-Weeks Relative Effort Avg**
  * **Calculate Efforts Last 90 Days**
  * All **Time Zone** window measures (7D / 1M / 3M / 6M / 1Y / YTD)

* **`DATEDIFF`**
  Used to calculate the number of days between two dates, e.g. to:

  * Adjust the 90-day window for segments created recently in
    **Calculate Efforts Last 90 Days**

---

### 3. Context transition & filter shaping

Most advanced measures rely on **context manipulation**:

* **`CALCULATE`**, **`CALCULATETABLE`**
  Core engine for recalculating expressions in a modified filter context, e.g.:

  * **3-Weeks Relative Effort Avg** – recompute effort over last 21 days
  * **Weekly Distance (Meters)**, **Weekly Time (Seconds)** – restrict to a single week
  * **Total Distance (All)**, **Total Moving Time (All)** – remove report filters
  * Segment-based logic in **Calculate Efforts Last 90 Days**

* **`FILTER`**
  Defines row subsets inside a calculation, e.g.:

  * **Running Total Distance** – all dates ≤ current date
  * **Location Rank**, **Top Segment**, **Sport Active Last 12 Weeks**
  * **Top Day of Week**, **Top Part of Day**

* **`ALL`**, **`ALLSELECTED`**, **`ALLEXCEPT`**
  Precisely control which filters are removed or preserved:

  * **Total Distance (All)**, **Total Effort (All)** – full-history totals via `ALL`
  * **Running Total Distance** – `ALLSELECTED` to respect report filters but ignore the current row’s date
  * **Sports with Relative Effort**, **Years with Relative Effort** – `ALLEXCEPT` to keep only sport/year context

* **`KEEPFILTERS`**
  Keeps existing filters when adding new conditions, e.g.:

  * **Activity Device Name**, **Activity Gear Name**, **Activity Location**, **Activity Workout Name**
  * **Calculate Efforts Last 90 Days** – respect current segment selection

* **`SUMMARIZE`**, **`ADDCOLUMNS`**
  Build intermediate aggregated tables for ranking and “top” logic:

  * **Location Rank** – aggregate by country/city and compute ranking
  * **Top Segment / Top Segment – Efforts / Top Segment – Map**
  * **Top Day of Week**, **Top Part of Day**

* **`TOPN`**, **`ROWNUMBER`**
  Implement “top N” and ranking patterns:

  * **Top Segment**, **Top Segment – Efforts**, **Top Segment – Map**
  * **Top Effort Time**, **Latest Segment Efforts**
  * **Location Rank** – uses `ROWNUMBER` over the summarized table

---

### 4. Conditional logic & switching

Routing logic, thresholds and metric toggles:

* **`VAR` / `RETURN`**
  Used throughout for **step-by-step logic**, e.g.:

  * **3-Weeks Relative Effort Avg**
  * **Calculate Efforts Last 90 Days**
  * **Weekly Effort Text**, **Total Time Zone Time Intelligence – % Zone 2 Diff**
  * All KPI summary labels (e.g. **Total Distance Summary**)

* **`SWITCH(TRUE())`**
  Multi-branch logic based on thresholds or selected metric:

  * **Weekly Effort Color**, **Weekly Effort Text**, **Daily Effort Level** – classify training load
  * **Avg Total X Daily**, **Avg Total X Monthly**, **Total X Monthly Heatmap** – metric toggles (Time / Distance / Effort / Count)
  * **Total Time Zone Time Intelligence – % Zone 2** and related measures – choose the correct time window

* **`IF`**
  Simple binary conditions for flags and UX states:

  * **Is Week Filter Active**, **Running Activities**, **Sport Active Last 12 Weeks**
  * **Sports with Relative Effort**, **Show Top Segment Id?**
  * **Efforts To Local Legend**, **Weekly Effort Transparency Background/Text**

* **`SELECTEDVALUE`**
  Reads the **current user selection** and drives dynamic behavior:

  * Metric toggles via the **SummaryType** table (Time / Distance / Effort / Count)
  * Period toggles via the **Training Zones Time Intelligence** table (7D / 1M / 3M / 6M / YTD / 1Y)
  * **Gear Avg Pace/Speed**, **Sport Icon Slicer**
  * **Activity Device/Gear/Workout Name**, **Activity Location**
  * **Yearly Summary – Is Relative Effort Selected?**

---

### 5. Lookup & dimension enrichment

Transforming internal IDs into readable labels:

* **`LOOKUPVALUE`**
  Used to fetch descriptive attributes from dimension tables:

  * **Activity Device Name** – fetch device name from the device dimension
  * **Activity Gear Name** – fetch gear name from the gear dimension
  * **Activity Workout Name** – fetch workout type
  * **Activity Location** – combine city and country from the location dimension

---

### 6. Ranking, streaks & “top X” patterns

More advanced analytics based on the functions above:

* **Streaks & activeness**

  * Use **`DISTINCTCOUNT`**, **`CALCULATE`** and date filters over the calendar to derive:

    * **Days Active**, **Days Active %**, **Days Total**
    * Longest streak metrics over consecutive active days

* **Location & segment ranking**

  * Combine **`SUMMARIZE`**, **`ADDCOLUMNS`**, **`TOPN`**, **`ROWNUMBER`** and **`CALCULATE`** to:

    * Rank locations in **Location Rank**
    * Identify **Top Segment**, its **Efforts**, and **Map**
    * Find the best time in **Top Effort Time**

---

### 7. Time, distance & effort toggles

Field-parameter style behavior built entirely on standard functions:

* **Metric switching (Time / Distance / Effort / Count)**

  * Driven by **`SELECTEDVALUE`** on a helper table and combined with:

    * `SWITCH(TRUE())` to return the right base measure
    * `FORMAT` + arithmetic to produce the correct numeric or text label
  * Used in:

    * **Avg Total X Daily / Monthly** (+ labels and titles)
    * **Total Monthly – Chart / YTD / Table ...**
    * **Total Monthly Rolling Totals – Title**, **Total Monthly Totals – Title**
    * **Total X Monthly Heatmap** (+ labels and title)

* **Training zone time windows (7D / 1M / 3M / 6M / YTD / 1Y)**

  * Use **`SELECTEDVALUE`**, **`DATESINPERIOD`**, `MAX`, `MIN`, `CALCULATE` to:

    * Compute **Total Time Zone** for each window
    * Compare **% Zone 2** to previous periods
    * Generate human-readable date ranges with `FORMAT`


## Measure Groups

➡️ For a detailed description of individual measures, see the dedicated **DAX Measures Details** documentation [here](power_bi_measures_details.md).


### Weekly Effort & Training Load

| **Group**                            | **Key measures**                                                                                                | **Description**                                                                                                                                                                                                                                                                        |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **3-week Relative Effort baseline**  | `3-Weeks Relative Effort Avg`, `3-Weeks Relative Effort Avg Low Val`, `3-Weeks Relative Effort Avg High Val`    | Compute the **average Relative Effort over the previous 3 weeks** (21 days). The low and high values define a **65–135% range** used as a “healthy weekly load” band.                                                                                                                  |
| **3-week range heights for visuals** | `3-Weeks Relative Effort Avg - Height High`, `3-Weeks Relative Effort Avg - Height Total`                       | Helper measures for **stacked column visuals** – they calculate the height of the 3-week range band and the difference between **current week** and the **upper bound** of that range.                                                                                                 |
| **Current week Relative Effort**     | `Total Relative Effort`, `Total Relative Effort Card`                                                           | Sum of **Relative Effort** in the current filter context. The *Card* version only returns a value when **exactly one week** is selected (otherwise blank) to avoid misleading totals.                                                                                                  |
| **Weekly state flags & styling**     | `Is Week Filter Active`, `Weekly Effort Color`, `Weekly Effort Text`                                            | `Is Week Filter Active` checks whether a **single week** is selected. `Weekly Effort Color` maps current effort vs range to **colors**, while `Weekly Effort Text` produces labels like **“Below Weekly Range”, “Consistent Training”, “Steady Progress”, “Well Above Weekly Range”**. |
| **Empty state UX**                   | `Weekly Effort Placeholder Message`, `Weekly Effort Transparency Background`, `Weekly Effort Transparency Text` | Drive the **placeholder message** (“Select a week on the chart above…”) and **background/text colors** when no specific week is selected – used for a clean empty state experience.                                                                                                    |

---

### Activity Metadata & User Information

| **Group**                    | **Key measures**                                                                                                                | **Description**                                                                                                                                                                                |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Activity metadata**        | `Activity Device Name`, `Activity Gear Name`, `Activity Location`, `Activity Workout Name`                                      | For the **selected activity**, these measures resolve IDs into **friendly labels**: device name, gear name, “City – Country” location string, and workout type (e.g. **“Run – Race”**).        |
| **User identity**            | `User Name`, `User Profile Picture`                                                                                             | Store static information about the **athlete name** and **profile picture URL**, used on the Home page and other header sections.                                                              |
| **Icons & images**           | `Best Effort 1st`, `Icon KOM`, `Icon Local Legend`, `Button Icon - Gear Status`, `Button Icon - Gear Type`, `Sport Icon Slicer` | Provide **SVG/data image URLs** or image URLs for medals, KOM, Local Legend, gear status/type and sport icons. Used in **buttons, slicers and cards** to give the report a more app-like look. |
| **Text popups / info boxes** | `Local Legend Info`, `Relative Effort Info`, `Segment Info`                                                                     | Predefined **rich text snippets** explaining Strava concepts such as **Local Legend**, **Relative Effort** and **Segments**, displayed in **info tooltips or popups**.                         |
| **Technical placeholders**   | `Dummy Placeholder`, `Dummy Title`, `Elevation Profile Placeholder`                                                             | Empty string measures used as **generic placeholders** for visuals or titles where dynamic content may be injected later.                                                                      |

---

### Global Totals & Storytelling Summaries

| **Group**                      | **Key measures**                                                                                                                                                                                                                                                                                                                             | **Description**                                                                                                                                                                                                                                 |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Global totals (all-time)**   | `Total Distance (All)`, `Total Moving Time (All)`, `Total Elevation Gain (All)`, `Total Calories (All)`, `Total Days Active (All)`                                                                                                                                                                                                           | Compute **lifetime totals** over all activities (using `ALL(...)`): distance, time, elevation gain, calories and number of days with at least one activity.                                                                                     |
| **Global textual summaries**   | `Total Distance Summary`, `Total Moving Time Summary`, `Total Calories Summary`, `Total Days Active Summary`                                                                                                                                                                                                                                 | Turn numeric totals into **storytelling strings**, e.g. kilometers → **“Wroclaw–Paris trips”**, seconds of training → **“months of full-time work”**, calories → **Big Macs equivalent**, days active → **years of daily activity**.            |
| **Standard totals in context** | `Total Distance (Meters)`, `Total Distance (Kilometers)`, `Total Distance (Kilometers) - Str`, `Total Time (Seconds)`, `Total Time (Hours)`, `Total Time (Hours) - Str`, `Total Time (Hours) - Str Hours Label`, `Total Effort`, `Total Activities`, `Total Time Zone (Seconds)`, `Total Time Zone (Hours)`, `Total Time Zone (Hours) - Str` | Core measures returning **distance**, **time**, **Relative Effort**, **activity count** and **time-in-zone** in the current filter context. Many of them have both **numeric** and **formatted string** versions for use in cards and tooltips. |
| **Average per activity**       | `Avg Distance (Meters)`, `Avg Distance (Kilometers)`, `Avg Distance (Kilometers) - Str`, `Avg Time (Seconds)`, `Avg Time (Hours) - Str`                                                                                                                                                                                                      | Compute **average distance** and **average time** per activity, in both numeric and friendly text formats.                                                                                                                                      |

---

### Averages, Heatmaps & Field Parameter `SummaryType`

| **Group**                             | **Key measures**                                                                                                                                                                                                                                                                                                                                       | **Description**                                                                                                                                                                                                                                              |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Daily averages**                    | `Avg Total Time Daily`, `Avg Total Distance Daily`, `Avg Total Effort Daily`, `Avg Total X Daily`                                                                                                                                                                                                                                                      | Calculate **average daily time, distance and Relative Effort**. `Avg Total X Daily` uses **`SummaryType[SummaryType Order]`** to return the correct metric based on the current field parameter selection.                                                   |
| **Daily labels & titles**             | `Avg Total X Daily - Labels`, `Avg Total X Daily - Title`                                                                                                                                                                                                                                                                                              | Provide **text labels** (time formatted as duration, distance as km, effort / activity count as numbers) and **dynamic chart titles** like *“Average Daily Distance by Day of Week/Day Part”* depending on the selected metric.                              |
| **Monthly averages**                  | `Avg Total Time Monthly`, `Avg Total Distance Monthly`, `Avg Total Effort Monthly`, `Avg Total Activities Monthly`, `Avg Total X Monthly`                                                                                                                                                                                                              | Compute **average monthly time, distance, effort and number of activities**. Again, `Avg Total X Monthly` abstracts over the current metric selected in `SummaryType`.                                                                                       |
| **Monthly labels & titles**           | `Avg Total X Monthly - Labels`, `Avg Total X Monthly - Title`                                                                                                                                                                                                                                                                                          | Text labels (formatted duration/distance, etc.) and titles like *“Average Monthly Distance by Year/Month”* for the currently selected metric.                                                                                                                |
| **Monthly totals for Yearly Summary** | `Total X Monthly Heatmap`, `Total X Monthly Heatmap - Labels`, `Total X Monthly Heatmap - Title`, `Total Activities Monthly Heatmap`, `Total Distance Monthly Heatmap`, `Total Time Monthly Heatmap`, `Total Effort Monthly Heatmap`                                                                                                                   | Supply values and titles for **year × month heatmaps** and tables used on **Yearly Summary / Period Summary** pages; they adapt dynamically based on `SummaryType`.                                                                                          |
| **Yearly totals & comparisons**       | `Total Monthly - Chart`, `Total Monthly - Chart YTD`, `Total Monthly - Chart YTD Tooltip`, `Total Monthly - Table Curr Year`, `Total Monthly - Table Prev Year`, `Total Monthly - Table Diff Val`, `Total Monthly - Table Diff Str`, `Total Monthly Rolling Totals - Title`, `Total Monthly Totals - Title`, `Total Monthly Yearly Comparison - Title` | Encapsulate logic for **current vs previous year**, **YTD**, **absolute and formatted diffs** and **rolling yearly totals**. Also return **dynamic chart/table titles** that adapt to the currently selected metric (Time / Distance / Effort / Activities). |
| **Daily heatmaps**                    | `Avg Total Distance Daily Heatmap`, `Avg Total Effort Daily Heatmap`, `Avg Total Time Daily Heatmap`, `Avg Total X Daily Heatmap`, `Avg Total X Daily Heatmap - Title`                                                                                                                                                                                 | Dedicated measures for **day-of-week × day-part heatmaps**, with values and titles dependent on the chosen metric in `SummaryType`.                                                                                                                          |

---

### Time-in-Zone & Training Intensity (Period Selector)

| **Group**                        | **Key measures**                                                                                                                                                | **Description**                                                                                                                                                                                                        |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Raw time-in-zone totals**      | `Total Time Zone (Seconds)`, `Total Time Zone (Hours)`, `Total Time Zone (Hours) - Str`                                                                         | Aggregate **time spent in all heart-rate zones** (from `gold fact_zones`) and expose it in **seconds**, **hours** and **formatted duration text**.                                                                     |
| **Time-in-zone by period**       | `Total Time Zone 7D`, `Total Time Zone 1M`, `Total Time Zone 3M`, `Total Time Zone 6M`, `Total Time Zone YTD`, `Total Time Zone 1Y`                             | Wrap a helper function `TotalTimeZoneTimeIntelligence(...)` to produce time-in-zone totals for predefined windows: **7 days, 1/3/6 months, YTD, 1 year**.                                                              |
| **Dynamic % Zone 2**             | `Total Time Zone Time Intelligence - % Zone 2`, `Total Time Zone Time Intelligence - Prev Period % Zone 2`, `Total Time Zone Time Intelligence - % Zone 2 Diff` | Drive the **Training Intensity** KPI for **Zone 2 Time**. They compute the **current % Zone 2**, the **previous period %**, and a **human-readable diff string** with arrows (↑/↓).                                    |
| **Dynamic period label & total** | `Total Time Zone Time Intelligence - Dates`, `Total Time Zone Time Intelligence (Hours) - Str`                                                                  | Return a **date range label** (e.g. *“01 January 2024 – 31 March 2024”*) and **formatted total hours** for the current period selected via the **period toggle** (`Training Zones Time Intelligence` field parameter). |

---

### Weekly Summary – Time & Distance per Sport

| **Group**                          | **Key measures**                                                                                                                                     | **Description**                                                                                                                                                                                                                                                  |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Weekly distance/time values**    | `Weekly Distance (Meters)`, `Weekly Distance (Kilometers)`, `Weekly Distance Str`, `Weekly Time (Seconds)`, `Weekly Time (Hours)`, `Weekly Time Str` | Compute **total weekly distance and time** for the week specified by `week_start_date`. Provide both metric values and **formatted strings** for cards/labels.                                                                                                   |
| **Dynamic weekly metric by sport** | `Weekly Time/Distance`, `Weekly Time/Distance Str`, `Weekly Time/Distance Title`                                                                     | Based on `DefineSportSummaryType(SelectedSport)`, these measures choose whether to show **distance** (e.g. for cycling/running) or **time** (e.g. for some sports). They also provide a **matching title** (*“Total Weekly Distance”* vs *“Total Weekly Time”*). |

---

### Rewind & Year-in-Review Metrics

| **Group**                          | **Key measures**                                                                                                      | **Description**                                                                                                                                                                                                                  |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Days active & consistency**      | `Days Active`, `Days Total`, `Days Active %`, `Days Active % - Fill to 100`                                           | Measure **how many days with activities** exist in the current context, the **total days**, and the **share of active days**. The “Fill to 100” version is used in **donut/ring charts** to show the remaining inactive portion. |
| **Longest streak**                 | `Longest Streak - Value`, `Longest Streak - Dates`                                                                    | Wrap a reusable function `CalculateLongestStreak(...)` to compute **longest sequence of consecutive active days** and the **start/end dates** of that streak.                                                                    |
| **Top day & time of day**          | `Top Day of Week`, `Top Part Of Day`                                                                                  | Identify the **weekday** with the most activities and the **hour block** with the highest total training time (e.g. *“18 – 19”*). Used on **Rewind** pages to describe when the athlete trains most often.                       |
| **Year-level distance/time label** | `Yearly Time/Distance`                                                                                                | Based on sport type, returns either **total yearly time** or **total yearly distance**, formatted as text, to be used in Rewind headers.                                                                                         |
| **Relative Effort availability**   | `Years with Relative Effort`, `Sports with Relative Effort`, `Running Activites`                                      | Helper flags that mark which **years/sports** have **Relative Effort** or **running activities** available – useful for conditionally showing or hiding visuals.                                                                 |
| **Location ranking**               | `Location Rank`                                                                                                       | Ranks **city–country pairs** by total activities and total time, returning the **rank** of the current location within all locations – shown in **Top locations** tables.                                                        |
| **Top kudoers logic**              | `Top Kudoers - Label`, `Top Kudoers - Order`, `Top Kudoers - Max Val`, `Top Kudoers - Min Val`                        | Wrap `KudoerOrderAndLabel(...)` to generate **ordering**, **axis range** and **labels** for bar charts that show **friends who give the most kudos**.                                                                            |
| **Top segment & its visibility**   | `Top Segment`, `Top Segment - Efforts`, `Top Segment - Map`, `Show Top Segment Id?`, `Show Top Segment Map testetst?` | Determine the **segment with the most efforts**, its **effort count** and **map ID**, and provide boolean flags (0/1) used to conditionally show **top segment details and map** visuals.                                        |
| **Summary flags**                  | `Total Time (Hours) - Coalesce`, `Yearly Summary - Is Relative Effort Selected?`                                      | Utility measures for **Rewind** and **Yearly Summary** pages; ensure robust handling of blanks and indicate when **Relative Effort** is the currently selected metric.                                                           |

---

### Gear Analysis & Performance

| **Group**                                  | **Key measures**         | **Description**                                                                                                                                                                             |
| ------------------------------------------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Gear-specific pace/speed**               | `Gear Avg Pace/Speed`    | For the selected piece of gear, calculates either **average pace (/km)** for shoes or **average speed (km/h)** for bikes, returning a **ready-to-display string**.                          |
| **Races per gear**                         | `Races Completed`        | Counts how many activities with workout type **“Run – Race”** or **“Ride – Race”** have been completed – used to show how often a specific gear item was used in races.                     |
| **Cumulative distance per gear (running)** | `Running Total Distance` | Builds a **running total** of distance up to the current date and hides the **final maximum value**, so the line chart ends just before the total — mimicking Strava-like cumulative plots. |

---

### Segments, KOM & Local Legend Logic

| **Group**                     | **Key measures**                                                                             | **Description**                                                                                                                                                                                                            |
| ----------------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Local Legend maths**        | `Calculate Efforts Last 90 Days`, `Efforts To Local Legend`                                  | Count **distinct efforts on a segment over the last 90 days** (or since creation if younger than 90 days), and compute how many more efforts are needed to take the **Local Legend** title compared to the current holder. |
| **Latest efforts & coloring** | `Latest Segment Efforts`, `Latest Segment Efforts Colors`, `Top Effort Time`, `Selected KOM` | Control which **recent efforts** are included (Top 40 + PR), highlight **PR attempts** with a distinct color, and compute the **fastest segment time** plus **selected KOM time** for visual comparison.                   |
| **Static icons & labels**     | `Icon KOM`, `Icon Local Legend`                                                              | Provide image URLs for **KOM/QOM** and **Local Legend** icons displayed next to segment stats.                                                                                                                             |
| **Info popups**               | `Segment Info`, `Local Legend Info`                                                          | Predefined explanatory text for **Segment** and **Local Legend** mechanics, displayed in help tooltips/popups.                                                                                                             |

---

### Filters, Flags & Helper Logic

| **Group**                       | **Key measures**                                                                 | **Description**                                                                                                                                                                                                                          |
| ------------------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sport activity flags**        | `Sport Active Last 12 Weeks`, `Sports with Relative Effort`, `Running Activites` | Return **1/0 flags** that indicate whether a sport has any activity in the last 12 weeks, whether Relative Effort exists for that sport, and whether running activities exist at all – used to control **visibility of tiles/sections**. |
| **Daily Effort categorisation** | `Daily Effort Level`                                                             | Groups daily **Relative Effort** into discrete levels (0–4) used in **color scales / icons** for daily effort grids.                                                                                                                     |
| **Segment / map visibility**    | `Show Top Segment Id?`, `Show Top Segment Map testetst?`                         | Simple 0/1 measures used in filters to show/hide **top segment** and its **map** on Rewind pages.                                                                                                                                        |
| **Zones coloring helpers**      | `PaceZonesColors`, `HeartrateZonesColors`                                        | Wrap `SplitsColorLabels(...)` to assign **color labels** to laps based on **pace** or **heart rate**; used by **Deneb visuals** for Lap Pace and Lap Heart Rate charts.                                                                  |

