## DAX Measures Details


### `3-Weeks Relative Effort Avg`

```DAX
VAR PrevWeekStartDate = MAX('gold dim_calendar'[week_start_date]) - 1
VAR TotalEffort3Weeks =
    CALCULATE(
        [Total Relative Effort],
        DATESINPERIOD(
            'gold dim_calendar'[date],
            PrevWeekStartDate,
            -21,
            DAY
        )
    )
VAR AvgEffort3Weeks =
    DIVIDE(
        TotalEffort3Weeks,
        3
    )
RETURN
    AvgEffort3Weeks
```

**Description:** Calculates the **average Relative Effort over the previous 3 weeks (21 days)**, using the **week before** the current one as the end of the window. Used as a **training-load baseline** in weekly views.

---

### `3-Weeks Relative Effort Avg - Height High`

```DAX
[3-Weeks Relative Effort Avg High Val] - [3-Weeks Relative Effort Avg Low Val]
```

**Description:** Returns the **difference between the high and low threshold** of the 3-week effort range. Used as a **band height** in stacked column visuals.

---

### `3-Weeks Relative Effort Avg - Height Total`

```DAX
[Total Relative Effort] - [3-Weeks Relative Effort Avg High Val]
```

**Description:** Returns how much the **current week’s effort** is **above the upper bound** of the 3-week range. Used to show “**excess load**” on charts.

---

### `3-Weeks Relative Effort Avg High Val`

```DAX
[3-Weeks Relative Effort Avg] * 1.35
```

**Description:** Upper bound of the **healthy training range** (135% of 3-week average Relative Effort).

---

### `3-Weeks Relative Effort Avg Low Val`

```DAX
[3-Weeks Relative Effort Avg] * 0.65
```

**Description:** Lower bound of the **healthy training range** (65% of 3-week average Relative Effort).

---

### `Activity Device Name`

```DAX
VAR ActivityId = SELECTEDVALUE('gold fact_activities'[id])
VAR DeviceId =
    CALCULATE(
        SELECTEDVALUE('gold fact_activities'[device_id]),
        KEEPFILTERS('gold fact_activities'[id] = ActivityId)
    )
RETURN
    IF (
        NOT ISBLANK(DeviceId),
        LOOKUPVALUE(
            'gold dim_device'[device_name],
            'gold dim_device'[id], DeviceId
        )
    )
```

**Description:** For the **currently selected activity**, retrieves its **device_id** and looks up a **human-friendly device name** from `gold dim_device`.

---

### `Activity Gear Name`

```DAX
VAR ActivityId = SELECTEDVALUE('gold fact_activities'[id])
VAR GearId =
    CALCULATE(
        SELECTEDVALUE('gold fact_activities'[gear_id]),
        KEEPFILTERS('gold fact_activities'[id] = ActivityId)
    )
RETURN
    IF (
        NOT ISBLANK(GearId),
        LOOKUPVALUE(
            'gold dim_gear'[name],
            'gold dim_gear'[id], GearId
        )
    )
```

**Description:** For the **selected activity**, looks up the **gear name** (shoes, bike, etc.) from `gold dim_gear`.

---

### `Activity Location`

```DAX
VAR ActivityId = SELECTEDVALUE('gold fact_activities'[id])
VAR LocationId =
    CALCULATE(
        SELECTEDVALUE('gold fact_activities'[location_id]),
        KEEPFILTERS('gold fact_activities'[id] = ActivityId)
    )
VAR ActivityCountry =
    IF (
        NOT ISBLANK(LocationId),
        LOOKUPVALUE(
            'gold dim_location'[country],
            'gold dim_location'[id], LocationId
        )
    )
VAR ActivityCity =
    IF (
        NOT ISBLANK(LocationId),
        LOOKUPVALUE(
            'gold dim_location'[locality],
            'gold dim_location'[id], LocationId
        )
    )
RETURN
    ActivityCity & " - " & ActivityCountry
```

**Description:** Builds a **“City – Country” location string** for the selected activity, based on `location_id` and the `gold dim_location` table.

---

### `Activity Workout Name`

```DAX
VAR ActivityId = SELECTEDVALUE('gold fact_activities'[id])
VAR WorkoutId =
    CALCULATE(
        SELECTEDVALUE('gold fact_activities'[workout_type_id]),
        KEEPFILTERS('gold fact_activities'[id] = ActivityId)
    )
RETURN
    IF (
        NOT ISBLANK(WorkoutId),
        LOOKUPVALUE(
            'gold dim_workout_type'[type],
            'gold dim_workout_type'[id], WorkoutId
        )
    )
```

**Description:** For the selected activity, returns a **friendly workout type name** (e.g. **“Run – Race”**) from `gold dim_workout_type`.

---

### `Avg Distance (Kilometers)`

```DAX
DIVIDE(
    [Avg Distance (Meters)],
    1000
)
```

**Description:** Converts **average distance per activity** from **meters to kilometers**.

---

### `Avg Distance (Kilometers) - Str`

```DAX
MetersToDistanceText([Avg Distance (Meters)])
```

**Description:** Uses a helper formatter to convert the **average distance in meters** into a **readable distance string** (e.g. `"7.3 km"`).

---

### `Avg Distance (Meters)`

```DAX
AVERAGE(
    'gold fact_activities'[distance]
)
```

**Description:** Average **activity distance** in **meters** across the current filter context.

---

### `Avg Time (Hours) - Str`

```DAX
SecondsToDurationText([Avg Time (Seconds)])
```

**Description:** Converts the **average moving time in seconds** into a **formatted duration string** (“hh:mm:ss”).

---

### `Avg Time (Seconds)`

```DAX
AVERAGE(
    'gold fact_activities'[moving_time_s]
)
```

**Description:** Average **moving time per activity** in **seconds**.

---

### `Avg Total Activities Monthly`

```DAX
AvgTotalsMonthly([Total Activities])
```

**Description:** Uses a helper to compute **average number of activities per month**, based on `[Total Activities]`.

---

### `Avg Total Distance Daily`

```DAX
AvgTotalsDaily([Total Distance (Kilometers)])
```

**Description:** Average **daily distance** in kilometers, using a daily helper over `[Total Distance (Kilometers)]`.

---

### `Avg Total Distance Daily Heatmap`

```DAX
TotalsDailyHeatmap(AvgTotalsDaily([Total Distance (Kilometers)]))
```

**Description:** Prepares **daily average distance values** for **day-of-week/day-part heatmaps**.

---

### `Avg Total Distance Monthly`

```DAX
AvgTotalsMonthly([Total Distance (Kilometers)])
```

**Description:** Average **monthly distance** in kilometers.

---

### `Avg Total Effort Daily`

```DAX
AvgTotalsDaily([Total Effort])
```

**Description:** Average **daily Relative Effort**.

---

### `Avg Total Effort Daily Heatmap`

```DAX
TotalsDailyHeatmap(AvgTotalsDaily([Total Effort]))
```

**Description:** Heatmap-ready **daily Relative Effort** values.

---

### `Avg Total Effort Monthly`

```DAX
AvgTotalsMonthly([Total Effort])
```

**Description:** Average **monthly Relative Effort**.

---

### `Avg Total Time Daily`

```DAX
AvgTotalsDaily([Total Time (Hours)])
```

**Description:** Average **daily training time** in hours.

---

### `Avg Total Time Daily Heatmap`

```DAX
TotalsDailyHeatmap(AvgTotalsDaily([Total Time (Hours)]))
```

**Description:** Heatmap-ready **daily average time** values.

---

### `Avg Total Time Monthly`

```DAX
AvgTotalsMonthly([Total Time (Hours)])
```

**Description:** Average **monthly training time** in hours.

---

### `Avg Total X Daily`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, COALESCE([Avg Total Time Daily], 0),
        SelectedField = 1, COALESCE([Avg Total Distance Daily], 0),
        SelectedField = 2, COALESCE([Avg Total Effort Daily], 0),
        SelectedField = 3, COALESCE([Total Activities], 0),
        BLANK()
    )
```

**Description:** **Field-parameter pattern**: returns **daily average** of **Time / Distance / Effort / Activity count**, depending on the selected `SummaryType`.

---

### `Avg Total X Daily - Labels`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, SecondsToDurationText(COALESCE([Avg Total Time Daily] * 3600, 0)),
        SelectedField = 1, MetersToDistanceText(COALESCE([Avg Total Distance Daily] * 1000, 0)),
        SelectedField = 2, COALESCE([Avg Total Effort Daily], 0),
        SelectedField = 3, COALESCE([Total Activities], 0),
        BLANK()
    )
```

**Description:** Returns **formatted labels** for `Avg Total X Daily`: time as **duration**, distance as **km text**, effort and count as **numbers**.

---

### `Avg Total X Daily - Title`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, "Average Daily Time by Day of Week/Day Part",
        SelectedField = 1, "Average Daily Distance by Day of Week/Day Part",
        SelectedField = 2, "Average Daily Effort by Day of Week/Day Part",
        SelectedField = 3, "Total Number of Activities by Day of Week/Day Part",
        BLANK()
    )
```

**Description:** Dynamic **chart title** for daily averages, based on `SummaryType`.

---

### `Avg Total X Daily Heatmap`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, COALESCE([Avg Total Time Daily Heatmap], 0),
        SelectedField = 1, COALESCE([Avg Total Distance Daily Heatmap], 0),
        SelectedField = 2, COALESCE([Avg Total Effort Daily Heatmap], 0),
        SelectedField = 3, COALESCE([Total Activities], 0),
        BLANK()
    )
```

**Description:** Master measure for **daily heatmap values** (time, distance, effort, activities) selected via `SummaryType`.

---

### `Avg Total X Daily Heatmap - Title`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, "Average Daily Time by Day of Week and Day Part",
        SelectedField = 1, "Average Daily Distance by Day of Week and Day Part",
        SelectedField = 2, "Average Daily Effort by Day of Week and Day Part",
        SelectedField = 3, "Total Number of Activities by Day of Week and Day Part",
        BLANK()
    )
```

**Description:** Dynamic **title** for the **daily heatmap** visual.

---

### `Avg Total X Monthly`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, COALESCE([Avg Total Time Monthly], 0),
        SelectedField = 1, COALESCE([Avg Total Distance Monthly], 0),
        SelectedField = 2, COALESCE([Avg Total Effort Monthly], 0),
        SelectedField = 3, COALESCE([Avg Total Activities Monthly], 0),
        BLANK()
    )
```

**Description:** Master measure for **monthly averages** of time, distance, effort and activity count.

---

### `Avg Total X Monthly - Labels`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, SecondsToDurationText(COALESCE([Avg Total Time Monthly] * 3600, 0)),
        SelectedField = 1, MetersToDistanceText(COALESCE([Avg Total Distance Monthly] * 1000, 0)),
        SelectedField = 2, COALESCE([Avg Total Effort Monthly], 0),
        SelectedField = 3, COALESCE([Avg Total Activities Monthly], 0),
        BLANK()
    )
```

**Description:** **Formatted labels** for `Avg Total X Monthly`, similar pattern as for daily averages.

---

### `Avg Total X Monthly - Title`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, "Average Monthly Time by Year/Month",
        SelectedField = 1, "Average Monthly Distance by Year/Month",
        SelectedField = 2, "Average Monthly Effort by Year/Month",
        SelectedField = 3, "Average Monthly Number of Activities by Year/Month",
        BLANK()
    )
```

**Description:** Dynamic **title** for the **monthly average** chart.

---

### `Best Effort 1st`

```DAX
"data:image/svg+xml;utf8,<svg ...>...</svg>"
```

**Description:** Returns an **inline SVG icon** (1st place / medal). Used as an **Image URL** in visuals.

---

### `Button Icon - Gear Status`

```DAX
ButtonIconUrl(SELECTEDVALUE('gold dim_gear'[status]))
```

**Description:** Uses a helper to return an **icon URL** for the **current gear status** (e.g. active/retired).

---

### `Button Icon - Gear Type`

```DAX
ButtonIconUrl(SELECTEDVALUE('gold dim_gear'[gear_type]))
```

**Description:** Returns an **icon URL** representing the **gear type** (shoes, bike, etc.).

---

### `Calculate Efforts Last 90 Days`

```DAX
VAR CurrentDate =
    MAX('gold dim_calendar'[date])

VAR SelectedSegment =
    SELECTEDVALUE('gold dim_segment'[id])

VAR SegmentCreationDate =
    CALCULATE(
        SELECTEDVALUE('gold dim_segment'[created_date]),
        KEEPFILTERS('gold fact_segments_efforts'[segment_id] = SelectedSegment)
    )

VAR SegmentCreationDateDays =
    DATEDIFF(
        SegmentCreationDate,
        CurrentDate,
        DAY
    )
VAR EffortDays =
    IF (
        SegmentCreationDateDays < 90,
        -1 * SegmentCreationDateDays,
        -90
    )
RETURN
    COUNTROWS (
        CALCULATETABLE(
            DISTINCT('gold fact_segments_efforts'[datetime]),
            KEEPFILTERS('gold fact_segments_efforts'[segment_id] = SelectedSegment),
            KEEPFILTERS('gold fact_segments_efforts'[visibility] = "everyone"),
            DATESINPERIOD(
                'gold dim_calendar'[date],
                CurrentDate,
                EffortDays,
                DAY
            )
        )
    )
```

**Description:** Counts **distinct segment efforts** for the **selected segment** over the **last 90 days**, or since **segment creation** if it’s younger than 90 days. Used for **Local Legend** logic.

---

### `Daily Effort Level`

```DAX
VAR Effort =
    COALESCE(
        SUM('gold fact_activities'[relative_effort]),
        0
    )
VAR EffortLevel =
    SWITCH(
        TRUE(),
        Effort > 200, 4,
        Effort > 100, 3,
        Effort > 50, 2,
        Effort > 0, 1,
        0
    )
RETURN
    EffortLevel
```

**Description:** Buckets **daily Relative Effort** into **levels 0–4** for use in **color scales / icons**.

---

### `Days Active`

```DAX
COALESCE(
    DISTINCTCOUNT('gold fact_activities'[date]),
    0
)
```

**Description:** Counts **distinct days** with at least one activity in the current context.

---

### `Days Active %`

```DAX
DIVIDE(
    [Days Active],
    [Days Total]
)
```

**Description:** Share of days that were **active** out of **all days** in the period.

---

### `Days Active % - Fill to 100`

```DAX
1 - [Days Active %]
```

**Description:** Complement of `Days Active %` – used to **fill donut charts up to 100%**.

---

### `Days Total`

```DAX
DISTINCTCOUNT(
    'gold dim_calendar'[date]
)
```

**Description:** **Total number of calendar days** in the current context (used as denominator for activity percentage).

---

### `Dummy Placeholder`

```DAX
""
```

**Description:** Empty string placeholder for visuals that require a **measure binding** but no value.

---

### `Dummy Title`

```DAX
""
```

**Description:** Placeholder for **empty titles** or technical binding in visuals.

---

### `Efforts To Local Legend`

```DAX
SELECTEDVALUE('gold dim_segment'[local_legend_effort_count]) - [Calculate Efforts Last 90 Days] + 1
```

**Description:** Calculates **how many more efforts** are needed to become **Local Legend**, assuming you need to exceed the current **local_legend_effort_count**.

---

### `Elevation Profile Placeholder`

```DAX
""
```

**Description:** Empty placeholder for **future elevation profile content**.

---

### `Gear Avg Pace/Speed`

```DAX
VAR GearType = SELECTEDVALUE('gold dim_gear'[gear_type])
VAR AvgSpeed = AVERAGE('gold fact_activities'[average_speed])

VAR AvgSpeedStr =
    FORMAT(AvgSpeed * 3.6, "0.00") & " km/h"

VAR AvgPaceStr =
    VAR sec_per_km = DIVIDE(1000, AvgSpeed)
    VAR m = INT(sec_per_km / 60)
    VAR s = MOD(sec_per_km, 60)
    RETURN
        FORMAT(m, "0") & ":" & FORMAT(s, "00") & " /km"

RETURN
    SWITCH(
        TRUE(),
        GearType = "Shoes", AvgPaceStr,
        GearType = "Bike", AvgSpeedStr,
        BLANK()
    )
```

**Description:** For the **selected gear**, returns either **average pace per km** (for shoes) or **average speed in km/h** (for bikes) as a **ready-to-display string**.

---

### `HeartrateZonesColors`

```DAX
SplitsColorLabels("Heartrate", MAX('gold fact_laps'[average_heartrate]))
```

**Description:** Uses a helper to map **average heart rate** to a **color label** for lap visuals.

---

### `Icon KOM`

```DAX
"https://d3nn82uaxijpm6.cloudfront.net/assets/svg/icon-segment-effort-01-6ffa3b..."
```

**Description:** URL to a **KOM/QOM icon**, used as an image in segment visuals.

---

### `Icon Local Legend`

```DAX
"https://zwiftinsider.com/wp-content/uploads/2021/10/laurel-strava.png"
```

**Description:** URL to a **Local Legend laurel icon**.

---

### `Is Week Filter Active`

```DAX
IF(
    HASONEVALUE('gold dim_calendar'[week_start_date]),
    1,
    0
)
```

**Description:** Returns **1** if **exactly one week** is selected, used as a **flag** for weekly cards and empty states.

---

### `Latest Segment Efforts`

```DAX
VAR SelectedSegment = SELECTEDVALUE('gold fact_segments_efforts'[segment_id])

VAR AllEfforts =
    FILTER(
        ALL('gold fact_segments_efforts'),
        'gold fact_segments_efforts'[segment_id] = SelectedSegment
    )

VAR TopEffortsWithPR =
    TOPN(40, AllEfforts, 'gold fact_segments_efforts'[datetime], DESC)

VAR TopEffortsWithoutPR =
    TOPN(40, AllEfforts, 'gold fact_segments_efforts'[datetime], DESC)

VAR TopEffort =
    TOPN(
        1,
        AllEfforts,
        'gold fact_segments_efforts'[moving_time], ASC,
        'gold fact_segments_efforts'[datetime], ASC
    )

VAR PR_Id = MAXX(TopEffort, 'gold fact_segments_efforts'[id])

VAR PR_is_in_Top40 =
    CONTAINS(TopEffortsWithPR, 'gold fact_segments_efforts'[id], PR_Id)

VAR CurId = SELECTEDVALUE('gold fact_segments_efforts'[id])

RETURN
    IF(
        PR_is_in_Top40,
        IF(CONTAINS(TopEffortsWithPR, 'gold fact_segments_efforts'[id], CurId), 1, 0),
        IF(CONTAINS(UNION(TopEffortsWithoutPR, TopEffort), 'gold fact_segments_efforts'[id], CurId), 1, 0)
    )
```

**Description:** Marks **which efforts should be shown** on the segment efforts chart: **top 40** recent efforts plus **PR** (if needed). Returns **1** for rows that should be visible.

---

### `Latest Segment Efforts Colors`

```DAX
VAR SelectedSegment = SELECTEDVALUE('gold fact_segments_efforts'[segment_id])

VAR AllEfforts =
    FILTER(
        ALL('gold fact_segments_efforts'),
        'gold fact_segments_efforts'[segment_id] = SelectedSegment
    )

VAR TopEffort =
    TOPN(
        1,
        AllEfforts,
        'gold fact_segments_efforts'[moving_time], ASC,
        'gold fact_segments_efforts'[datetime], ASC
    )

VAR PR_Id = MAXX(TopEffort, 'gold fact_segments_efforts'[id])

VAR CurId = SELECTEDVALUE('gold fact_segments_efforts'[id])

RETURN
    IF(CurId = PR_Id, "#FFA41A", "#FC5200")
```

**Description:** Returns a **highlight color** for the **PR effort** and another color for all other efforts in the latest efforts chart.

---

### `Local Legend Info`

```DAX
"
Local Legend achievement is awarded to the athlete who completes a given segment the most over a rolling 90-day period...
"
```

**Description:** Static **informational text** explaining **Local Legend** mechanics. Used in **tooltips/popups**.

---

### `Location Rank`

```DAX
VAR AggTable =
    SUMMARIZE(
        FILTER(
            ALL('gold dim_location'),
            NOT ( ISBLANK('gold dim_location'[locality]) )
        ),
        'gold dim_location'[country],
        'gold dim_location'[locality],
        "TotalAcitivities", DISTINCTCOUNT('gold fact_activities'[id]),
        "TotalTime", SUM('gold fact_activities'[moving_time_s])
    )
VAR RankTable =
    ADDCOLUMNS(
        AggTable,
        "Rank",
        ROWNUMBER(
            AggTable,
            ORDERBY([TotalAcitivities], DESC, [TotalTime], DESC)
        )
    )
RETURN
    MAXX(
        FILTER(
            RankTable,
            'gold dim_location'[country]
                = SELECTEDVALUE('gold dim_location'[country])
                && 'gold dim_location'[locality]
                = SELECTEDVALUE('gold dim_location'[locality])
        ),
        [Rank]
    )
```

**Description:** Ranks **locations (city–country)** by **number of activities** and **total time**. Returns the **rank of the current location** among all locations.

---

### `Longest Streak - Dates`

```DAX
CalculateLongestStreak("Dates")
```

**Description:** Uses a helper function to return **start/end dates** of the **longest streak of consecutive active days**.

---

### `Longest Streak - Value`

```DAX
CalculateLongestStreak("Val")
```

**Description:** Returns the **length in days** of the **longest active streak**.

---

### `PaceZonesColors`

```DAX
SplitsColorLabels("Pace", MAX('gold fact_laps'[average_speed]))
```

**Description:** Assigns a **color label** to a lap based on **pace**, used for **pace zone charts**.

---

### `Races Completed`

```DAX
COALESCE(
    CALCULATE(
        COUNTROWS('gold fact_activities'),
        KEEPFILTERS(
            'gold dim_workout_type'[type] = "Run - Race"
                || 'gold dim_workout_type'[type] = "Ride - Race"
        )
    ),
    0
)
```

**Description:** Counts **how many activities** are tagged as **Run – Race** or **Ride – Race**.

---

### `Relative Effort Info`

```DAX
"
Relative Effort measures how much cardiovascular work went into any activity that has heart rate data...
"
```

**Description:** Static **description text** for **Relative Effort**, shown in help popups.

---

### `Running Activites`

```DAX
VAR RowsWithRunning =
    CALCULATE(
        COUNTROWS('gold fact_activities'),
        KEEPFILTERS(
            'gold dim_sport_type'[sport_type] = "Run"
        )
    )
RETURN
    IF(RowsWithRunning > 0, 1, 0)
```

**Description:** Returns **1** if there is at least one **running activity** in the current context, otherwise **0**. Used for **conditional visibility**.

---

### `Running Total Distance`

```DAX
VAR CurrentDistance =
    CALCULATE(
        [Total Distance (Kilometers)],
        FILTER(
            ALLSELECTED('gold dim_calendar'),
            'gold dim_calendar'[date] <= MAX('gold dim_calendar'[date])
        )
    )
VAR MaxDistance =
    CALCULATE(
        [Total Distance (Kilometers)],
        ALLSELECTED('gold dim_calendar'[date])
    )
RETURN
    IF (
        ROUND(CurrentDistance, 2) = ROUND(MaxDistance, 2),
        BLANK(),
        CurrentDistance
    )
```

**Description:** Computes a **running total of distance** over time, but hides the **final value** so the line chart ends slightly **before the last point**, mimicking Strava-like gear-distance charts.

---

### `Segment Info`

```DAX
"
Segments are portions of roads or trails created by members where athletes can compare times...
"
```

**Description:** Static information about **segments**, used in **info popups**.

---

### `Selected KOM`

```DAX
SELECTEDVALUE('gold dim_segment'[Athlete Segment Stats Pr Time])
```

**Description:** Returns the **PR/KOM time** for the **selected segment** (from segment stats).

---

### `Show Top Segment Id?`

```DAX
VAR TopSegmentId = [Top Segment]
RETURN
    IF(MAX('gold dim_segment'[id]) = TopSegmentId, 1, 0)
```

**Description:** Returns **1** only for the **row representing the top segment**, used as a filter to show **top-segment-only** visuals.

---

### `Sport Active Last 12 Weeks`

```DAX
VAR CurrentWeek = MAX('gold dim_calendar'[week_start_date])
VAR StartWeek =
    CALCULATE(
        MAX('gold dim_calendar'[week_start_date]),
        FILTER(
            ALL('gold dim_calendar'),
            'gold dim_calendar'[week_start_date] <= CurrentWeek
        )
    ) - 7 * 12
VAR RowsWithEffort =
    CALCULATE(
        COUNTROWS('gold fact_activities'),
        KEEPFILTERS(
            'gold dim_calendar'[week_start_date] > StartWeek
                && 'gold dim_calendar'[week_start_date] <= CurrentWeek
        )
    )
RETURN
    IF(RowsWithEffort > 0, 1, 0)
```

**Description:** Checks whether there has been **any activity in the last 12 weeks** for the current sport; returns **1/0**. Used for **sport tiles visibility**.

---

### `Sport Icon Slicer`

```DAX
VAR Sport = SELECTEDVALUE('gold dim_sport_type'[sport_type])
RETURN
    SportIconUrl(Sport)
```

**Description:** Returns an **icon URL** for the **selected sport**, used in **image slicers**.

---

### `Sports with Relative Effort`

```DAX
VAR RowsWithEffort =
    CALCULATE(
        COUNTROWS(
            FILTER(
                'gold fact_activities',
                NOT ISBLANK('gold fact_activities'[relative_effort])
            )
        ),
        ALLEXCEPT('gold dim_sport_type', 'gold dim_sport_type'[sport_type])
    )
RETURN
    IF(RowsWithEffort > 0, 1, 0)
```

**Description:** For each sport, returns **1** if there is at least one activity with **Relative Effort**; used to **enable/disable** relative-effort visuals per sport.

---

### `Top Day of Week`

```DAX
VAR AggTable =
    SUMMARIZE(
        'gold dim_calendar',
        'gold dim_calendar'[day_of_week_name],
        "TotalActivities", DISTINCTCOUNT('gold fact_activities'[id])
    )
VAR TopRow =
    TOPN(
        1,
        AggTable,
        [TotalActivities],
        DESC
    )
RETURN
    MAXX(TopRow, 'gold dim_calendar'[day_of_week_name])
```

**Description:** Returns the **weekday name** with the **highest number of activities** in the current context.

---

### `Top Effort Time`

```DAX
VAR SelectedSegment = SELECTEDVALUE('gold fact_segments_efforts'[segment_id])

VAR AllEfforts =
    FILTER(
        ALL('gold fact_segments_efforts'),
        'gold fact_segments_efforts'[segment_id] = SelectedSegment
    )

VAR TopEffort =
    TOPN(
        1,
        AllEfforts,
        'gold fact_segments_efforts'[moving_time], ASC,
        'gold fact_segments_efforts'[datetime], ASC
    )

VAR PR_Time = MAXX(TopEffort, 'gold fact_segments_efforts'[moving_time])
RETURN
    PR_Time
```

**Description:** Returns the **best (fastest) segment time** for the selected segment.

---

### `Top Kudoers - Label`

```DAX
KudoerOrderAndLabel("Label")
```

**Description:** Helper that returns **labels** for “**Top kudoers**” charts.

---

### `Top Kudoers - Max Val`

```DAX
KudoerOrderAndLabel("Max") + 2
```

**Description:** Upper **axis bound** for kudoers charts (adds margin to max value).

---

### `Top Kudoers - Min Val`

```DAX
KudoerOrderAndLabel("Min") - 2
```

**Description:** Lower **axis bound** for kudoers charts (adds margin below min).

---

### `Top Kudoers - Order`

```DAX
KudoerOrderAndLabel("Order")
```

**Description:** Provides **sort order** for kudoers bar chart.

---

### `Top Part Of Day`

```DAX
VAR AggTable =
    SUMMARIZE(
        'gold dim_time',
        'gold dim_time'[hour],
        "TotalTime", SUM('gold fact_activities'[moving_time_s])
    )

VAR TopRow =
    TOPN(
        1,
        AggTable,
        [TotalTime],
        DESC
    )
VAR HourValue = MAXX(TopRow, 'gold dim_time'[hour])
RETURN
    IF (
        HourValue <> 23,
        HourValue & " - " & HourValue + 1,
        "23 - 0"
    )
```

**Description:** Returns the **hour slot** (e.g. `"18 - 19"`) with the **highest total training time**.

---

### `Top Segment`

```DAX
VAR AggTable =
    SUMMARIZE(
        'gold dim_segment',
        'gold dim_segment'[id],
        'gold dim_segment'[name],
        "TotalActivities", DISTINCTCOUNT('gold fact_segments_efforts'[id])
    )
VAR TopRow =
    TOPN(
        1,
        AggTable,
        [TotalActivities],
        DESC
    )
RETURN
    MAXX(TopRow, 'gold dim_segment'[name])
```

**Description:** Returns the **name of the segment** with the **highest number of efforts**.

---

### `Top Segment - Efforts`

```DAX
VAR AggTable =
    SUMMARIZE(
        'gold dim_segment',
        'gold dim_segment'[id],
        'gold dim_segment'[name],
        "TotalActivities", DISTINCTCOUNT('gold fact_segments_efforts'[id])
    )
VAR TopRow =
    TOPN(
        1,
        AggTable,
        [TotalActivities],
        DESC
    )
RETURN
    MAXX(TopRow, [TotalActivities])
```

**Description:** Returns the **effort count** for the **top segment**.

---

### `Top Segment - Map`

```DAX
VAR AggTable =
    SUMMARIZE(
        'gold dim_segment',
        'gold dim_segment'[id],
        'gold dim_segment'[map_id],
        "TotalActivities", DISTINCTCOUNT('gold fact_segments_efforts'[id])
    )
VAR TopRow =
    TOPN(
        1,
        AggTable,
        [TotalActivities],
        DESC
    )
RETURN
    MAXX(TopRow, 'gold dim_segment'[map_id])
```

**Description:** Returns the **map_id** of the **top segment**, used to display its **polyline/map**.

---

### `Total Activities`

```DAX
COALESCE(
    DISTINCTCOUNT('gold fact_activities'[id]),
    0
)
```

**Description:** Counts **distinct activities** in the current context.

---

### `Total Activities Monthly Heatmap`

```DAX
TotalsMonthlyHeatmap([Total Activities])
```

**Description:** Prepares **monthly activity counts** for **heatmap visuals**.

---

### `Total Calories (All)`

```DAX
CALCULATE(
    SUM('gold fact_activities'[calories]),
    ALL('gold fact_activities')
)
```

**Description:** **Lifetime total calories** over all activities, ignoring report filters.

---

### `Total Calories Summary`

```DAX
VAR Calories = [Total Calories (All)]
VAR BigMacs = ROUND(DIVIDE(Calories, 550), 0)

RETURN
    FORMAT(Calories, "#,0") & " kcal ≈ " & FORMAT(BigMacs, "0") & " Big Macs"
```

**Description:** Storytelling KPI: converts lifetime calories to an **approximate number of Big Macs**.

---

### `Total Days Active (All)`

```DAX
CALCULATE(
    DISTINCTCOUNT('gold fact_activities'[date]),
    ALL('gold fact_activities')
)
```

**Description:** **Lifetime number of days** with at least one activity.

---

### `Total Days Active Summary`

```DAX
VAR ActiveDays = [Total Days Active (All)]
VAR YearsActive = ROUND(DIVIDE(ActiveDays, 365), 1)

RETURN
    FORMAT(ActiveDays, "0") & " ≈ " &
    FORMAT(YearsActive, "0.0") & " years of daily activity"
```

**Description:** Storytelling KPI: expresses total **active days** as **years of daily activity**.

---

### `Total Distance (All)`

```DAX
CALCULATE(
    SUM('gold fact_activities'[distance]),
    ALL('gold fact_activities')
)
```

**Description:** **Lifetime total distance** in meters, across all activities.

---

### `Total Distance (Kilometers)`

```DAX
DIVIDE(
    [Total Distance (Meters)],
    1000
)
```

**Description:** Converts total distance from **meters** to **kilometers**.

---

### `Total Distance (Kilometers) - Str`

```DAX
MetersToDistanceText([Total Distance (Meters)])
```

**Description:** Text version of total distance in kilometers.

---

### `Total Distance (Meters)`

```DAX
SUM('gold fact_activities'[distance])
```

**Description:** Total **distance** (in meters) in the current filter context.

---

### `Total Distance Monthly Heatmap`

```DAX
TotalsMonthlyHeatmap([Total Distance (Kilometers)])
```

**Description:** Monthly **distance totals** prepared for **heatmap visuals**.

---

### `Total Distance Summary`

```DAX
VAR TotalDistKm = DIVIDE([Total Distance (All)], 1000)
VAR TripsWroclawParis = ROUND(DIVIDE(TotalDistKm, 1200), 1) // Wroclaw ↔ Paris ~1200 km

RETURN
    FORMAT(TotalDistKm, "#,0") & " km ≈ " &
    FORMAT(TripsWroclawParis, "0") & " Wroclaw-Paris trips"
```

**Description:** Converts **lifetime distance** into the **number of Wrocław–Paris trips** as a storytelling metric.

---

### `Total Effort`

```DAX
COALESCE(
    SUM('gold fact_activities'[relative_effort]),
    0
)
```

**Description:** Total **Relative Effort** in current context.

---

### `Total Effort Monthly Heatmap`

```DAX
TotalsMonthlyHeatmap([Total Effort])
```

**Description:** Monthly **Relative Effort totals** for heatmap visuals.

---

### `Total Elevation Gain (All)`

```DAX
CALCULATE(
    SUM('gold fact_activities'[total_elevation_gain]),
    ALL('gold fact_activities')
)
```

**Description:** Lifetime **elevation gain** over all activities.

---

### `Total Monthly - Chart`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    YearlyTotals(SelectedField, "Curr", "")
```

**Description:** Returns **current-year monthly totals** for the selected metric (time, distance, effort, activities) for the **chart series**.

---

### `Total Monthly - Chart YTD`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    YearlyTotals(SelectedField, "YTD", "")
```

**Description:** Returns **YTD totals** per month for the selected metric.

---

### `Total Monthly - Chart YTD Tooltip`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    YearlyTotals(SelectedField, "YTD", "Str")
```

**Description:** Text-formatted **YTD totals** per month for use in tooltips.

---

### `Total Monthly - Table Curr Year`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    YearlyTotals(SelectedField, "Curr", "Str")
```

**Description:** **Formatted current-year monthly totals** for the comparison table.

---

### `Total Monthly - Table Diff Str`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    YearlyTotals(SelectedField, "Diff", "Str")
```

**Description:** Text-formatted **difference between current and previous year**.

---

### `Total Monthly - Table Diff Val`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    YearlyTotals(SelectedField, "Diff", "")
```

**Description:** Numeric **difference** between current-year and previous-year totals.

---

### `Total Monthly - Table Prev Year`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    YearlyTotals(SelectedField, "Prev", "Str")
```

**Description:** **Formatted previous-year monthly totals** for comparison tables.

---

### `Total Monthly Rolling Totals - Title`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
VAR Title =
    SWITCH(
        TRUE(),
        SelectedField = 0, "Yearly Rolling Total Time",
        SelectedField = 1, "Yearly Rolling Total Distance",
        SelectedField = 2, "Yearly Rolling Total Effort",
        SelectedField = 3, "Yearly Rolling Total Activities",
        BLANK()
    )

RETURN
    Title
```

**Description:** Dynamic title for **rolling yearly totals** chart.

---

### `Total Monthly Totals - Title`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
VAR Title =
    SWITCH(
        TRUE(),
        SelectedField = 0, "Monthly Total Time",
        SelectedField = 1, "Monthly Total Distance",
        SelectedField = 2, "Monthly Total Effort",
        SelectedField = 3, "Monthly Total Activities",
        BLANK()
    )

RETURN
    Title
```

**Description:** Title for the **monthly totals** chart.

---

### `Total Monthly Yearly Comparison - Title`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
VAR Title =
    SWITCH(
        TRUE(),
        SelectedField = 0, "Yearly Comparison - Total Time",
        SelectedField = 1, "Yearly Comparison - Total Distance",
        SelectedField = 2, "Yearly Comparison - Total Effort",
        SelectedField = 3, "Yearly Comparison - Total Activities",
        BLANK()
    )

RETURN
    Title
```

**Description:** Title for **year-over-year comparison** visuals.

---

### `Total Moving Time (All)`

```DAX
CALCULATE(
    SUM('gold fact_activities'[moving_time_s]),
    ALL('gold fact_activities')
)
```

**Description:** Lifetime **moving time** in seconds.

---

### `Total Moving Time Summary`

```DAX
VAR TotalTimeSec = [Total Moving Time (All)]

VAR TotalTimeHours = INT(DIVIDE(TotalTimeSec, 3600))
VAR WorkMonths = ROUND(DIVIDE(TotalTimeSec, (8 * 3600 * 20)), 1)

RETURN
    FORMAT(TotalTimeHours, "#,0") & " h ≈ " &
    FORMAT(WorkMonths, "0.0") & " months of full-time work"
```

**Description:** Storytelling KPI: compares lifetime **training time** to **months of full-time work** (8h/day, 20 days/month).

---

### `Total Relative Effort`

```DAX
COALESCE(
    SUM('gold fact_activities'[relative_effort]),
    0
)
```

**Description:** Total **Relative Effort** in the current context.

---

### `Total Relative Effort Card`

```DAX
IF(
    [Is Week Filter Active],
    [Total Relative Effort],
    ""
)
```

**Description:** Shows **weekly Relative Effort** only when **exactly one week** is selected; otherwise returns an empty string for a clean card.

---

### `Total Time (Hours)`

```DAX
DIVIDE(
    [Total Time (Seconds)],
    3600
)
```

**Description:** Converts **total moving time** from seconds to **hours**.

---

### `Total Time (Hours) - Coalesce`

```DAX
COALESCE(
    DIVIDE(
        [Total Time (Seconds)],
        3600
    ),
    0
)
```

**Description:** Same as `Total Time (Hours)` but guarantees a **non-blank 0** for robust visuals.

---

### `Total Time (Hours) - Str`

```DAX
SecondsToDurationText([Total Time (Seconds)])
```

**Description:** Text **duration** for total time (hh:mm:ss).

---

### `Total Time (Hours) - Str Hours Label`

```DAX
SecondsToHoursText([Total Time (Seconds)])
```

**Description:** Converts total seconds into a **friendly “hours” label** (e.g. `"120 h"`).

---

### `Total Time (Seconds)`

```DAX
SUM('gold fact_activities'[moving_time_s])
```

**Description:** Total **moving time in seconds** in the current context.

---

### `Total Time Monthly Heatmap`

```DAX
TotalsMonthlyHeatmap([Total Time (Hours)])
```

**Description:** Monthly **time totals** prepared for **heatmap visuals**.

---

### `Total Time Zone (Hours)`

```DAX
DIVIDE(
    [Total Time Zone (Seconds)],
    3600
)
```

**Description:** Converts **time in heart-rate zones** from seconds to **hours**.

---

### `Total Time Zone (Hours) - Str`

```DAX
SecondsToDurationText([Total Time Zone (Seconds)])
```

**Description:** Text duration for **time-in-zone**.

---

### `Total Time Zone (Seconds)`

```DAX
COALESCE(
    SUM('gold fact_zones'[time]),
    0
)
```

**Description:** Total **time in all zones** (seconds) from `gold fact_zones`.

---

### `Total Time Zone 1M`

```DAX
TotalTimeZoneTimeIntelligence("1M")
```

**Description:** Wrapper that returns **time-in-zone** over the **last month**.

---

### `Total Time Zone 1Y`

```DAX
TotalTimeZoneTimeIntelligence("1Y")
```

**Description:** Time-in-zone over the **last 12 months**.

---

### `Total Time Zone 3M`

```DAX
TotalTimeZoneTimeIntelligence("3M")
```

**Description:** Time-in-zone over the **last 3 months**.

---

### `Total Time Zone 6M`

```DAX
TotalTimeZoneTimeIntelligence("6M")
```

**Description:** Time-in-zone over the **last 6 months**.

---

### `Total Time Zone 7D`

```DAX
TotalTimeZoneTimeIntelligence("7D")
```

**Description:** Time-in-zone over the **last 7 days**.

---

### `Total Time Zone Time Intelligence - % Zone 2`

```DAX
VAR SelectedTotalTimeZoneTimeIntelligence =
    SELECTEDVALUE('Training Zones Time Intelligence'[Training Zones Time Intelligence Order])
VAR PctZone2Value =
    SWITCH(
        TRUE(),
        SelectedTotalTimeZoneTimeIntelligence = 0, TotalTimeZonePctZone2([Total Time Zone 7D]),
        SelectedTotalTimeZoneTimeIntelligence = 1, TotalTimeZonePctZone2([Total Time Zone 1M]),
        SelectedTotalTimeZoneTimeIntelligence = 2, TotalTimeZonePctZone2([Total Time Zone 3M]),
        SelectedTotalTimeZoneTimeIntelligence = 3, TotalTimeZonePctZone2([Total Time Zone 6M]),
        SelectedTotalTimeZoneTimeIntelligence = 4, TotalTimeZonePctZone2([Total Time Zone YTD]),
        SelectedTotalTimeZoneTimeIntelligence = 5, TotalTimeZonePctZone2([Total Time Zone 1Y]),
        BLANK()
    )
RETURN
    PctZone2Value
```

**Description:** Returns the **percentage of time spent in Zone 2** for the **selected period** (7D, 1M, 3M, 6M, YTD, 1Y).

---

### `Total Time Zone Time Intelligence - % Zone 2 Diff`

```DAX
VAR Difference =
    IF(
        ISNUMBER([Total Time Zone Time Intelligence - % Zone 2])
            && ISNUMBER([Total Time Zone Time Intelligence - Prev Period % Zone 2]),
        ([Total Time Zone Time Intelligence - % Zone 2]
            - [Total Time Zone Time Intelligence - Prev Period % Zone 2]) * 100,
        BLANK()
    )
VAR ReturnString =
    IF(
        Difference >= 0 && NOT ( ISBLANK(Difference) ),
        "(" & UNICHAR(8593) & FORMAT(Difference, "0") & "%)",
        IF(
            Difference < 0 && NOT ( ISBLANK(Difference) ),
            "(" & UNICHAR(8595) & FORMAT(Difference, "0") & "%)",
            BLANK()
        )
    )
RETURN
    ReturnString
```

**Description:** Compares **current Zone 2 %** to the **previous period**, returning a **text diff** with **up/down arrows**.

---

### `Total Time Zone Time Intelligence - Dates`

```DAX
VAR SelectedTotalTimeZoneTimeIntelligence =
    SELECTEDVALUE('Training Zones Time Intelligence'[Training Zones Time Intelligence Order])
VAR MaxDate = MAX('gold dim_calendar'[date])

VAR Date_7D =
    CALCULATE(
        MIN('gold dim_calendar'[date]),
        DATESINPERIOD(
            'gold dim_calendar'[date],
            MAX('gold dim_calendar'[date]),
            -7,
            DAY
        )
    )
VAR Date_1M =
    CALCULATE(
        MIN('gold dim_calendar'[date]),
        DATESINPERIOD(
            'gold dim_calendar'[date],
            MAX('gold dim_calendar'[date]),
            -7 * 4,
            DAY
        )
    )
VAR Date_3M =
    CALCULATE(
        MIN('gold dim_calendar'[date]),
        DATESINPERIOD(
            'gold dim_calendar'[date],
            MAX('gold dim_calendar'[date]),
            -7 * 4 * 3,
            DAY
        )
    )
VAR Date_6M =
    CALCULATE(
        MIN('gold dim_calendar'[date]),
        DATESINPERIOD(
            'gold dim_calendar'[date],
            MAX('gold dim_calendar'[date]),
            -7 * 4 * 6,
            DAY
        )
    )
VAR Date_YTD =
    CALCULATE(
        SELECTEDVALUE('gold dim_calendar'[year_start_date]),
        FILTER(
            'gold dim_calendar',
            'gold dim_calendar'[year] = MAX('gold dim_calendar'[year])
        )
    )
VAR Date_1Y =
    CALCULATE(
        MIN('gold dim_calendar'[date]),
        DATESINPERIOD(
            'gold dim_calendar'[date],
            MAX('gold dim_calendar'[date]),
            -7 * 4 * 12,
            DAY
        )
    )
VAR DateValue =
    SWITCH(
        TRUE(),
        SelectedTotalTimeZoneTimeIntelligence = 0, Date_7D,
        SelectedTotalTimeZoneTimeIntelligence = 1, Date_1M,
        SelectedTotalTimeZoneTimeIntelligence = 2, Date_3M,
        SelectedTotalTimeZoneTimeIntelligence = 3, Date_6M,
        SelectedTotalTimeZoneTimeIntelligence = 4, Date_YTD,
        SelectedTotalTimeZoneTimeIntelligence = 5, Date_1Y,
        BLANK()
    )
RETURN
    FORMAT(DateValue, "d mmmm yyyy") & " - "
        & FORMAT(MAX('gold dim_calendar'[date]), "d mmmm yyyy")
```

**Description:** Builds a **human-readable date range label** for the **selected training zone period**.

---

### `Total Time Zone Time Intelligence - Prev Period % Zone 2`

```DAX
VAR SelectedTotalTimeZoneTimeIntelligence =
    SELECTEDVALUE('Training Zones Time Intelligence'[Training Zones Time Intelligence Order])
VAR PrevPctZone2Value =
    SWITCH(
        TRUE(),
        SelectedTotalTimeZoneTimeIntelligence = 0, TotalTimeZonePrevTimeIntelligence("7D"),
        SelectedTotalTimeZoneTimeIntelligence = 1, TotalTimeZonePrevTimeIntelligence("1M"),
        SelectedTotalTimeZoneTimeIntelligence = 2, TotalTimeZonePrevTimeIntelligence("3M"),
        SelectedTotalTimeZoneTimeIntelligence = 3, TotalTimeZonePrevTimeIntelligence("6M"),
        SelectedTotalTimeZoneTimeIntelligence = 4, TotalTimeZonePrevTimeIntelligence("YTD"),
        SelectedTotalTimeZoneTimeIntelligence = 5, TotalTimeZonePrevTimeIntelligence("1Y"),
        BLANK()
    )
RETURN
    PrevPctZone2Value
```

**Description:** Returns **Zone 2 %** for the **previous period** (matching the currently selected window).

---

### `Total Time Zone Time Intelligence (Hours) - Str`

```DAX
VAR SelectedTotalTimeZoneTimeIntelligence =
    SELECTEDVALUE('Training Zones Time Intelligence'[Training Zones Time Intelligence Order])
VAR StrHours =
    SWITCH(
        TRUE(),
        SelectedTotalTimeZoneTimeIntelligence = 0, SecondsToDurationText([Total Time Zone 7D]),
        SelectedTotalTimeZoneTimeIntelligence = 1, SecondsToDurationText([Total Time Zone 1M]),
        SelectedTotalTimeZoneTimeIntelligence = 2, SecondsToDurationText([Total Time Zone 3M]),
        SelectedTotalTimeZoneTimeIntelligence = 3, SecondsToDurationText([Total Time Zone 6M]),
        SelectedTotalTimeZoneTimeIntelligence = 4, SecondsToDurationText([Total Time Zone YTD]),
        SelectedTotalTimeZoneTimeIntelligence = 5, SecondsToDurationText([Total Time Zone 1Y]),
        BLANK()
    )
RETURN
    StrHours
```

**Description:** Returns **formatted total time-in-zone** for the **selected period**.

---

### `Total Time Zone YTD`

```DAX
TotalTimeZoneTimeIntelligence("YTD")
```

**Description:** Time-in-zone for **Year-To-Date**.

---

### `Total X Monthly Heatmap`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, COALESCE([Total Time Monthly Heatmap], 0),
        SelectedField = 1, COALESCE([Total Distance Monthly Heatmap], 0),
        SelectedField = 2, COALESCE([Total Effort Monthly Heatmap], 0),
        SelectedField = 3, COALESCE([Total Activities], 0),
        BLANK()
    )
```

**Description:** Master measure for **month × year heatmap** values, switching between time, distance, effort and activities.

---

### `Total X Monthly Heatmap - Labels`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, SecondsToDurationText(COALESCE([Total Time Monthly Heatmap] * 3600, 0)),
        SelectedField = 1, MetersToDistanceText(COALESCE([Total Distance Monthly Heatmap] * 1000, 0)),
        SelectedField = 2, COALESCE([Total Effort Monthly Heatmap], 0),
        SelectedField = 3, COALESCE([Total Activities], 0),
        BLANK()
    )
```

**Description:** Formatted labels for **monthly heatmap** cells.

---

### `Total X Monthly Heatmap - Title`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    SWITCH(
        TRUE(),
        SelectedField = 0, "Total Monthly Time by Month and Year",
        SelectedField = 1, "Total Monthly Distance by Month and Year",
        SelectedField = 2, "Total Monthly Effort by Month and Year",
        SelectedField = 3, "Total Monthly Number of Activities by Month and Year",
        BLANK()
    )
```

**Description:** Dynamic **title** for the **monthly heatmap** visual.

---

### `User Name`

```DAX
"Bartosz Łowicki"
```

**Description:** Stores the **athlete’s name** for display on header cards.

---

### `User Profile Picture`

```DAX
"https://dgalywyr863hv.cloudfront.net/pictures/athletes/81055898/21708389/8/large.jpg"
```

**Description:** URL to the **athlete’s profile picture**, used as an **Image URL**.

---

### `Weekly Distance (Kilometers)`

```DAX
DIVIDE(
    [Weekly Distance (Meters)],
    1000
)
```

**Description:** Weekly distance converted from **meters** to **kilometers**.

---

### `Weekly Distance (Meters)`

```DAX
COALESCE(
    CALCULATE(
        [Total Distance (Meters)],
        FILTER(
            'gold dim_calendar',
            'gold dim_calendar'[week_start_date] = MAX('gold dim_calendar'[week_start_date])
        )
    ),
    0
)
```

**Description:** Total **distance** for the **week selected** via `week_start_date`.

---

### `Weekly Distance Str`

```DAX
MetersToDistanceText([Weekly Distance (Meters)])
```

**Description:** Weekly distance as a **formatted string**.

---

### `Weekly Effort Color`

```DAX
SWITCH(
    TRUE(),
    [Total Relative Effort] < [3-Weeks Relative Effort Avg Low Val], "#ae9dff",
    [Total Relative Effort] >= [3-Weeks Relative Effort Avg Low Val]
        && [Total Relative Effort] <= [3-Weeks Relative Effort Avg High Val], "#b03cff",
    [Total Relative Effort] > [3-Weeks Relative Effort Avg High Val], "#bd0c0f"
)
```

**Description:** Returns a **color code** representing whether weekly effort is **below, within or above** the 3-week range.

---

### `Weekly Effort Placeholder Message`

```DAX
"Select a week on the chart above to see details."
```

**Description:** Default **placeholder text** when no specific week is selected.

---

### `Weekly Effort Text`

```DAX
SWITCH(
    TRUE(),
    [Total Relative Effort] < [3-Weeks Relative Effort Avg Low Val], "Below Weekly Range",
    [Total Relative Effort] >= [3-Weeks Relative Effort Avg Low Val]
        && [Total Relative Effort] < [3-Weeks Relative Effort Avg], "Consistent Training",
    [Total Relative Effort] >= [3-Weeks Relative Effort Avg]
        && [Total Relative Effort] < [3-Weeks Relative Effort Avg High Val], "Steady Progress",
    [Total Relative Effort] > [3-Weeks Relative Effort Avg High Val], "Well Above Weekly Range"
)
```

**Description:** Provides **human-readable text** describing the **weekly training load** vs. the 3-week baseline.

---

### `Weekly Effort Transparency Background`

```DAX
IF(
    [Is Week Filter Active],
    "rgba(0,0,0,0)",
    "#FFF7F3"
)
```

**Description:** Controls **background color** of the weekly card depending on whether a week is selected (used for empty state).

---

### `Weekly Effort Transparency Text`

```DAX
IF(
    [Is Week Filter Active],
    "rgba(0,0,0,0)",
    "#252423"
)
```

**Description:** Controls **text color** in the weekly card in the same way.

---

### `Weekly Time (Hours)`

```DAX
DIVIDE(
    [Weekly Time (Seconds)],
    3600
)
```

**Description:** Weekly total **time** in hours.

---

### `Weekly Time (Seconds)`

```DAX
COALESCE(
    CALCULATE(
        [Total Time (Seconds)],
        FILTER(
            'gold dim_calendar',
            'gold dim_calendar'[week_start_date] = MAX('gold dim_calendar'[week_start_date])
        )
    ),
    0
)
```

**Description:** Weekly total **moving time in seconds**.

---

### `Weekly Time Str`

```DAX
SecondsToDurationText([Weekly Time (Seconds)])
```

**Description:** Weekly time as a **formatted duration** string.

---

### `Weekly Time/Distance`

```DAX
VAR SelectedSport = SELECTEDVALUE('gold dim_sport_type'[sport_type])
VAR SportType = DefineSportSummaryType(SelectedSport)

RETURN
    SWITCH(
        TRUE(),
        SportType = "Distance", [Weekly Distance (Kilometers)],
        SportType = "Time", [Weekly Time (Hours)],
        BLANK()
    )
```

**Description:** For each sport, returns either **weekly distance** or **weekly time** depending on the **sport type classification**.

---

### `Weekly Time/Distance Str`

```DAX
VAR SelectedSport = SELECTEDVALUE('gold dim_sport_type'[sport_type])
VAR SportType = DefineSportSummaryType(SelectedSport)

RETURN
    SWITCH(
        TRUE(),
        SportType = "Distance", [Weekly Distance Str],
        SportType = "Time", [Weekly Time Str],
        BLANK()
    )
```

**Description:** Text version of the above **weekly summary**.

---

### `Weekly Time/Distance Title`

```DAX
VAR SelectedSport = SELECTEDVALUE('gold dim_sport_type'[sport_type])
VAR SportType = DefineSportSummaryType(SelectedSport)

RETURN
    SWITCH(
        TRUE(),
        SportType = "Distance", "Total Weekly Distance",
        SportType = "Time", "Total Weekly Time",
        BLANK()
    )
```

**Description:** Dynamic **title** for weekly summary cards depending on sport type.

---

### `Yearly Summary - Is Relative Effort Selected?`

```DAX
VAR SelectedField = SELECTEDVALUE(SummaryType[SummaryType Order])
RETURN
    IF(
        SelectedField = 2,
        1,
        0
    )
```

**Description:** Returns **1** if the **Relative Effort metric** is selected in `SummaryType`, otherwise **0**. Used to show/hide **Relative Effort** info.

---

### `Yearly Time/Distance`

```DAX
VAR SelectedSport = SELECTEDVALUE('gold dim_sport_type'[sport_type])
VAR SportType = DefineSportSummaryType(SelectedSport)

RETURN
    SWITCH(
        TRUE(),
        SportType = "Distance", [Total Distance (Kilometers) - Str],
        SportType = "Time", [Total Time (Hours) - Str],
        BLANK()
    )
```

**Description:** For **Rewind/year summary**, returns either **yearly distance** or **yearly time** as **formatted text** depending on sport type.

---

### `Years with Relative Effort`

```DAX
VAR RowsWithEffort =
    CALCULATE(
        COUNTROWS(
            FILTER(
                'gold fact_activities',
                NOT ISBLANK('gold fact_activities'[relative_effort])
            )
        ),
        ALLEXCEPT('gold dim_calendar', 'gold dim_calendar'[year])
    )
RETURN
    IF(RowsWithEffort > 0, 1, 0)
```

**Description:** For each year, returns **1** if there is at least one activity with **Relative Effort** available; used to control **visibility of Relative Effort visuals** per year.

---

