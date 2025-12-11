# 🏃‍♂️ Strava Analytics - Python + PostgreSQL + Power BI Project

![Dashboard demo](docs/gifs/Demo_Gif.gif)

📎 [**View Interactive Dashboard**](https://app.powerbi.com/view?r=eyJrIjoiNTdiMWRkOGYtNGE0Ny00YmI5LWJiYzAtYWYxZGQ2MmFmMmM0IiwidCI6IjY0YmU5OWY5LTI2N2MtNDIxMS1iMDlhLTQ0YmZlNjYyMzY0MCJ9&pageName=b48096df7ec0b97d2d07)

# README – under construction 🚧 

This README is under construction. More details coming soon. ✨


## 📊 Power BI Reporting & Analytics

This project includes an end-to-end **Power BI report** built on top of the **Gold layer** in PostgreSQL, turning raw Strava exports into an interactive **training analytics app**. The model follows a clean **star schema** with a rich **DAX layer**, **helper tables** and **user-defined functions (UDFs)** that power **dynamic slicers**, **metric toggles**, **time-intelligence** and storytelling features such as **yearly rewinds**, **training load** and **segment / Local Legend** analysis. Custom **UX patterns** (collapsible navigation, metric/period toggles, pop-ups, Deneb visuals) make the report feel closer to a modern **web dashboard** than a standard Power BI file.


<details>
<summary>Data Model Overview</summary>

## Data Model Overview

The Power BI model follows a **star schema** centered on the **`gold fact_activities`** table, which stores individual **Strava activities** (runs, rides, rides, walks, etc.). This **core fact table** is linked to a set of **dimension tables** for **date** (**`gold dim_calendar`**), **time of day** (**`gold dim_time`**), **sport type** (**`gold dim_sport_type`**), **workout type** (**`gold dim_workout_type`**), **gear** (**`gold dim_gear`**), **location** (**`gold dim_location`**), **device** (**`gold dim_device`**) and **segments / effort types** (**`gold dim_segment`**, **`gold dim_effort_type`**).

Additional **fact tables** – **`gold fact_best_efforts`**, **`gold fact_laps`**, **`gold fact_segments_efforts`**, **`gold fact_zones`**, **`gold fact_kudos`**, and map tables (**`gold fact_maps_activities`**, **`gold fact_maps_segments`**) – capture **best performances**, **laps/splits**, **segment efforts**, **time in training zones**, **social interactions** and **GPS geometry** for routes and segments.

All KPIs and advanced logic are implemented in a **measure-only table** (**`Measures Table`**) supported by small **helper tables** (**`SummaryType`**, **`Training Zones Time Intelligence`**) that drive **dynamic slicers**, **metric switching** and **time-intelligence** scenarios used across the dashboards.

➡️ For a detailed description of data model, see the dedicated **Data Model** documentation [here](docs/power_bi_data_model.md).

</details>

<details>
<summary>Power BI Dashboard Overview</summary>

## Power BI Dashboard Overview

The **Power BI report** sits on top of the **Gold** layer in PostgreSQL and gives a complete view of my **Strava training history**.

The pages are organized around key questions a Strava power user might ask:

- **Home** – high-level **summary of the last weeks** and lifetime KPIs: total distance, time, calories and active days, plus weekly distance and **sport-type breakdown**.  
- **All Activities** – interactive **activity log** for all sports with rich filtering (date, sport, workout type, gear, location) and detailed metrics for each workout.  
- **Running Activities** – focused view on **running-only data**, with filters for workout type, gear, location and device, and **drillthrough** to detailed activity pages.  
- **Gear** – analytics for **shoes and bikes**: usage over time, total distance, pace and status (active/retired), helping to track wear and gear lifecycle.  
- **Local Legends & Segments** – tools for **segment-based analysis** and **Local Legend** chasing: segments where I am close to the title, my current titles and drillthrough views for segment details and effort history.  
- **Training Intensity** – view of **effort and heart-rate zones**: time in each zone, weekly effort vs rolling average and recent high-effort activities.  
- **Yearly Summary** – long-term **year-over-year comparison** of time, distance, activity count and effort, showing stronger and weaker seasons.  
- **Period Summary** – analysis of **when** I train: heatmaps by year/month and by day of week/day part, plus average monthly and daily training time.  
- **Rewind** – a **“year in review”** experience comparing two years side by side: total time, top sports, days active, longest streaks and top locations/kudoers.

➡️ A detailed page-by-page description can be found here [here](docs/power_bi_dashboard.md).
</details>

<details>
<summary>Custom Power BI Elements</summary>

## Custom Power BI Elements

To make the report feel more like a **web app** than a standard Power BI report, I implemented several custom UX patterns using only **native features** (bookmarks, buttons, field parameters, DAX) plus one **Deneb** visual:

- **Collapsible navigation menu** – icon-only sidebar that can expand into a labelled menu via **bookmarks**, replacing default page tabs.  
- **Metric toggles with field parameters** – tab-like buttons that switch charts between **Time / Distance / Count / Effort** without duplicating visuals.  
- **Period toggles for Training Intensity** – pre-defined time windows (**7D, 1M, 3M, 6M, YTD, 1Y**) controlled by bookmarks and measures, mimicking training load dashboards from sports apps.  
- **Informational pop-ups for Strava concepts** – modal-style pop-ups opened from **help buttons** next to key visuals, explaining terms like **Segments**, **Local Legend** and **Relative Effort** using dedicated pages and bookmarks.  
- **Custom Deneb lap visuals** – Strava-like charts with **variable-width bars** that encode lap distance, pace and heart-rate zones for detailed workout analysis.


➡️ Detailed descriptions, screenshots and Vega-Lite specs are available [here](docs/power_bi_custom_elements.md).

</details>
<details>
<summary>DAX Measures</summary>

## DAX Measures

The report uses a **rich DAX layer**, but it is built almost entirely on **standard DAX functions** such as **`SUM`**, **`AVERAGE`**, **`DISTINCTCOUNT`**, **`DIVIDE`**, **`CALCULATE`**, **`FILTER`**, **`DATESINPERIOD`**, **`SELECTEDVALUE`**, **`SWITCH`**, **`IF`**, **`FORMAT`** and **`LOOKUPVALUE`**.

At a high level, the measures fall into a few main groups:

* **Core totals & averages** – standard aggregations for **distance**, **time**, **Relative Effort**, **calories**, **activities** and **days active**, with both **numeric** and **formatted text** variants for cards and tooltips.
* **Time intelligence & comparisons** – rolling windows and period logic (7D / 1M / 3M / 6M / YTD / 1Y) to compare **time**, **distance**, **effort** and **time in zones** across weeks, months and years.
* **Training load & time-in-zone** – measures that summarise **Relative Effort**, derive a **3-week training baseline**, classify each week into **intensity bands**, and track **Zone 2 share** over configurable time windows.
* **Field-parameter style toggles** – a single set of visuals can switch between **Time / Distance / Effort / Activities** using helper tables and `SELECTEDVALUE` + `SWITCH`, with fully dynamic **titles**, **labels** and **heatmaps**.
* **Rewind & storytelling metrics** – “year in review” measures for **longest streak**, **top day of week**, **top part of day**, **location rank**, and **high-level summaries** (e.g. distance ≈ **number of Wroclaw–Paris trips**).
* **Segments, KOM & Local Legend** – logic to count **segment efforts in the last 90 days**, compute **efforts to Local Legend**, highlight **PR attempts** and show **top segment** details.
* **Gear & activity metadata** – lookups that translate IDs into friendly **device**, **gear**, **workout type** and **location** names, plus gear-level stats such as **average pace/speed** and **cumulative distance**.
* **UX helpers** – icon/image measures, explanatory text blocks and empty-state placeholders that make the report feel more like an **interactive training app** than a static dashboard.

➡️ For a detailed description of individual measures and patterns, see the dedicated **DAX Measures** documentation [here](docs/power_bi_measures.md).
</details>

<details>
<summary>DAX user-defined functions (UDFs)</summary>

## DAX user-defined functions (UDFs)

The project makes extensive use of **Power BI DAX user-defined functions** to keep the model **modular**, **reusable** and **easy to maintain**. Instead of repeating complex logic across multiple measures, I encapsulated it into parameterised functions and reused them throughout the report.

At a high level, these UDFs cover:

* **Aggregations over time** – functions that calculate **average totals per active day/month** and drive **daily/monthly heatmaps** for activities, distance, time and effort.
* **Formatting & units** – helpers to convert **seconds to readable durations**, **meters to km text**, parse **result strings to seconds**, and compute **pace/speed** depending on the sport (min/km, min/100m, min/500m, km/h).
* **UI & icons** – functions that return **icon URLs** for **sports**, **gear types/status**, **PR/segment ranks**, and **color codes** for **pace/heart-rate zones**, used in buttons, cards and heatmap visuals.
* **Training load & zones** – time-intelligence helpers that calculate **time in Zone 2** over rolling periods (7D, 1M, 3M, 6M, YTD, 1Y) and compare it with **previous periods**.
* **Calendar & streak logic** – logic to compute the **longest streak of consecutive active days** together with the **date range** of that streak.
* **Yearly summary engines** – two central functions (`YearlyTotals` and `YearlyRunningTotals`) that return **current**, **previous**, **difference** and **YTD** values for **time, distance, effort and activities**, powering the **Yearly Summary** and **Running Rewind** pages.

Thanks to these UDFs, most “final” measures in the report become thin wrappers around **shared building blocks**, which improves **readability**, **consistency** and makes it much easier to **extend** the model with new metrics in the future.

➡️ For a detailed description of individual functions, see the dedicated **DAX user-defined functions (UDFs)** documentation [here](docs/power_bi_functions.md).
</details>
