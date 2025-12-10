## DAX user-defined functions (UDFs)

This project makes heavy use of **DAX user-defined functions (UDFs)**, a new Power BI feature that lets you define **reusable, parameterised logic** and call it from multiple measures.

This helped to:

* keep the DAX layer **DRY** (Don’t Repeat Yourself),
* centralise complex logic (time intelligence, streaks, formatting),
* make individual measures **shorter, cleaner and easier to reason about**.

Below is an overview of all UDFs used in the model.

---

### 1. Average over active days/months

#### 1.1 `AvgTotalsDaily`

**Purpose:**
Compute **average daily value** of a given measure, but **only on days with activity** (Total > 0).

**Used by:**
`Avg Total Time Daily`, `Avg Total Distance Daily`, `Avg Total Effort Daily`, `Avg Total X Daily`, daily heatmap measures.

```DAX
function AvgTotalsDaily =
    (TotalMeasure: expr) =>
        VAR DayTable = 
            SUMMARIZE(
                'gold dim_calendar',
                'gold dim_calendar'[date], 
                "Total", CALCULATE(TotalMeasure)
            )
        VAR ActiveDays =
            FILTER(DayTable, [Total] > 0) 
            
        RETURN 
            DIVIDE(
                SUMX(ActiveDays, [Total]), 
                COUNTROWS(ActiveDays))
```

---

#### 1.2 `AvgTotalsMonthly`

**Purpose:**
Compute **average monthly value** of a given measure, but **only for months with activity**.

**Used by:**
`Avg Total Time Monthly`, `Avg Total Distance Monthly`, `Avg Total Effort Monthly`, `Avg Total Activities Monthly`, `Avg Total X Monthly`.

```DAX
function AvgTotalsMonthly =
    (TotalMeasure: expr) =>
        VAR MonthTable = 
            SUMMARIZE(
                'gold dim_calendar',
                'gold dim_calendar'[month_start_date], 
                "Total", CALCULATE(TotalMeasure)
            )
        VAR ActiveMonths =
            FILTER(MonthTable, [Total] > 0) 
            
        RETURN 
            DIVIDE(
                SUMX(ActiveMonths, [Total]), 
                COUNTROWS(ActiveMonths))
```

---

### 2. Icons and buttons

#### 2.1 `ButtonIconUrl`

**Purpose:**
Map **gear type/status** to a **button icon URL**.

**Used by:**
`Button Icon - Gear Status`, `Button Icon - Gear Type`.

```DAX
function ButtonIconUrl =
    (ButtonVal: string val) =>
        VAR ButtonIcon =
            SWITCH (
                ButtonVal,

                "Bike",              "https://img.icons8.com/material-outlined/48/bicycle.png",
                "Shoes",             "https://img.icons8.com/material-outlined/48/trainers.png",
                
                "Retired",           "https://img.icons8.com/material-outlined/48/waste.png",
                "Active",            "https://img.icons8.com/material-outlined/48/ok--v1.png",

                BLANK()
            )
        RETURN
            ButtonIcon
```

---

### 3. Streaks and calendar logic

#### 3.1 `CalculateLongestStreak`

**Purpose:**
Calculate the **longest streak of consecutive active days**, and optionally return the **date range** of that streak.

**Used by:**
`Longest Streak - Value`, `Longest Streak - Dates`.

```DAX
function CalculateLongestStreak =
    (ReturnValue: string val) =>
        VAR DatesOnly =
            DISTINCT('gold fact_activities'[date])
        VAR DatesWithRank =
            ADDCOLUMNS(
                DatesOnly,
                "DateRank",
                    RANKX(
                        DatesOnly,
                        'gold fact_activities'[date],
                        ,
                        ASC,
                        DENSE
                    )
            )
        VAR Grouped =
            ADDCOLUMNS(
                DatesWithRank,
                "GroupKey",
                    INT('gold fact_activities'[date]) - [DateRank]
            )
        VAR Streaks =
            ADDCOLUMNS(
                SUMMARIZE(Grouped, [GroupKey]),
                "StreakLength",
                    VAR GK = [GroupKey]
                    RETURN
                        COUNTROWS(
                            FILTER(Grouped, [GroupKey] = GK)
                        ),
                "MinDate",
                    VAR GK = [GroupKey]
                    RETURN
                        CALCULATE(
                            MIN('gold fact_activities'[date]),
                            FILTER(Grouped, [GroupKey] = GK)
                        ),
                "MaxDate",
                    VAR GK = [GroupKey]
                    RETURN
                        CALCULATE(
                            MAX('gold fact_activities'[date]),
                            FILTER(Grouped, [GroupKey] = GK)
                        )
            )
        VAR LongestStreakValue = MAXX(Streaks, [StreakLength])
        VAR LongestStreakRow =
            TOPN(
                1,
                FILTER(Streaks, [StreakLength] = LongestStreakValue),
                [MinDate],
                ASC
            )
        VAR MaxDateValue = MINX(LongestStreakRow, [MaxDate])
        VAR MinDateValue = MINX(LongestStreakRow, [MinDate])
    RETURN
        SWITCH(
            TRUE(),
            ReturnValue = "Val",   LongestStreakValue,
            ReturnValue = "Dates", FORMAT(MinDateValue, "d mmmm yyyy")  & " - " & FORMAT(MaxDateValue, "d mmmm yyyy"),
            BLANK()
        )
```

---

### 4. Pace, speed and distance formatting

#### 4.1 `CalculatePace`

**Purpose:**
Return a **sport-specific pace/speed string** based on sport type and speed (m/s).

**Used by:**
Activity-level visuals where speed/pace needs to be shown in different units per sport.

```DAX
function CalculatePace =
    (SportName: string val, SpeedValueMPS: val) =>
        VAR pace_km_sports =
            {"Run", "Walk", "Hike", "Snowshoe", "VirtualRun", "Wheelchair", "Soccer"}

        VAR pace_100m_sports =
            {"Swim"}

        VAR pace_500m_sports =
            {"Rowing"}

        VAR speed_kmh_sports =
            {"Ride", "EBikeRide", "Canoe", "IceSkate", "AlpineSki", "Kitesurf", "VirtualRide",
            "RollerSki", "Snowboard", "Handcycle", "StandUpPaddling", "BackcountrySki",
            "Elliptical", "Velomobile", "NordicSki", "Kayaking", "Skateboard",
            "Sail", "Windsurf", "InlineSkate"}

        VAR none_sports =
            {"WeightTraining", "Crossfit", "Workout", "RockClimb", "StairStepper",
            "Golf", "Surfing", "Yoga"}

        // -------- Pace in min/km --------
        VAR pace_km =
            VAR sec_per_km = DIVIDE(1000, SpeedValueMPS)
            VAR round_sec_per_km  = ROUND(sec_per_km, 0)
            VAR m = INT(round_sec_per_km / 60)
            VAR s = MOD(round_sec_per_km, 60)
            RETURN FORMAT(m, "0") & ":" & FORMAT (s, "00") & "/km"

        // -------- Pace in min/100m (swimming) --------
        VAR pace_100m =
            VAR sec_per_100 = DIVIDE(100, SpeedValueMPS)
            VAR round_sec_per_100  = ROUND(sec_per_100, 0)
            VAR m = INT(round_sec_per_100 / 60)
            VAR s = MOD(round_sec_per_100, 60)
            RETURN FORMAT(m, "0") & ":" & FORMAT(s, "00") & "/100m"

        // -------- Pace in min/500m (rowing) --------
        VAR pace_500m =
            VAR sec_per_500 = DIVIDE(500, SpeedValueMPS)
            VAR round_sec_per_500  = ROUND(sec_per_500, 0)
            VAR m = INT(round_sec_per_500 / 60)
            VAR s = MOD(round_sec_per_500, 60)
            RETURN FORMAT(m, "0") & ":" & FORMAT(s, "00") & "/500m"

        // -------- Speed in km/h --------
        VAR speed_kmh =
            FORMAT (SpeedValueMPS * 3.6, "0.00") & "km/h"

        // -------- Return correct format depending on sport type --------
        RETURN
            SWITCH(
                TRUE(),
                SpeedValueMPS <= 0,                 BLANK(),
                SportName IN pace_km_sports,        pace_km,
                SportName IN pace_100m_sports,      pace_100m,
                SportName IN pace_500m_sports,      pace_500m,
                SportName IN speed_kmh_sports,      speed_kmh,
                // Other sports (e.g. Crossfit, Yoga) → no speed/pace shown
                BLANK()
            )
```

---

#### 4.2 `CalculatePaceByDistanceAndTime`

**Purpose:**
Return **pace (min/km)** based on distance (m) and time (s).

```DAX
function CalculatePaceByDistanceAndTime =
    (DistanceMeters: int64 val, TimeSeconds: int64 val) =>
        VAR SpeedMps =
            DIVIDE(
                DistanceMeters,
                TimeSeconds
            )
        
        VAR SecPerKm = DIVIDE(1000, SpeedMps)
        VAR m = INT(SecPerKm / 60)
        VAR s = MOD(SecPerKm, 60)
        RETURN FORMAT(m, "0") & ":" & FORMAT (s, "00") & "/km"
```

---

#### 4.3 `CalculatePaceByDistanceNameAndTime`

**Purpose:**
Return pace for **named standard distances** (5K, 10K, Marathon…) using the above function.

```DAX
function CalculatePaceByDistanceNameAndTime =
    (DistanceName: string val, TimeSeconds: int64 val) =>
        VAR Distance =
            SWITCH(
                TRUE(),
                DistanceName = "400m",        400,
                DistanceName = "1/2 mile",    1609.344 / 2,
                DistanceName = "1K",          1000,
                DistanceName = "1 mile",      1609.344,
                DistanceName = "2 mile",      2 * 1609.344,
                DistanceName = "5K",          5000,
                DistanceName = "10K",         10000,
                DistanceName = "15K",         15000,
                DistanceName = "10 mile",     10 * 1609.344,
                DistanceName = "20K",         20000,
                DistanceName = "Half-Marathon", 21097.5,
                DistanceName = "30K",         30000,
                DistanceName = "Marathon",    42195
            )
        RETURN
            CalculatePaceByDistanceAndTime(Distance, TimeSeconds)
```

---

#### 4.4 `MetersToDistanceText`

**Purpose:**
Convert **meters** into a **formatted km string**.

```DAX
function MetersToDistanceText =
    (TotalMeters: decimal) =>
        VAR Kilometers = 
            DIVIDE(
                TotalMeters,
                1000
            )
        VAR FormattedDistance = 
            IF(NOT(ISBLANK(Kilometers)),
                FORMAT(Kilometers, "0.0") & " km",
                BLANK()
            )
        RETURN
            FormattedDistance
```

---

#### 4.5 `ResultToSeconds`

**Purpose:**
Parse a **human-typed time** (e.g. `1:23:45`, `45:10`, `38s`) into **seconds**.

```DAX
function ResultToSeconds =
    (Result: string val) =>
        VAR Raw = LOWER(TRIM(Result))
        VAR Base = IF(RIGHT(Raw, 1) = "s", LEFT(Raw, LEN(Raw) - 1), Raw)
        VAR IsBlank = (Base = "") || (Base = BLANK())
        VAR ColonCount = LEN(Base) - LEN(SUBSTITUTE(Base, ":", ""))
        VAR Parts = SUBSTITUTE(Base, ":", "|")
        VAR H = IF(ColonCount = 2, VALUE(PATHITEM(Parts, 1)), 0)
        VAR M = SWITCH(
            TRUE(),
            ColonCount = 2, VALUE(PATHITEM(Parts, 2)),
            ColonCount = 1, VALUE(PATHITEM(Parts, 1)),
            0
        )
        VAR S = SWITCH(
            TRUE(),
            ColonCount = 2, VALUE(PATHITEM(Parts, 3)),
            ColonCount = 1, VALUE(PATHITEM(Parts, 2)),
            ColonCount = 0, VALUE(Base),
            0
        )
        RETURN IF( IsBlank, BLANK(), H*3600 + M*60 + S )
```

---

#### 4.6 `SecondsToDurationResultText`

**Purpose:**
Format seconds as a **result style**: `HH:MM:SS` or `MM:SS`.

```DAX
function SecondsToDurationResultText =
    (TotalSeconds: int64) =>
        VAR Hours = INT(TotalSeconds / 3600)
        VAR Minutes = INT(MOD(TotalSeconds, 3600) / 60)
        VAR Seconds = MOD(TotalSeconds, 60)
        VAR FormattedTime =
            IF(
                Hours > 0,
                FORMAT(Hours, "00") & ":" & FORMAT(Minutes, "00") & ":" & FORMAT(Seconds, "00"),
                FORMAT(Minutes, "00") & ":" & FORMAT(Seconds, "00")
            )
        RETURN
            FormattedTime
```

---

#### 4.7 `SecondsToDurationText`

**Purpose:**
Format seconds as a **human-readable duration** like `1h 23m`, `45m 10s`, `30s`.

```DAX
function SecondsToDurationText =
    (TotalSeconds: int64) =>
        VAR Hours = INT(TotalSeconds / 3600)
        VAR Minutes = INT(MOD(TotalSeconds, 3600) / 60)
        VAR Seconds = MOD(TotalSeconds, 60)
        VAR FormattedTime =
            SWITCH(
                TRUE(),
                Hours > 0,          FORMAT(Hours, "0") & "h " & FORMAT(Minutes, "0") & "m",
                Minutes > 0,        FORMAT(Minutes, "0") & "m " & FORMAT(Seconds, "0") & "s",
                NOT(ISBLANK(Seconds)), FORMAT(Seconds, "0") & "s",
                BLANK()
            )
        RETURN
            FormattedTime
```

---

#### 4.8 `SecondsToHoursText`

**Purpose:**
Convert seconds into a **rounded number of hours** with an `H` suffix.

```DAX
function SecondsToHoursText =
    (TotalSeconds: int64) =>
        VAR Hours = 
            IF(NOT(ISBLANK(TotalSeconds)),
                INT(TotalSeconds / 3600),
                0
            )
        VAR FormattedHour = FORMAT(Hours, "0") & "H"
        RETURN
            FormattedHour
```

---

### 5. Split colors (pace / heart rate zones)

#### 5.1 `SplitsColorLabels`

**Purpose:**
Return a **color hex code** for a split based on **heart-rate or pace zones** loaded from `gold fact_zones`.

**Used by:**
`HeartrateZonesColors`, `PaceZonesColors`.

```DAX
function SplitsColorLabels =
    (SplitTypeName: string val, SplitValue: expr) =>
        VAR ActivityId = SELECTEDVALUE('gold fact_activities'[id])
        VAR SplitType = IF(
            SplitTypeName = "Heartrate" || SplitTypeName = "Pace",
            SplitTypeName,
            BLANK()
        )
                
        VAR Zone1 = CALCULATE(
            MAX('gold fact_zones'[max]),
            KEEPFILTERS('gold fact_zones'[activity_id] = ActivityId),
            KEEPFILTERS('gold fact_zones'[type] = SplitType),
            KEEPFILTERS('gold fact_zones'[zone_number] = 1)
            )
        VAR Zone2 = CALCULATE(
            MAX('gold fact_zones'[max]),
            KEEPFILTERS('gold fact_zones'[activity_id] = ActivityId),
            KEEPFILTERS('gold fact_zones'[type] = SplitType),
            KEEPFILTERS('gold fact_zones'[zone_number] = 2)
            )
        VAR Zone3 = CALCULATE(
            MAX('gold fact_zones'[max]),
            KEEPFILTERS('gold fact_zones'[activity_id] = ActivityId),
            KEEPFILTERS('gold fact_zones'[type] = SplitType),
            KEEPFILTERS('gold fact_zones'[zone_number] = 3)
            )
        VAR Zone4 = CALCULATE(
            MAX('gold fact_zones'[max]),
            KEEPFILTERS('gold fact_zones'[activity_id] = ActivityId),
            KEEPFILTERS('gold fact_zones'[type] = SplitType),
            KEEPFILTERS('gold fact_zones'[zone_number] = 4)
            )
        VAR Zone5 = CALCULATE(
            MAX('gold fact_zones'[max]),
            KEEPFILTERS('gold fact_zones'[activity_id] = ActivityId),
            KEEPFILTERS('gold fact_zones'[type] = SplitType),
            KEEPFILTERS('gold fact_zones'[zone_number] = 5)
            )
        
        VAR ColorZones =
            IF(SplitType = "Heartrate",
                SWITCH(
                    TRUE(),
                    Zone1 = BLANK() || Zone2 = BLANK() || Zone3 = BLANK() || Zone4 = BLANK() || Zone4 = BLANK() || Zone5 = BLANK(), "#D9A7A8",
                    SplitValue <= Zone1, "#FAD4D4",
                    SplitValue <= Zone2, "#F28E8E",
                    SplitValue <= Zone3, "#E85C5C",
                    SplitValue <= Zone4, "#D93030",
                    SplitValue > Zone4, "#990000"
                ),
            IF(SplitType = "Pace",
                SWITCH(
                    TRUE(),
                    Zone1 = BLANK() || Zone2 = BLANK() || Zone3 = BLANK() || Zone4 = BLANK() || Zone4 = BLANK() || Zone5 = BLANK(), "#79D4F4",
                    SplitValue <= Zone1, "#CAEEFB",
                    SplitValue <= Zone2, "#79D4F4",
                    SplitValue <= Zone3, "#22B9ED",
                    SplitValue <= Zone4, "#0095C6",
                    SplitValue <= Zone5, "#007198",
                    SplitValue > Zone5, "#034359"
                ), BLANK())
            )
                
        RETURN
            ColorZones
```

---

### 6. Sport icons and metric type

#### 6.1 `SportIconUrl`

**Purpose:**
Return the **Strava-style icon URL** for a given sport type.

**Used by:**
`Sport Icon Slicer`.

```DAX
function SportIconUrl =
    (Sport: string val) =>
        VAR SportIcon =
            SWITCH (
                Sport,
                "Run",                   "https://support.strava.com/hc/article_attachments/35452502579341",
                "TrailRun",              "https://support.strava.com/hc/article_attachments/35452494454541",
                "Walk",                  "https://support.strava.com/hc/article_attachments/35452494455309",
                "Hike",                  "https://support.strava.com/hc/article_attachments/35452494456205",
                "VirtualRun",            "https://support.strava.com/hc/article_attachments/35452502579341",

                "Ride",                  "https://support.strava.com/hc/article_attachments/35452494458253",
                "MountainBikeRide",      "https://support.strava.com/hc/article_attachments/35452502587917",
                "GravelRide",            "https://support.strava.com/hc/article_attachments/35452494461197",
                "E-BikeRide",            "https://support.strava.com/hc/article_attachments/35452502593037",
                "E-MountainBikeRide",    "https://support.strava.com/hc/article_attachments/35452494465165",
                "Velomobile",            "https://support.strava.com/hc/article_attachments/35452502596365",
                "VirtualRide",           "https://support.strava.com/hc/article_attachments/35452502598157",

                "Canoe",                 "https://support.strava.com/hc/article_attachments/35452502601101",
                "Kayak",                 "https://support.strava.com/hc/article_attachments/35452502601101",
                "Kitesurf",              "https://support.strava.com/hc/article_attachments/35452494519437",
                "Rowing",                "https://support.strava.com/hc/article_attachments/35452494555021",
                "StandUpPaddling",       "https://support.strava.com/hc/article_attachments/35452502676621",
                "Surf",                  "https://support.strava.com/hc/article_attachments/35452494564109",
                "Swim",                  "https://support.strava.com/hc/article_attachments/35452502704141",
                "Windsurf",              "https://support.strava.com/hc/article_attachments/35452494579085",
                "Sail",                  "https://support.strava.com/hc/article_attachments/35452494579085",

                "IceSkate",              "https://support.strava.com/hc/article_attachments/35452502713997",
                "AlpineSki",             "https://support.strava.com/hc/article_attachments/35452502718093",
                "BackcountrySki",        "https://support.strava.com/hc/article_attachments/35452502718093",
                "NordicSki",             "https://support.strava.com/hc/article_attachments/35452494606605",
                "Snowboard",             "https://support.strava.com/hc/article_attachments/35452494607757",
                "Snowshoe",              "https://support.strava.com/hc/article_attachments/35452494609549",

                "Handcycle",             "https://support.strava.com/hc/article_attachments/35452494612365",
                "InlineSkate",           "https://support.strava.com/hc/article_attachments/35452502728077",
                "RockClimbing",          "https://support.strava.com/hc/article_attachments/35452502729101",
                "RollerSki",             "https://support.strava.com/hc/article_attachments/35452502730765",
                "Golf",                  "https://support.strava.com/hc/article_attachments/35452494619149",
                "Skateboard",            "https://support.strava.com/hc/article_attachments/35452502733453",
                "Soccer",                "https://support.strava.com/hc/article_attachments/35452494623245",
                "Wheelchair",            "https://support.strava.com/hc/article_attachments/35452494624909",
                "Badminton",             "https://support.strava.com/hc/article_attachments/35452502745101",
                "Tennis",                "https://support.strava.com/hc/article_attachments/35452494627085",
                "Pickleball",            "https://support.strava.com/hc/article_attachments/35452502751117",
                "Crossfit",              "https://support.strava.com/hc/article_attachments/35452502754573",
                "Elliptical",            "https://support.strava.com/hc/article_attachments/35452502754573",
                "StairStepper",          "https://support.strava.com/hc/article_attachments/35452502754573",
                "WeightTraining",        "https://support.strava.com/hc/article_attachments/35452502756365",
                "Yoga",                  "https://support.strava.com/hc/article_attachments/35452494637581",
                "Workout",               "https://support.strava.com/hc/article_attachments/35452502754573",
                "HIIT",                  "https://support.strava.com/hc/article_attachments/35452502762125",
                "Pilates",               "https://support.strava.com/hc/article_attachments/35452494641677",
                "TableTennis",           "https://support.strava.com/hc/article_attachments/35452502763277",
                "Squash",                "https://support.strava.com/hc/article_attachments/35452494644621",
                "Racquetball",           "https://support.strava.com/hc/article_attachments/35452502764685",
                "VirtualRowing",         "https://support.strava.com/hc/article_attachments/35452494649997",

                BLANK()
            )
        RETURN
            SportIcon
```

---

#### 6.2 `DefineSportSummaryType`

**Purpose:**
Decide whether a sport should be summarized by **distance** or by **time**.

**Used by:**
`Weekly Time/Distance`, `Weekly Time/Distance Str`, `Weekly Time/Distance Title`, `Yearly Time/Distance`.

```DAX
function DefineSportSummaryType =
    (SportName: string) =>
        VAR DistanceSports =
            {
                "Run","TrailRun","Walk","Hike","VirtualRun",
                "Ride","MountainBikeRide","GravelRide","E-BikeRide","E-MountainBikeRide","Velomobile","VirtualRide",
                "Canoe","Kayak","Kayaking","Rowing","VirtualRowing","StandUpPaddling",
                "Swim",
                "Kitesurf","Windsurf","Sail",
                "IceSkate","RollerSki","InlineSkate",
                "AlpineSki","BackcountrySki","NordicSki","Snowboard","Snowshoe",
                "Skateboard",
                "Wheelchair"
            }

        VAR TimeSports =
        {
            "WeightTraining","Crossfit","Workout","HIIT",
            "RockClimbing",
            "StairStepper","Elliptical",
            "Yoga","Pilates",
            "Golf","Surfing",
            "Squash","Tennis","TableTennis","Badminton","Pickleball","Racquetball"
        }
        
        VAR SummaryType =
            SWITCH(
                TRUE(),
                SportName IN DistanceSports, "Distance",
                SportName IN TimeSports,     "Time",
                BLANK()
            )
    
        RETURN
            SummaryType
```

---

### 7. Kudoers logic

#### 7.1 `KudoerOrderAndLabel`

**Purpose:**
Return information about **top kudo givers** per year: label, display order and min/max values.

**Used by:**
`Top Kudoers - Label`, `Top Kudoers - Order`, `Top Kudoers - Min Val`, `Top Kudoers - Max Val`.

```DAX
function KudoerOrderAndLabel =
    (ReturnValue: string val) =>
        VAR SelectedKudoer = SELECTEDVALUE('gold fact_kudos'[full_name])
        VAR AggTable =
            SUMMARIZE(
                ALLEXCEPT('gold fact_kudos', 'gold dim_calendar'[year]),
                'gold fact_kudos'[full_name],
                'gold fact_kudos'[first_name],
                "TotalKudos", DISTINCTCOUNT('gold fact_kudos'[activity_id])
            )
        VAR RankTable = 
            ADDCOLUMNS(
                AggTable,
                "KudoersRank",
                ROWNUMBER(
                    AggTable,
                    ORDERBY([TotalKudos], DESC)
                )
            )
        VAR TopRow = 
            TOPN(
                3, 
                RankTable,
                [TotalKudos],
                DESC
                )
        VAR TopRowOrder =
            ADDCOLUMNS(
                TopRow,
                "KudoersOrder",  
                SWITCH(
                    TRUE(),
                    [KudoersRank] = 1, 2,
                    [KudoersRank] = 2, 1,
                    [KudoersRank]
                )
            )
        VAR KudoerKudosValue =
            MAXX(
                FILTER(TopRowOrder, 'gold fact_kudos'[full_name] = SelectedKudoer),
                [TotalKudos]
            )
        VAR KudoerOrder =
            MAXX(
                FILTER(TopRowOrder, 'gold fact_kudos'[full_name] = SelectedKudoer),
                [KudoersOrder]
            )
        VAR KudoerName =
            MAXX(
                FILTER(TopRowOrder, 'gold fact_kudos'[full_name] = SelectedKudoer),
                'gold fact_kudos'[first_name]
            )
        VAR KudoerLabel = KudoerName & UNICHAR(10) & " - " & UNICHAR(10) & KudoerKudosValue
        VAR MaxKudosVal = MAXX(TopRow, [TotalKudos])
        VAR MinKudosVal = MINX(TopRow, [TotalKudos])
    RETURN 
        SWITCH(
            TRUE(),
            ReturnValue = "Label", KudoerLabel,
            ReturnValue = "Order", KudoerOrder,
            ReturnValue = "Min",   MinKudosVal,
            ReturnValue = "Max",   MaxKudosVal,
            BLANK()
        )
```

---

### 8. Heatmaps: daily and monthly

#### 8.1 `TotalsDailyHeatmap`

**Purpose:**
Return a value for **daily heatmaps** (day of week × day part) for a given measure.

```DAX
function TotalsDailyHeatmap =
    (TotalMeasure: expr) =>
        VAR Day = MAX('gold dim_calendar'[day_of_week])
        VAR DayPart = MAX('gold dim_time'[day_part_number])

        VAR TotalValue = 
            COALESCE(
                CALCULATE(
                    TotalMeasure,
                    KEEPFILTERS('gold dim_calendar'[day_of_week]  = Day),
                    KEEPFILTERS('gold dim_time'[day_part_number] = DayPart)
                ),
                0
            )
        RETURN
            TotalValue
```

---

#### 8.2 `TotalsMonthlyHeatmap`

**Purpose:**
Return a value for **monthly heatmaps** (year × month) for a given measure.

```DAX
function TotalsMonthlyHeatmap =
    (TotalMeasure: expr) =>
        VAR TotalValue = 
            COALESCE(
                CALCULATE(
                    TotalMeasure,
                    FILTER(
                        'gold dim_calendar',
                        'gold dim_calendar'[year] = MAX('gold dim_calendar'[year]) &&
                        'gold dim_calendar'[month] = MAX('gold dim_calendar'[month])
                    )
                ),
                0
            )
        RETURN
            TotalValue
```

---

### 9. Time-in-zone logic

#### 9.1 `TotalTimeZonePctZone2`

**Purpose:**
Compute **% of time spent in Zone 2** within a selected time window.

```DAX
function TotalTimeZonePctZone2 =
    (TotalTimeZoneTimeIntelligenceMeasure: expr) =>
        VAR TotalTime = 
            TotalTimeZoneTimeIntelligenceMeasure
        VAR TotalTimeZone2 = 
            CALCULATE(
                TotalTimeZoneTimeIntelligenceMeasure,
                KEEPFILTERS(
                    'gold fact_zones'[zone_number] = 2
                )
            )
        RETURN
            COALESCE(
                DIVIDE(
                    TotalTimeZone2,
                    TotalTime,
                    "No Zone Data"
                ),
                "No Zone Data"
            )
```

---

#### 9.2 `TotalTimeZonePrevTimeIntelligence`

**Purpose:**
Return **% Zone 2** for the **previous period** (previous 7D / 1M / 3M / 6M / YTD / 1Y).

```DAX
function TotalTimeZonePrevTimeIntelligence =
    (TimeIntelligenceType: string val) =>
        VAR TotalTime_Prev7D = 
            CALCULATE(
                TotalTimeZonePctZone2(TotalTimeZoneTimeIntelligence("7D")),
                DATEADD(
                    'gold dim_calendar'[date],
                    -7,
                    DAY
                )
            )
        VAR TotalTime_Prev1M = 
            CALCULATE(
                TotalTimeZonePctZone2(TotalTimeZoneTimeIntelligence("1M")),
                DATEADD(
                    'gold dim_calendar'[date],
                    -7 * 4,
                    DAY
                )
            )
        VAR TotalTime_Prev3M = 
            CALCULATE(
                TotalTimeZonePctZone2(TotalTimeZoneTimeIntelligence("3M")),
                DATEADD(
                    'gold dim_calendar'[date],
                    -7 * 4 * 3,
                    DAY
                )
            )
        VAR TotalTime_Prev6M = 
            CALCULATE(
                TotalTimeZonePctZone2(TotalTimeZoneTimeIntelligence("6M")),
                DATEADD(
                    'gold dim_calendar'[date],
                    -7 * 4 * 6,
                    DAY
                )
            )
        VAR TotalTime_PrevYTD = 
            CALCULATE(
                TotalTimeZonePctZone2(TotalTimeZoneTimeIntelligence("YTD")),
                DATEADD(
                    'gold dim_calendar'[date],
                    -7 * 4 * 12,
                    DAY
                )
            )
        VAR TotalTime_Prev1Y = 
            CALCULATE(
                TotalTimeZonePctZone2(TotalTimeZoneTimeIntelligence("1Y")),
                DATEADD(
                    'gold dim_calendar'[date],
                    -7 * 4 * 12,
                    DAY
                )
            )
        RETURN 
            SWITCH(
                TRUE(),
                TimeIntelligenceType = "7D", TotalTime_Prev7D,
                TimeIntelligenceType = "1M", TotalTime_Prev1M,
                TimeIntelligenceType = "3M", TotalTime_Prev3M,
                TimeIntelligenceType = "6M", TotalTime_Prev6M,
                TimeIntelligenceType = "YTD", TotalTime_PrevYTD,
                TimeIntelligenceType = "1Y", TotalTime_Prev1Y,
                BLANK()
            )
```

---

#### 9.3 `TotalTimeZoneTimeIntelligence`

**Purpose:**
Compute **total time-in-zone seconds** for a selected time window.

```DAX
function TotalTimeZoneTimeIntelligence =
    (TimeIntelligenceType: string val) =>
        VAR TotalTime_7D = 
            CALCULATE(
                [Total Time Zone (Seconds)],
                DATESINPERIOD(
                    'gold dim_calendar'[date],
                    MAX('gold dim_calendar'[date]),
                    -7,
                    DAY
                )
            )
        VAR TotalTime_1M = 
            CALCULATE(
                [Total Time Zone (Seconds)],
                DATESINPERIOD(
                    'gold dim_calendar'[date],
                    MAX('gold dim_calendar'[date]),
                    -7 * 4,
                    DAY
                )
            )
        VAR TotalTime_3M = 
            CALCULATE(
                [Total Time Zone (Seconds)],
                DATESINPERIOD(
                    'gold dim_calendar'[date],
                    MAX('gold dim_calendar'[date]),
                    -7 * 4 * 3,
                    DAY
                )
            )
        VAR TotalTime_6M = 
            CALCULATE(
                [Total Time Zone (Seconds)],
                DATESINPERIOD(
                    'gold dim_calendar'[date],
                    MAX('gold dim_calendar'[date]),
                    -7 * 4 * 6,
                    DAY
                )
            )
        VAR TotalTime_YTD = 
            CALCULATE(
                [Total Time Zone (Seconds)],
                DATESYTD(
                    'gold dim_calendar'[date]
                )
            )
        VAR TotalTime_1Y = 
            CALCULATE(
                [Total Time Zone (Seconds)],
                DATESINPERIOD(
                    'gold dim_calendar'[date],
                    MAX('gold dim_calendar'[date]),
                    -7 * 4 * 12,
                    DAY
                )
            )
        RETURN 
            SWITCH(
                TRUE(),
                TimeIntelligenceType = "7D",  TotalTime_7D,
                TimeIntelligenceType = "1M",  TotalTime_1M,
                TimeIntelligenceType = "3M",  TotalTime_3M,
                TimeIntelligenceType = "6M",  TotalTime_6M,
                TimeIntelligenceType = "YTD", TotalTime_YTD,
                TimeIntelligenceType = "1Y",  TotalTime_1Y,
                BLANK()
            )
```

---

### 10. Yearly totals (all sports vs running only)

You have **two UDFs** for yearly metrics:

* **`YearlyTotals`** – works across **all sports**,
* **`YearlyRunningTotals`** – focuses on **running only**.

Both share the same contract:

* **`SelectedMeasure`**:

  * 0 = Time, 1 = Distance, 2 = Effort, 3 = Activities
* **`SummaryType`**:

  * `"Curr"`, `"Prev"`, `"Diff"`, `"YTD"`
* **`SummaryFormat`**:

  * `""` → numeric,
  * `"Str"` → formatted string with arrows / units.

They encapsulate a lot of logic used by your **Yearly Summary** and **Running Rewind** pages (YTD, YoY comparison, differences with arrows, etc.).

#### 10.1 `YearlyRunningTotals`

```DAX
function YearlyRunningTotals =
    (SelectedMeasure: int64 val, SummaryType: string val, SummaryFormat: string val) =>
        VAR SelectedYear = SELECTEDVALUE('gold dim_calendar'[year])
        
        VAR PrevYear = SelectedYear - 1
        
        VAR DoesPrevYearExists = 
            CONTAINS(
                ALL('gold dim_calendar'),
                'gold dim_calendar'[year], 
                PrevYear)
                
        VAR TotalMonthlyRunningActivities = 
            COALESCE(
                CALCULATE(
                    [Total Activities],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    )
                ), 0
            )
            
        VAR TotalMonthlyRunningActivitiesYTD = 
            COALESCE(
                CALCULATE(
                    [Total Activities],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )
            
        VAR TotalMonthlyRunningEffort = 
            COALESCE(
                CALCULATE(
                    [Total Effort],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    )
                ), 0
            )
            
        VAR TotalMonthlyRunningEffortYTD = 
            COALESCE(
                CALCULATE(
                    [Total Effort],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )
            
        VAR TotalMonthlyRunningDistanceMeters = 
            COALESCE(
                CALCULATE(
                    [Total Distance (Meters)],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    )
                ), 0
            )
            
        VAR TotalMonthlyRunningDistanceMetersYTD = 
            COALESCE(
                CALCULATE(
                    [Total Distance (Meters)],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )
            
        VAR TotalMonthlyRunningTimeSeconds = 
            COALESCE(
                CALCULATE(
                    [Total Time (Seconds)],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    )
                ), 0
            )
            
        VAR TotalMonthlyRunningTimeSecondsYTD = 
            COALESCE(
                CALCULATE(
                    [Total Time (Seconds)],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )

        VAR PrevYearTotalMonthlyRunningActivities =
            IF(DoesPrevYearExists,
                CALCULATE(
                    [Total Activities],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    FILTER(
                        ALL('gold dim_calendar'[year]),
                        'gold dim_calendar'[year] = PrevYear
                    )
                ),
                BLANK()
            )
            
        VAR PrevYearTotalMonthlyRunningEffort =
            IF(DoesPrevYearExists,
                CALCULATE(
                    [Total Effort],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    FILTER(
                        ALL('gold dim_calendar'[year]),
                        'gold dim_calendar'[year] = PrevYear
                    )
                ),
                BLANK()
            )
            
        VAR PrevYearTotalMonthlyRunningDistanceMeters =
            IF(DoesPrevYearExists,
                CALCULATE(
                    [Total Distance (Meters)],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    FILTER(
                        ALL('gold dim_calendar'[year]),
                        'gold dim_calendar'[year] = PrevYear
                    )
                ),
                BLANK()
            )
        
        VAR PrevYearTotalMonthlyRunningTimeSeconds = 
            IF(DoesPrevYearExists,
                CALCULATE(
                    [Total Time (Seconds)],
                    KEEPFILTERS(
                        'gold dim_sport_type'[sport_type] = "Run"
                    ),
                    FILTER(
                        ALL('gold dim_calendar'[year]),
                        'gold dim_calendar'[year] = PrevYear
                    )
                ),
                BLANK()
            )
            
        VAR TotalMonthlyRunningReturnVal =
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", SecondsToDurationText(TotalMonthlyRunningTimeSeconds), DIVIDE(TotalMonthlyRunningTimeSeconds, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", MetersToDistanceText(TotalMonthlyRunningDistanceMeters), DIVIDE(TotalMonthlyRunningDistanceMeters, 1000)),
                SelectedMeasure = 2, TotalMonthlyRunningEffort,
                SelectedMeasure = 3, TotalMonthlyRunningActivities,
                BLANK()
            )
            
        VAR PrevYearTotalMonthlyRunningReturnVal =
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", SecondsToDurationText(PrevYearTotalMonthlyRunningTimeSeconds), DIVIDE(PrevYearTotalMonthlyRunningTimeSeconds, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", MetersToDistanceText(PrevYearTotalMonthlyRunningDistanceMeters), DIVIDE(PrevYearTotalMonthlyRunningDistanceMeters, 1000)),
                SelectedMeasure = 2, PrevYearTotalMonthlyRunningEffort,
                SelectedMeasure = 3, PrevYearTotalMonthlyRunningActivities,
                BLANK()
            )
            
        VAR YearlyDiffTotalMonthlyRunningTimeReturnStr = 
            IF((TotalMonthlyRunningTimeSeconds - PrevYearTotalMonthlyRunningTimeSeconds) > 0,
                UNICHAR(8593) & " " & SecondsToDurationText(ABS(TotalMonthlyRunningTimeSeconds - PrevYearTotalMonthlyRunningTimeSeconds)),
                IF((TotalMonthlyRunningTimeSeconds - PrevYearTotalMonthlyRunningTimeSeconds) < 0,
                    UNICHAR(8595) & " " & SecondsToDurationText(ABS(TotalMonthlyRunningTimeSeconds - PrevYearTotalMonthlyRunningTimeSeconds)),
                    SecondsToDurationText(ABS(TotalMonthlyRunningTimeSeconds - PrevYearTotalMonthlyRunningTimeSeconds))))
                    
        VAR YearlyDiffTotalMonthlyRunningDistanceReturnStr = 
            IF((TotalMonthlyRunningDistanceMeters - PrevYearTotalMonthlyRunningDistanceMeters) > 0,
                UNICHAR(8593) & " " & MetersToDistanceText(ABS(TotalMonthlyRunningDistanceMeters - PrevYearTotalMonthlyRunningDistanceMeters)),
                IF((TotalMonthlyRunningDistanceMeters - PrevYearTotalMonthlyRunningDistanceMeters) < 0,
                    UNICHAR(8595) & " " & MetersToDistanceText(ABS(TotalMonthlyRunningDistanceMeters - PrevYearTotalMonthlyRunningDistanceMeters)),
                    MetersToDistanceText(ABS(TotalMonthlyRunningDistanceMeters - PrevYearTotalMonthlyRunningDistanceMeters))))
                    
        VAR YearlyDiffTotalMonthlyRunningEffortReturnStr = 
            IF((TotalMonthlyRunningEffort - PrevYearTotalMonthlyRunningEffort) > 0,
                UNICHAR(8593) & " " & ABS(TotalMonthlyRunningEffort - PrevYearTotalMonthlyRunningEffort),
                IF((TotalMonthlyRunningEffort - PrevYearTotalMonthlyRunningEffort) < 0,
                    UNICHAR(8595) & " " & ABS(TotalMonthlyRunningEffort - PrevYearTotalMonthlyRunningEffort),
                    ABS(TotalMonthlyRunningEffort - PrevYearTotalMonthlyRunningEffort)))
                    
        VAR YearlyDiffTotalMonthlyRunningActivitiesReturnStr = 
            IF((TotalMonthlyRunningActivities - PrevYearTotalMonthlyRunningActivities) > 0,
                UNICHAR(8593) & " " & ABS(TotalMonthlyRunningActivities - PrevYearTotalMonthlyRunningActivities),
                IF((TotalMonthlyRunningActivities - PrevYearTotalMonthlyRunningActivities) < 0,
                    UNICHAR(8595) & " " & ABS(TotalMonthlyRunningActivities - PrevYearTotalMonthlyRunningActivities),
                    ABS(TotalMonthlyRunningActivities - PrevYearTotalMonthlyRunningActivities)))        
        
        VAR YearlyDiffTotalMonthlyRunningReturnVal = 
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyRunningTimeReturnStr, DIVIDE(TotalMonthlyRunningTimeSeconds - PrevYearTotalMonthlyRunningTimeSeconds, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyRunningDistanceReturnStr, DIVIDE(TotalMonthlyRunningDistanceMeters - PrevYearTotalMonthlyRunningDistanceMeters, 1000)),
                SelectedMeasure = 2, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyRunningEffortReturnStr, TotalMonthlyRunningEffort - PrevYearTotalMonthlyRunningEffort),
                SelectedMeasure = 3, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyRunningActivitiesReturnStr, TotalMonthlyRunningActivities - PrevYearTotalMonthlyRunningActivities),
                BLANK()
                )
                
        VAR YTDTotalMonthlyRunningReturnVal =
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", SecondsToDurationText(TotalMonthlyRunningTimeSecondsYTD), DIVIDE(TotalMonthlyRunningTimeSecondsYTD, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", MetersToDistanceText(TotalMonthlyRunningDistanceMetersYTD), DIVIDE(TotalMonthlyRunningDistanceMetersYTD, 1000)),
                SelectedMeasure = 2, TotalMonthlyRunningEffortYTD,
                SelectedMeasure = 3, TotalMonthlyRunningActivitiesYTD,
                BLANK()
            )
            
        RETURN
            SWITCH(
                TRUE(),
                SummaryType = "Curr", TotalMonthlyRunningReturnVal,
                SummaryType = "Prev", PrevYearTotalMonthlyRunningReturnVal,
                SummaryType = "Diff", YearlyDiffTotalMonthlyRunningReturnVal,
                SummaryType = "YTD",  YTDTotalMonthlyRunningReturnVal,
                BLANK()
            )
```

---

#### 10.2 `YearlyTotals`

**Purpose:**
Same as above, but for **all sports** (not only running). This UDF powers most of the **Yearly Summary** visuals.

```DAX
function YearlyTotals =
    (SelectedMeasure: int64 val, SummaryType: string val, SummaryFormat: string val) =>
        VAR SelectedYear = SELECTEDVALUE('gold dim_calendar'[year])
        
        VAR PrevYear = SelectedYear - 1
        
        VAR DoesPrevYearExists = 
            CONTAINS(
                ALL('gold dim_calendar'),
                'gold dim_calendar'[year], 
                PrevYear)
                
        VAR TotalMonthlyActivities = 
            COALESCE([Total Activities], 0
            )
            
        VAR TotalMonthlyActivitiesYTD = 
            COALESCE(
                CALCULATE(
                    [Total Activities],
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )
            
        VAR TotalMonthlyEffort = 
            COALESCE([Total Effort], 0
            )
            
        VAR TotalMonthlyEffortYTD = 
            COALESCE(
                CALCULATE(
                    [Total Effort],
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )
            
        VAR TotalMonthlyDistanceMeters = 
            COALESCE([Total Distance (Meters)], 0
            )
            
        VAR TotalMonthlyDistanceMetersYTD = 
            COALESCE(
                CALCULATE(
                    [Total Distance (Meters)],
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )
            
        VAR TotalMonthlyTimeSeconds = 
            COALESCE([Total Time (Seconds)], 0
            )
            
        VAR TotalMonthlyTimeSecondsYTD = 
            COALESCE(
                CALCULATE(
                    [Total Time (Seconds)],
                    DATESYTD(
                        'gold dim_calendar'[date]
                    )
                ), 0
            )

        VAR PrevYearTotalMonthlyActivities =
            IF(DoesPrevYearExists,
                COALESCE(
                    CALCULATE(
                        [Total Activities],
                        FILTER(
                            ALL('gold dim_calendar'[year]),
                            'gold dim_calendar'[year] = PrevYear
                        )
                ), 0),
                BLANK()
            )
            
        VAR PrevYearTotalMonthlyEffort =
            IF(DoesPrevYearExists,
                COALESCE(
                    CALCULATE(
                        [Total Effort],
                        FILTER(
                            ALL('gold dim_calendar'[year]),
                            'gold dim_calendar'[year] = PrevYear
                        )
                    ), 0),
                BLANK()
            )
            
        VAR PrevYearTotalMonthlyDistanceMeters =
            IF(DoesPrevYearExists,
                COALESCE(
                    CALCULATE(
                        [Total Distance (Meters)],
                        FILTER(
                            ALL('gold dim_calendar'[year]),
                            'gold dim_calendar'[year] = PrevYear
                        )
                    ), 0),
                BLANK()
            )
        
        VAR PrevYearTotalMonthlyTimeSeconds = 
            IF(DoesPrevYearExists,
                COALESCE(
                    CALCULATE(
                        [Total Time (Seconds)],
                        FILTER(
                            ALL('gold dim_calendar'[year]),
                            'gold dim_calendar'[year] = PrevYear
                        )
                    ), 0),
                BLANK()
            )
            
        VAR TotalMonthlyReturnVal =
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", SecondsToDurationText(TotalMonthlyTimeSeconds), DIVIDE(TotalMonthlyTimeSeconds, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", MetersToDistanceText(TotalMonthlyDistanceMeters), DIVIDE(TotalMonthlyDistanceMeters, 1000)),
                SelectedMeasure = 2, TotalMonthlyEffort,
                SelectedMeasure = 3, TotalMonthlyActivities,
                BLANK()
            )
            
        VAR PrevYearTotalMonthlyReturnVal =
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", SecondsToDurationText(PrevYearTotalMonthlyTimeSeconds), DIVIDE(PrevYearTotalMonthlyTimeSeconds, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", MetersToDistanceText(PrevYearTotalMonthlyDistanceMeters), DIVIDE(PrevYearTotalMonthlyDistanceMeters, 1000)),
                SelectedMeasure = 2, PrevYearTotalMonthlyEffort,
                SelectedMeasure = 3, PrevYearTotalMonthlyActivities,
                BLANK()
            )
            
        VAR YearlyDiffTotalMonthlyTimeReturnStr = 
            IF((TotalMonthlyTimeSeconds - PrevYearTotalMonthlyTimeSeconds) > 0,
                UNICHAR(8593) & " " & SecondsToDurationText(ABS(TotalMonthlyTimeSeconds - PrevYearTotalMonthlyTimeSeconds)),
                IF((TotalMonthlyTimeSeconds - PrevYearTotalMonthlyTimeSeconds) < 0,
                    UNICHAR(8595) & " " & SecondsToDurationText(ABS(TotalMonthlyTimeSeconds - PrevYearTotalMonthlyTimeSeconds)),
                    SecondsToDurationText(ABS(TotalMonthlyTimeSeconds - PrevYearTotalMonthlyTimeSeconds))))
                    
        VAR YearlyDiffTotalMonthlyDistanceReturnStr = 
            IF((TotalMonthlyDistanceMeters - PrevYearTotalMonthlyDistanceMeters) > 0,
                UNICHAR(8593) & " " & MetersToDistanceText(ABS(TotalMonthlyDistanceMeters - PrevYearTotalMonthlyDistanceMeters)),
                IF((TotalMonthlyDistanceMeters - PrevYearTotalMonthlyDistanceMeters) < 0,
                    UNICHAR(8595) & " " & MetersToDistanceText(ABS(TotalMonthlyDistanceMeters - PrevYearTotalMonthlyDistanceMeters)),
                    MetersToDistanceText(ABS(TotalMonthlyDistanceMeters - PrevYearTotalMonthlyDistanceMeters))))
                    
        VAR YearlyDiffTotalMonthlyEffortReturnStr = 
            IF((TotalMonthlyEffort - PrevYearTotalMonthlyEffort) > 0,
                UNICHAR(8593) & " " & ABS(TotalMonthlyEffort - PrevYearTotalMonthlyEffort),
                IF((TotalMonthlyEffort - PrevYearTotalMonthlyEffort) < 0,
                    UNICHAR(8595) & " " & ABS(TotalMonthlyEffort - PrevYearTotalMonthlyEffort),
                    ABS(TotalMonthlyEffort - PrevYearTotalMonthlyEffort)))
                    
        VAR YearlyDiffTotalMonthlyActivitiesReturnStr = 
            IF((TotalMonthlyActivities - PrevYearTotalMonthlyActivities) > 0,
                UNICHAR(8593) & " " & ABS(TotalMonthlyActivities - PrevYearTotalMonthlyActivities),
                IF((TotalMonthlyActivities - PrevYearTotalMonthlyActivities) < 0,
                    UNICHAR(8595) & " " & ABS(TotalMonthlyActivities - PrevYearTotalMonthlyActivities),
                    ABS(TotalMonthlyActivities - PrevYearTotalMonthlyActivities)))        
        
        VAR YearlyDiffTotalMonthlyReturnVal = 
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyTimeReturnStr, DIVIDE(TotalMonthlyTimeSeconds - PrevYearTotalMonthlyTimeSeconds, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyDistanceReturnStr, DIVIDE(TotalMonthlyDistanceMeters - PrevYearTotalMonthlyDistanceMeters, 1000)),
                SelectedMeasure = 2, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyEffortReturnStr, TotalMonthlyEffort - PrevYearTotalMonthlyEffort),
                SelectedMeasure = 3, IF(SummaryFormat = "Str", YearlyDiffTotalMonthlyActivitiesReturnStr, TotalMonthlyActivities - PrevYearTotalMonthlyActivities),
                BLANK()
                )
                
        VAR YTDTotalMonthlyReturnVal =
            SWITCH(
                TRUE(),
                SelectedMeasure = 0, IF(SummaryFormat = "Str", SecondsToDurationText(TotalMonthlyTimeSecondsYTD), DIVIDE(TotalMonthlyTimeSecondsYTD, 3600)),
                SelectedMeasure = 1, IF(SummaryFormat = "Str", MetersToDistanceText(TotalMonthlyDistanceMetersYTD), DIVIDE(TotalMonthlyDistanceMetersYTD, 1000)),
                SelectedMeasure = 2, TotalMonthlyEffortYTD,
                SelectedMeasure = 3, TotalMonthlyActivitiesYTD,
                BLANK()
            )
            
        RETURN
            SWITCH(
                TRUE(),
                SummaryType = "Curr", TotalMonthlyReturnVal,
                SummaryType = "Prev", PrevYearTotalMonthlyReturnVal,
                SummaryType = "Diff", YearlyDiffTotalMonthlyReturnVal,
                SummaryType = "YTD",  YTDTotalMonthlyReturnVal,
                BLANK()
            )
```

---

If you want, next step I can do a similar **English block for the measures themselves** (grouped by folder) so your README has both **UDFs** and **measure documentation** side by side.
