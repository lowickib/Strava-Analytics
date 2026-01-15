## API Ingestion & Bronze Layer

### **Overview**

This notebook establishes the **Bronze Layer** for a Strava-based analytics pipeline.
It focuses on **reliable data extraction** from the **Strava API**, implementing **incremental ingestion**, **rate-limit-aware design**, and **robust persistence** into a **PostgreSQL database**.

The Bronze Layer is designed to be **lossless**, **auditable**, and **replayable** — serving as the single source of truth for downstream transformation layers (Silver/Gold) and BI tools like Power BI.

---

### Objective

Build a **production-ready ingestion workflow** that:

- Extracts activity data from the Strava API using **OAuth2 authentication**
- Supports **incremental synchronization** (backfill + rolling refresh)
- Handles **rate limits and retries** gracefully
- Stores tables with **minimal transformation**, preserving selected nested structures as **JSONB** in PostgreSQL


---

### What the Bronze Layer Captures

The Bronze Layer stores **API responses with minimal transformation**, prioritizing:
- traceability (raw payload availability),
- reproducibility,
- ability to reprocess into curated Silver tables later.

The ingestion covers:
- **Activity list (summary)** for discovery and incremental loads
- **Activity details** (full activity metadata)
- **Kudos** (social engagement signal)
- **Zones** (heart-rate / power zone distributions, when available)
- **Segment details** (segment-level context used by some activities)

---

### Data source

All ingested data comes from the **Strava API v3** endpoints described in Strava’s official API [reference](https://developers.strava.com/docs/reference/).



| Category             | Endpoint                     | Description                                               |
| -------------------- | ---------------------------- | --------------------------------------------------------- |
| **Auth**             | `POST /oauth/token`          | Refresh access tokens using a stored refresh token        |
| **Activities List**  | `GET /athlete/activities`    | Retrieve recent athlete activities (pagination supported) |
| **Activity Details** | `GET /activities/{id}`       | Retrieve detailed metrics for each activity               |
| **Kudos**            | `GET /activities/{id}/kudos` | Capture social engagement data                            |
| **Zones**            | `GET /activities/{id}/zones` | Capture heart rate, pace, and power zone distributions    |
| **Segments**         | `GET /segments/{id}`         | Capture metadata for referenced segments                  |

**Authentication** uses the OAuth2 refresh-token flow (client ID, client secret, and refresh token securely stored in `.env`).

---

### Rate limits and how the notebook mitigates them

Strava enforces per-application request limits. The default **“non-upload”** rate limit is:
- **100 requests per 15 minutes**
- **up to 1,000 requests per day**

To operate within these constraints, the notebook implements:
- **HTTP 429 handling** (rate limiting), including honoring `Retry-After` when present.
- **Retry/backoff with jitter** for 429, 5xx, and network exceptions.
- A conservative per-request **sleep (7–9 seconds)** during per-activity calls (details/kudos/zones) to reduce burstiness and avoid hitting short-window limits.

---


### Tech stack (as used in the notebook)

- **Python**
- **requests** (incl. `Session` + Authorization headers)
- **pandas** (`json_normalize`, `explode`, transformations, empty-frame guards)
- **SQLAlchemy** (engine, `text()`, `inspect()`, `to_sql()`, column types)
- **PostgreSQL** (schema/tables, staging tables, **ON CONFLICT DO UPDATE**)
- **dotenv**, **logging**, **tqdm**
- Standard libs: **time**, **random**, **datetime**, **json**, **os**



---

### Pipeline overview (what the notebook implements)

#### 1) Imports and global setup
- Load `.env`, configure logging, set pandas options.
- `UPDATE_SEGMENT` flag controls whether segment details are fetched.

#### 2) PostgreSQL connectivity
- Create SQLAlchemy engine with pooling (`pool_pre_ping=True`, `pool_size`, `max_overflow`).
- Ensure schema exists: `CREATE SCHEMA IF NOT EXISTS bronze` (based on `TARGET_B_SCHEMA`).

#### 3) OAuth2 authentication
- `get_access_token(...)` retrieves an `access_token` via POST and validates the response.

#### 4) HTTP layer: retries + rate limiting
- `get_json_with_retry(...)` implements retries for **429**, **5xx**, and network errors, with backoff + jitter and JSON parse handling.

#### 5) Activities list ingestion (pagination)
- `fetch_all_activities(...)` iterates `page` and `per_page` until empty payload or `MAX_PAGES`.
- Normalization via `pd.json_normalize(..., sep='_')`.

#### 6) Persist `bronze.activities` (full overwrite)
- Uses explicit `dtype` mapping (incl. JSONB only where applicable).
- Writes using `to_sql(..., if_exists="replace", method="multi", chunksize=1000)`.

#### 7) Determine which activities to (re)download: missing + last 30 days
The notebook downloads:
- **Missing activities** (present in `bronze.activities` but absent in `bronze.activities_details`),
- **AND activities from the last 30 days** (`REFRESH_THRESHOLD_DAYS = 30`).

Rationale (explicit design choice in this notebook):
- In recent activities, fields may change after the initial ingestion — **most commonly kudos_count**, but also edits like **activity description** or updates to **workout/activity type** — so the notebook refreshes the latest 30 days to keep `activities_details` consistent with the source of truth.

#### 8) Fetch + persist `bronze.activities_details` (UPSERT)
- Per-activity downloads using the activity detail endpoint.
- Applies a 7–9s delay between calls.
- Persists via:
  - staging table,
  - `INSERT ... ON CONFLICT ("id") DO UPDATE ...`,
  - staging cleanup.

#### 9) Sanity check
- Compares `COUNT(*)` between `bronze.activities` and `bronze.activities_details`.

#### 10) Fetch + persist `bronze.kudos` (UPSERT)
- Fetches kudos only where `kudos_count > 0` and the activity is missing in `bronze.kudos`.
- Enriches each kudos row with `activity_id` + generated `kudos_id`.
- Persists via staging + `ON CONFLICT DO UPDATE`.

#### 11) Fetch + persist `bronze.activities_zones` (UPSERT)
- Fetches zones for activities and handles missing/unexpected responses by creating a fallback row.
- Persists via staging + `ON CONFLICT DO UPDATE`.

#### 12) Segment details enrichment (running only)
Segment metadata is fetched **only for running activities**, implemented as:
- extracting `segment_efforts` from `bronze.activities_details`,
- exploding + normalizing nested JSON,
- filtering to `segment.activity_type == 'Run'`,
- deduplicating segment IDs before fetching details.

Rationale (explicit design choice in this notebook):
- **Running segments are fewer**, and segment detail calls can quickly consume the **daily request budget**, so limiting this enrichment to running keeps it feasible under the strict daily rate limit.
- Additionally, the **downstream analysis is primarily focused on running activities**, so prioritizing running segments aligns the enrichment scope with the analytical goals of the project.

---


### **Persistence Layer (PostgreSQL — Bronze Tables)**

All data is stored in a dedicated PostgreSQL schema defined by `TARGET_B_SCHEMA` (typically `bronze`).
Each domain object is stored in a separate table (names are parameterized via `.env`):

| Table (env var)                    | Grain / Rows                          | Primary key (as implemented)                 | Notes |
| ---------------------------------- | ------------------------------------- | -------------------------------------------- | ----- |
| `ACTIVITIES_B_TABLE`               | 1 row per activity                     | `id` (BIGINT)                                 | Full overwrite (`replace`) |
| `DETAILS_B_TABLE`                  | 1 row per activity                     | `id` (BIGINT)                                 | UPSERT via staging + `ON CONFLICT (id)` |
| `KUDOS_B_TABLE`                    | many rows per activity                 | `id` (TEXT)                                   | `id = "{activity_id}-{kudos_id}"`, UPSERT |
| `ZONES_B_TABLE`                    | many rows per activity (by zone type)  | `id` (TEXT)                                   | `id = "{activity_id}-{type}"`, UPSERT |
| `SEGMENTS_DETAILS_B_TABLE`         | 1 row per segment                      | `id` (BIGINT)                                 | Overwrite (`replace`) when enabled |


Nested structures are preserved where useful:
- `activities_details` keeps selected nested objects/lists (e.g., `segment_efforts`, `splits_*`, `laps`, `best_efforts`, `stats_visibility`) as **JSONB**.
- `activities_zones` stores `distribution_buckets` as **JSONB**.

---

### Bronze data model & load behavior

#### `bronze.activities`
- load type: **replace** (full overwrite)

#### `bronze.activities_details`
- key: **id (BIGINT)**
- load type: **UPSERT** via staging + `ON CONFLICT ("id") DO UPDATE`

#### `bronze.kudos`
- key: **id (TEXT)** + `activity_id`
- load type: **UPSERT** via staging + `ON CONFLICT ("id") DO UPDATE`

#### `bronze.activities_zones`
- key: **id (TEXT)** + `activity_id`
- load type: **UPSERT** via staging + `ON CONFLICT ("id") DO UPDATE`

#### `bronze.segments_details`
- load type: **replace** (full overwrite) when `UPDATE_SEGMENT = True`

---


### Reliability & Engineering Practices Demonstrated

#### Authentication handling
- OAuth token refresh is implemented using Strava’s recommended refresh flow and response structure.

#### Rate-limit awareness
- Request patterns are designed to stay compatible with Strava’s published limits (non-upload).
- The pipeline limits unnecessary refresh by combining **backfill + 30-day rolling refresh**.

#### API pagination & batching
- Activity discovery endpoints are consumed in a way that supports pagination (crucial for long history).
- Downstream calls (details/kudos/zones) are driven by activity IDs, which is a scalable pattern.

#### Data lineage & auditability
- Bronze stores payloads in a way that preserves “what Strava said at ingestion time”.
- Data is **replayable** by re-running ingestion; note that the activities table is maintained as a **current snapshot** via overwrite.

---

### Sample API responses

Below are **representative excerpts** aligned with Strava’s documentation examples.

#### 1) Refresh token response — `POST /oauth/token`
```json
{
  "token_type": "Bearer",
  "access_token": "<short_lived_access_token>",
  "expires_at": 1768426399,
  "expires_in": 21600,
  "refresh_token": "<new_refresh_token>"
}

```


---

#### 2) Activities list — `GET /athlete/activities`

```json
[
  {
    "resource_state": 2,
    "athlete": {
      "id": 81055898,
      "resource_state": 1
    },
    "name": "Afternoon Walk",
    "distance": 9518.1,
    "moving_time": 6881,
    "elapsed_time": 7501,
    "total_elevation_gain": 13.0,
    "type": "Walk",
    "sport_type": "Walk",
    "device_name": "Garmin Forerunner 970",
    "id": 16957346921,
    "start_date": "2026-01-06T12:51:48Z",
    "start_date_local": "2026-01-06T13:51:48Z",
    "timezone": "(GMT+01:00) Europe/Warsaw",
    "utc_offset": 3600.0,
    "location_city": null,
    "location_state": null,
    "location_country": null,
    "achievement_count": 0,
    "kudos_count": 9,
    "comment_count": 0,
    "athlete_count": 4,
    "photo_count": 0,
    "map": {
      "id": "a16957346921",
      "summary_polyline": "wm}vHacggBi@Lu@Nq@VQEMQSI[Ki@K_@@k@VSDO@q@[c@[G?SPF~@b@pDV`BVfAXd@X^r@t@ZFHDJCb@BRCBBFf@@VCNE}@EG_@@eBMe@?q@HQFa@CWQy@u@GKEQAe@PkAE_@Oe@QWMIgAWMFULo@f@_@b@Yd@a@rA[bBS|@iChEc@nAMLCAISA{@DgFH}A@gAGiBG_AOgB{@iGYkAWm@Ye@AWt@{FTcC@m@GgC]yES{@]}@WcAgAkIMgBBOFANH|@v@VZd@ZhAhAjA~@zA|AJThCrYHlCZhECjBFp@H`@Dz@Ab@O~@U`CId@Qt@CZhAh@v@X~Al@r@RPJRXjBv@RXFTP`ANpA?zAJnBCpAa@dFFTFDlDJvAE|ABlEGV@HNRp@NGPSNCj@@j@FPCVk@f@m@HQf@oBF]n@}BNaALg@JSTSf@[XKt@QNKVw@l@_CNYNKXIJOr@uBPy@Di@Zi@T}@Lw@JwBlAcPHaB?c@AOYS?GAD?BBCLDFGA]BSJGPa@PEv@LdAFRDDB@RIhB?j@Et@G`@BNLJj@PPH\\Zj@p@xG|IfAnAbAt@PVPh@Jz@_@lBYhAaAzC}@`DkClHg@~Ak@~CQlAy@dF]|A{ApEWl@Uz@gEbKEJQJME[OsEwCy@m@e@W]MoAMi@S}@SUMe@e@IMN_AnAsFt@iCHSb@q@`@mAjBkHj@iCb@{A^eARqA`@iB",
      "resource_state": 2
    },
    "trainer": false,
    "commute": false,
    "manual": false,
    "private": false,
    "visibility": "followers_only",
    "flagged": false,
    "gear_id": null,
    "start_latlng": [
      51.104475,
      17.084093
    ],
    "end_latlng": [
      51.105202,
      17.084899
    ],
    "average_speed": 1.383,
    "max_speed": 3.138,
    "average_cadence": 51.2,
    "has_heartrate": true,
    "average_heartrate": 102.4,
    "max_heartrate": 119.0,
    "heartrate_opt_out": false,
    "display_hide_heartrate_option": true,
    "elev_high": 120.4,
    "elev_low": 111.2,
    "upload_id": 18050722067,
    "upload_id_str": "18050722067",
    "external_id": "garmin_ping_520683949482",
    "from_accepted_tag": false,
    "pr_count": 0,
    "total_photo_count": 0,
    "has_kudoed": false,
    "suffer_score": 17.0
  },
  {
    "resource_state": 2,
    "athlete": {
      "id": 81055898,
      "resource_state": 1
    },
    "name": "Rolling 400s🔥",
    "distance": 11315.1,
    "moving_time": 3700,
    "elapsed_time": 3700,
    "total_elevation_gain": 21.0,
    "type": "Run",
    "sport_type": "Run",
    "workout_type": 3,
    "device_name": "Garmin Forerunner 970",
    "id": 16954650847,
    "start_date": "2026-01-06T10:03:04Z",
    "start_date_local": "2026-01-06T11:03:04Z",
    "timezone": "(GMT+01:00) Europe/Warsaw",
    "utc_offset": 3600.0,
    "location_city": null,
    "location_state": null,
    "location_country": null,
    "achievement_count": 5,
    "kudos_count": 9,
    "comment_count": 0,
    "athlete_count": 1,
    "photo_count": 0,
    "map": {
      "id": "a16954650847",
      "summary_polyline": "sc}vH_yngBoA~BuArBS`@sD`GWf@qAnBiCfEUXoBfDu@hAcAhByC~EyBnCqAhBqBhCIR@NHXPb@lCpF`BpDpAbCd@l@Nf@Hv@?j@O`BBr@A^[dFM~@?bAk@zHStBBhAD|GF~DEhAF|HBpAHp@HvMHtEClABzB@RP^Rt@JxAFPD@tBSv@Cl@O^Dp@@^CZI`@JfBDv@AnBNhAB|@HHAdAF|@@`@Fp@@^BXAVGJ@\\L~CLj@JF?BEBGAk@Fq@EsDBg@@xEGn@?n@AFE@[Es@CgAIq@Ae@MK?ODO@a@?cAG[EuEMq@Gi@?yAMS?u@CkA?e@M_@JY@k@?_@GK?g@LgBNaABN?z@IfAEZEZIJAZF\\A\\@NA^K^HfDFj@FpFVt@Fh@CVDnBF`@Il@LjADbCNBC@OAk@Hk@AsCAF@fCGr@?r@IHe@GaDMg@M]Fe@?aBKiACiAIcCEsBOgAEmA?g@ME?YJ]B[CU@OEUAYJWFe@?u@HaA@ABH@JE\\CJCfBIh@M`@Dz@@VA\\KXHbEHnAJxFTR?lBJ`@@\\IJ?TJH@hAFd@FTA^@b@FJC@G?k@Fs@CqCBjCGn@?p@ELG@UGw@AmBIQAa@Me@F_CMaBCWEkACa@EeDQyCCc@K_@H{@Do@IQBYJg@AmAJu@LMAEEKmAIc@_@cAIqB?a@Fu@?i@EwCK}NK{@C}AGkHFq@?YMmJ?qAIqCLwB@INIDW\\yE@WGWAO\\sD@g@JqAFuALoABq@Gq@Su@y@kA_C}Ee@y@IQAODQd@_AjBkC~@cB|@uA|ByDdAaBj@cAr@gAxDwGlEgHlAuBfBsCZk@JKt@qAp@aA|CaF`@k@xBeDzAqBxBgDfDyEv@_BZc@T{@A]u@wBGEMAYFKFk@n@",
      "resource_state": 2
    },
    "trainer": false,
    "commute": false,
    "manual": false,
    "private": false,
    "visibility": "everyone",
    "flagged": false,
    "gear_id": "g23642256",
    "start_latlng": [
      51.107753,
      17.123849
    ],
    "end_latlng": [
      51.106867,
      17.123285
    ],
    "average_speed": 3.058,
    "max_speed": 4.18,
    "average_cadence": 85.2,
    "average_watts": 350.8,
    "max_watts": 564,
    "weighted_average_watts": 356,
    "device_watts": true,
    "kilojoules": 1298.3,
    "has_heartrate": true,
    "average_heartrate": 160.7,
    "max_heartrate": 180.0,
    "heartrate_opt_out": false,
    "display_hide_heartrate_option": true,
    "elev_high": 125.0,
    "elev_low": 113.2,
    "upload_id": 18048002093,
    "upload_id_str": "18048002093",
    "external_id": "garmin_ping_520613910011",
    "from_accepted_tag": false,
    "pr_count": 2,
    "total_photo_count": 0,
    "has_kudoed": false,
    "suffer_score": 115.0
  }
]

```


---

### 3) Activity details — `GET /activities/{id}`
```json
{
  "resource_state": 3,
  "athlete": {
    "id": 81055898,
    "resource_state": 1
  },
  "name": "Rolling 400s🔥",
  "distance": 11315.1,
  "moving_time": 3700,
  "elapsed_time": 3700,
  "total_elevation_gain": 21.0,
  "type": "Run",
  "sport_type": "Run",
  "workout_type": 3,
  "device_name": "Garmin Forerunner 970",
  "id": 16954650847,
  "start_date": "2026-01-06T10:03:04Z",
  "start_date_local": "2026-01-06T11:03:04Z",
  "timezone": "(GMT+01:00) Europe/Warsaw",
  "utc_offset": 3600.0,
  "location_city": null,
  "location_state": null,
  "location_country": null,
  "achievement_count": 5,
  "kudos_count": 9,
  "comment_count": 0,
  "athlete_count": 1,
  "photo_count": 0,
  "map": {
    "id": "a16954650847",
    "polyline": "m~|vH_oogBYXCLBNZbA?NE`@Fb@Nl@h@rA@PCXQb@kAtAg@p@i@z@_AnBQXo@lAuArBS`@sD`GWf@qAnBiCfEUXoBfDu@hAcAhByC~EyBnCqAhBqBhCIR@NHXPb@lCpF`BpDpAbCd@l@Nf@Hv@?j@O`BBr@A^[dFM~@?bAk@zHStBBhAD|GF~DEhAF|HBpAHp@HvMHtEClABzB@RP^Rt@JxAFPD@tBS|@Ef@M^Dp@@^CZI`@JfBDv@AnBNhAB|@HHAdAF|@@`@Fp@@^BXAVGJ@\\L~CLj@JF?BEBGAk@Fq@EsDBg@@xEGn@?n@AFE@[Es@CgAIq@Ae@MK?ODO@a@?cAG[EuEMq@Gi@?yAMS?u@CkA?e@M_@JY@k@?_@GK?g@LgBNaABN?z@IfAEZEZIJAZF\\A\\@NA^K^HfDFj@FpFVt@Fh@CVDnBF`@Il@LjADbCNBC@OAk@Hk@AsCAF@fCGr@?r@IHe@GaDMg@M]Fe@?aBKiACiAIcCEsBOgAEmA?g@ME?YJ]B[CU@OEUAYJWFe@?u@HaA@ABH@JE\\CJCfBIh@M`@Dz@@VA\\KXHbEHnAJxFTR?lBJ`@@\\IJ?TJH@hAFd@FTA^@b@FJC@G?k@Fs@CqCBjCGn@?p@ELG@UGw@AmBIQAa@Me@F_CMaBCWEkACa@EeDQyCCc@K_@H{@Do@IQBYJg@AmAJu@LMAEEKmAIc@_@cAIqB?a@Fu@?i@EwCK}NK{@C}AGkHFq@?YMmJ?qAIqCLwB@INIDW\\yE@WGWAO\\sD@g@JqAFuALoABq@Gq@Su@y@kA_C}Ee@y@IQAODQd@_AjBkC~@cB|@uA|ByDdAaBj@cAr@gAxDwGlEgHlAuBfBsCZk@JKt@qAp@aA|CaF`@k@xBeDzAqBxBgDfDyEv@_BZc@T{@A]u@wBGEMAUDOHaBlBKNuAdBQNUJKLq@dAUTWJK?GGQYMIIBa@^",
    "resource_state": 3,
    "summary_polyline": "sc}vH_yngBoA~BuArBS`@sD`GWf@qAnBiCfEUXoBfDu@hAcAhByC~EyBnCqAhBqBhCIR@NHXPb@lCpF`BpDpAbCd@l@Nf@Hv@?j@O`BBr@A^[dFM~@?bAk@zHStBBhAD|GF~DEhAF|HBpAHp@HvMHtEClABzB@RP^Rt@JxAFPD@tBSv@Cl@O^Dp@@^CZI`@JfBDv@AnBNhAB|@HHAdAF|@@`@Fp@@^BXAVGJ@\\L~CLj@JF?BEBGAk@Fq@EsDBg@@xEGn@?n@AFE@[Es@CgAIq@Ae@MK?ODO@a@?cAG[EuEMq@Gi@?yAMS?u@CkA?e@M_@JY@k@?_@GK?g@LgBNaABN?z@IfAEZEZIJAZF\\A\\@NA^K^HfDFj@FpFVt@Fh@CVDnBF`@Il@LjADbCNBC@OAk@Hk@AsCAF@fCGr@?r@IHe@GaDMg@M]Fe@?aBKiACiAIcCEsBOgAEmA?g@ME?YJ]B[CU@OEUAYJWFe@?u@HaA@ABH@JE\\CJCfBIh@M`@Dz@@VA\\KXHbEHnAJxFTR?lBJ`@@\\IJ?TJH@hAFd@FTA^@b@FJC@G?k@Fs@CqCBjCGn@?p@ELG@UGw@AmBIQAa@Me@F_CMaBCWEkACa@EeDQyCCc@K_@H{@Do@IQBYJg@AmAJu@LMAEEKmAIc@_@cAIqB?a@Fu@?i@EwCK}NK{@C}AGkHFq@?YMmJ?qAIqCLwB@INIDW\\yE@WGWAO\\sD@g@JqAFuALoABq@Gq@Su@y@kA_C}Ee@y@IQAODQd@_AjBkC~@cB|@uA|ByDdAaBj@cAr@gAxDwGlEgHlAuBfBsCZk@JKt@qAp@aA|CaF`@k@xBeDzAqBxBgDfDyEv@_BZc@T{@A]u@wBGEMAYFKFk@n@"
  },
  "trainer": false,
  "commute": false,
  "manual": false,
  "private": false,
  "visibility": "everyone",
  "flagged": false,
  "gear_id": "g23642256",
  "start_latlng": [51.107753, 17.123849],
  "end_latlng": [51.106867, 17.123285],
  "average_speed": 3.058,
  "max_speed": 4.18,
  "average_cadence": 85.2,
  "average_watts": 350.8,
  "max_watts": 564,
  "weighted_average_watts": 356,
  "device_watts": true,
  "kilojoules": 1298.3,
  "has_heartrate": true,
  "average_heartrate": 160.7,
  "max_heartrate": 180.0,
  "heartrate_opt_out": false,
  "display_hide_heartrate_option": true,
  "elev_high": 125.0,
  "elev_low": 113.2,
  "upload_id": 18048002093,
  "upload_id_str": "18048002093",
  "external_id": "garmin_ping_520613910011",
  "from_accepted_tag": false,
  "pr_count": 2,
  "total_photo_count": 0,
  "has_kudoed": false,
  "suffer_score": 115.0,
  "description": "Znowu ciężary, ale do przodu🫡\n\n2.5km warm up at a conversational pace \n\nRepeat the following 8x:\n----------\n400m at 4:35/km\n400m at 5:20/km\n----------\n\n90s walking rest\n\n2km cool down at a conversational pace",
  "calories": 881.0,
  "perceived_exertion": null,
  "prefer_perceived_exertion": null,
  "segment_efforts": [
    {
      "id": 3443577559040153460,
      "resource_state": 2,
      "name": "666m na AWW (South)",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 202,
      "moving_time": 202,
      "start_date": "2026-01-06T10:19:27Z",
      "start_date_local": "2026-01-06T11:19:27Z",
      "distance": 669.1,
      "start_index": 983,
      "end_index": 1185,
      "average_cadence": 86.6,
      "device_watts": true,
      "average_watts": 370.4,
      "average_heartrate": 167.4,
      "max_heartrate": 172.0,
      "segment": {
        "id": 38484972,
        "resource_state": 2,
        "name": "666m na AWW (South)",
        "activity_type": "Run",
        "distance": 669.1,
        "average_grade": -0.2,
        "maximum_grade": 3.5,
        "elevation_high": 123.4,
        "elevation_low": 121.6,
        "start_latlng": [51.112615, 17.091152],
        "end_latlng": [51.106638, 17.090862],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wrocław",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": null,
      "achievements": [],
      "visibility": "everyone",
      "kom_rank": null,
      "hidden": false
    },
    {
      "id": 3443577559039492980,
      "resource_state": 2,
      "name": "AWW - Dembowskiego -> Mickiewicza",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 226,
      "moving_time": 226,
      "start_date": "2026-01-06T10:24:03Z",
      "start_date_local": "2026-01-06T11:24:03Z",
      "distance": 779.0,
      "start_index": 1259,
      "end_index": 1485,
      "average_cadence": 86.9,
      "device_watts": true,
      "average_watts": 382.7,
      "average_heartrate": 171.6,
      "max_heartrate": 176.0,
      "segment": {
        "id": 37375847,
        "resource_state": 2,
        "name": "AWW - Dembowskiego -> Mickiewicza",
        "activity_type": "Run",
        "distance": 779.0,
        "average_grade": 0.1,
        "maximum_grade": 7.7,
        "elevation_high": 123.0,
        "elevation_low": 122.0,
        "start_latlng": [51.106243, 17.090706],
        "end_latlng": [51.113237, 17.090958],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wrocław",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": 2,
      "achievements": [
        {
          "type_id": 3,
          "type": "pr",
          "rank": 2
        }
      ],
      "visibility": "everyone",
      "kom_rank": null,
      "hidden": false
    },
    {
      "id": 3443577559040238452,
      "resource_state": 2,
      "name": "666m na AWW (South)",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 196,
      "moving_time": 196,
      "start_date": "2026-01-06T10:28:09Z",
      "start_date_local": "2026-01-06T11:28:09Z",
      "distance": 669.1,
      "start_index": 1505,
      "end_index": 1701,
      "average_cadence": 87.0,
      "device_watts": true,
      "average_watts": 381.6,
      "average_heartrate": 170.3,
      "max_heartrate": 175.0,
      "segment": {
        "id": 38484972,
        "resource_state": 2,
        "name": "666m na AWW (South)",
        "activity_type": "Run",
        "distance": 669.1,
        "average_grade": -0.2,
        "maximum_grade": 3.5,
        "elevation_high": 123.4,
        "elevation_low": 121.6,
        "start_latlng": [51.112615, 17.091152],
        "end_latlng": [51.106638, 17.090862],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wrocław",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": 2,
      "achievements": [
        {
          "type_id": 3,
          "type": "pr",
          "rank": 2
        }
      ],
      "visibility": "everyone",
      "kom_rank": null,
      "hidden": false
    },
    {
      "id": 3443577559042012020,
      "resource_state": 2,
      "name": "AWW - Dembowskiego -> Mickiewicza",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 225,
      "moving_time": 225,
      "start_date": "2026-01-06T10:32:32Z",
      "start_date_local": "2026-01-06T11:32:32Z",
      "distance": 779.0,
      "start_index": 1768,
      "end_index": 1993,
      "average_cadence": 87.0,
      "device_watts": true,
      "average_watts": 386.7,
      "average_heartrate": 173.0,
      "max_heartrate": 177.0,
      "segment": {
        "id": 37375847,
        "resource_state": 2,
        "name": "AWW - Dembowskiego -> Mickiewicza",
        "activity_type": "Run",
        "distance": 779.0,
        "average_grade": 0.1,
        "maximum_grade": 7.7,
        "elevation_high": 123.0,
        "elevation_low": 122.0,
        "start_latlng": [51.106243, 17.090706],
        "end_latlng": [51.113237, 17.090958],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wrocław",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": 1,
      "achievements": [
        {
          "type_id": 3,
          "type": "pr",
          "rank": 1
        }
      ],
      "visibility": "everyone",
      "kom_rank": null,
      "hidden": false
    },
    {
      "id": 3443577559039421300,
      "resource_state": 2,
      "name": "666m na AWW (South)",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 194,
      "moving_time": 194,
      "start_date": "2026-01-06T10:36:37Z",
      "start_date_local": "2026-01-06T11:36:37Z",
      "distance": 669.1,
      "start_index": 2013,
      "end_index": 2207,
      "average_cadence": 86.4,
      "device_watts": true,
      "average_watts": 396.0,
      "average_heartrate": 172.8,
      "max_heartrate": 178.0,
      "segment": {
        "id": 38484972,
        "resource_state": 2,
        "name": "666m na AWW (South)",
        "activity_type": "Run",
        "distance": 669.1,
        "average_grade": -0.2,
        "maximum_grade": 3.5,
        "elevation_high": 123.4,
        "elevation_low": 121.6,
        "start_latlng": [51.112615, 17.091152],
        "end_latlng": [51.106638, 17.090862],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wrocław",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": 1,
      "achievements": [
        {
          "type_id": 3,
          "type": "pr",
          "rank": 1
        }
      ],
      "visibility": "everyone",
      "kom_rank": null,
      "hidden": false
    },
    {
      "id": 3443577559040015220,
      "resource_state": 2,
      "name": "AWW - Dembowskiego -> Mickiewicza",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 228,
      "moving_time": 228,
      "start_date": "2026-01-06T10:40:56Z",
      "start_date_local": "2026-01-06T11:40:56Z",
      "distance": 779.0,
      "start_index": 2272,
      "end_index": 2500,
      "average_cadence": 86.5,
      "device_watts": true,
      "average_watts": 384.6,
      "average_heartrate": 173.7,
      "max_heartrate": 180.0,
      "segment": {
        "id": 37375847,
        "resource_state": 2,
        "name": "AWW - Dembowskiego -> Mickiewicza",
        "activity_type": "Run",
        "distance": 779.0,
        "average_grade": 0.1,
        "maximum_grade": 7.7,
        "elevation_high": 123.0,
        "elevation_low": 122.0,
        "start_latlng": [51.106243, 17.090706],
        "end_latlng": [51.113237, 17.090958],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wrocław",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": 3,
      "achievements": [
        {
          "type_id": 3,
          "type": "pr",
          "rank": 3
        }
      ],
      "visibility": "everyone",
      "kom_rank": null,
      "hidden": false
    },
    {
      "id": 3443577559041481588,
      "resource_state": 2,
      "name": "Swojczycki - Śluza",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 540,
      "moving_time": 540,
      "start_date": "2026-01-06T10:53:21Z",
      "start_date_local": "2026-01-06T11:53:21Z",
      "distance": 1463.6,
      "start_index": 3017,
      "end_index": 3557,
      "average_cadence": 84.8,
      "device_watts": true,
      "average_watts": 316.4,
      "average_heartrate": 150.2,
      "max_heartrate": 153.0,
      "segment": {
        "id": 8494871,
        "resource_state": 2,
        "name": "Swojczycki - Śluza",
        "activity_type": "Run",
        "distance": 1463.6,
        "average_grade": -0.1,
        "maximum_grade": 2.1,
        "elevation_high": 118.7,
        "elevation_low": 114.7,
        "start_latlng": [51.113907, 17.110391],
        "end_latlng": [51.104109, 17.12431],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wroclaw",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": null,
      "achievements": [],
      "visibility": "everyone",
      "kom_rank": null,
      "hidden": false
    },
    {
      "id": 3443577559042221940,
      "resource_state": 2,
      "name": "Finisz na betonozie",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 107,
      "moving_time": 107,
      "start_date": "2026-01-06T11:02:57Z",
      "start_date_local": "2026-01-06T12:02:57Z",
      "distance": 345.6,
      "start_index": 3593,
      "end_index": 3700,
      "average_cadence": 84.5,
      "device_watts": true,
      "average_watts": 317.7,
      "average_heartrate": 148.3,
      "max_heartrate": 154.0,
      "segment": {
        "id": 37625977,
        "resource_state": 2,
        "name": "Finisz na betonozie",
        "activity_type": "Run",
        "distance": 345.6,
        "average_grade": -1.6,
        "maximum_grade": 5.9,
        "elevation_high": 119.4,
        "elevation_low": 113.8,
        "start_latlng": [51.104576, 17.12517],
        "end_latlng": [51.107079, 17.123627],
        "elevation_profile": null,
        "elevation_profiles": null,
        "climb_category": 0,
        "city": "Wrocław",
        "state": "Lower Silesian Voivodeship",
        "country": "Poland",
        "private": false,
        "hazardous": false,
        "starred": false
      },
      "pr_rank": null,
      "achievements": [],
      "visibility": "only_me",
      "kom_rank": null,
      "hidden": false
    }
  ],
  "splits_metric": [
    {
      "distance": 1000.8,
      "elapsed_time": 344,
      "elevation_difference": 5.0,
      "moving_time": 344,
      "split": 1,
      "average_speed": 2.91,
      "average_grade_adjusted_speed": 2.96,
      "average_heartrate": 143.00290697674419,
      "pace_zone": 2
    },
    {
      "distance": 1000.1,
      "elapsed_time": 353,
      "elevation_difference": -0.6,
      "moving_time": 353,
      "split": 2,
      "average_speed": 2.83,
      "average_grade_adjusted_speed": 2.85,
      "average_heartrate": 149.70254957507083,
      "pace_zone": 2
    },
    {
      "distance": 1000.8,
      "elapsed_time": 317,
      "elevation_difference": -1.2,
      "moving_time": 317,
      "split": 3,
      "average_speed": 3.16,
      "average_grade_adjusted_speed": 3.16,
      "average_heartrate": 159.6750788643533,
      "pace_zone": 2
    },
    {
      "distance": 1000.7,
      "elapsed_time": 303,
      "elevation_difference": -1.6,
      "moving_time": 303,
      "split": 4,
      "average_speed": 3.3,
      "average_grade_adjusted_speed": 3.3,
      "average_heartrate": 168.67218543046357,
      "pace_zone": 3
    },
    {
      "distance": 1001.1,
      "elapsed_time": 296,
      "elevation_difference": -1.2,
      "moving_time": 296,
      "split": 5,
      "average_speed": 3.38,
      "average_grade_adjusted_speed": 3.38,
      "average_heartrate": 170.8716216216216,
      "pace_zone": 3
    },
    {
      "distance": 997.6,
      "elapsed_time": 294,
      "elevation_difference": -0.2,
      "moving_time": 294,
      "split": 6,
      "average_speed": 3.39,
      "average_grade_adjusted_speed": 3.41,
      "average_heartrate": 172.36860068259386,
      "pace_zone": 3
    },
    {
      "distance": 1000.6,
      "elapsed_time": 297,
      "elevation_difference": 0.6,
      "moving_time": 297,
      "split": 7,
      "average_speed": 3.37,
      "average_grade_adjusted_speed": 3.38,
      "average_heartrate": 173.1919191919192,
      "pace_zone": 3
    },
    {
      "distance": 999.7,
      "elapsed_time": 303,
      "elevation_difference": -0.4,
      "moving_time": 303,
      "split": 8,
      "average_speed": 3.3,
      "average_grade_adjusted_speed": 3.31,
      "average_heartrate": 173.26732673267327,
      "pace_zone": 3
    },
    {
      "distance": 998.7,
      "elapsed_time": 326,
      "elevation_difference": -0.2,
      "moving_time": 326,
      "split": 9,
      "average_speed": 3.06,
      "average_grade_adjusted_speed": 3.07,
      "average_heartrate": 170.77914110429447,
      "pace_zone": 2
    },
    {
      "distance": 1001.4,
      "elapsed_time": 390,
      "elevation_difference": 1.2,
      "moving_time": 390,
      "split": 10,
      "average_speed": 2.57,
      "average_grade_adjusted_speed": 2.59,
      "average_heartrate": 150.03076923076924,
      "pace_zone": 1
    },
    {
      "distance": 1000.4,
      "elapsed_time": 365,
      "elevation_difference": 2.6,
      "moving_time": 365,
      "split": 11,
      "average_speed": 2.74,
      "average_grade_adjusted_speed": 2.77,
      "average_heartrate": 150.44931506849315,
      "pace_zone": 1
    },
    {
      "distance": 315.8,
      "elapsed_time": 112,
      "elevation_difference": -4.4,
      "moving_time": 112,
      "split": 12,
      "average_speed": 2.82,
      "average_grade_adjusted_speed": 2.75,
      "average_heartrate": 148.54464285714286,
      "pace_zone": 1
    }
  ],
  "splits_standard": [
    {
      "distance": 1610.6,
      "elapsed_time": 563,
      "elevation_difference": 9.2,
      "moving_time": 563,
      "split": 1,
      "average_speed": 2.86,
      "average_grade_adjusted_speed": 2.91,
      "average_heartrate": 145.69449378330373,
      "pace_zone": 2
    },
    {
      "distance": 1610.8,
      "elapsed_time": 522,
      "elevation_difference": -6.8,
      "moving_time": 522,
      "split": 2,
      "average_speed": 3.09,
      "average_grade_adjusted_speed": 3.07,
      "average_heartrate": 157.75862068965517,
      "pace_zone": 2
    },
    {
      "distance": 1608.1,
      "elapsed_time": 478,
      "elevation_difference": -1.0,
      "moving_time": 478,
      "split": 3,
      "average_speed": 3.36,
      "average_grade_adjusted_speed": 3.37,
      "average_heartrate": 170.65408805031447,
      "pace_zone": 3
    },
    {
      "distance": 1609.6,
      "elapsed_time": 480,
      "elevation_difference": -0.4,
      "moving_time": 480,
      "split": 4,
      "average_speed": 3.35,
      "average_grade_adjusted_speed": 3.36,
      "average_heartrate": 172.09394572025053,
      "pace_zone": 3
    },
    {
      "distance": 1609.0,
      "elapsed_time": 478,
      "elevation_difference": -0.8,
      "moving_time": 478,
      "split": 5,
      "average_speed": 3.37,
      "average_grade_adjusted_speed": 3.37,
      "average_heartrate": 173.23849372384936,
      "pace_zone": 3
    },
    {
      "distance": 1608.4,
      "elapsed_time": 572,
      "elevation_difference": 0.8,
      "moving_time": 572,
      "split": 6,
      "average_speed": 2.81,
      "average_grade_adjusted_speed": 2.83,
      "average_heartrate": 161.21153846153845,
      "pace_zone": 2
    },
    {
      "distance": 1609.7,
      "elapsed_time": 588,
      "elevation_difference": -1.2,
      "moving_time": 588,
      "split": 7,
      "average_speed": 2.74,
      "average_grade_adjusted_speed": 2.74,
      "average_heartrate": 150.20408163265307,
      "pace_zone": 1
    },
    {
      "distance": 51.5,
      "elapsed_time": 19,
      "elevation_difference": -0.2,
      "moving_time": 19,
      "split": 8,
      "average_speed": 2.71,
      "average_grade_adjusted_speed": 2.71,
      "average_heartrate": 147.31578947368422,
      "pace_zone": 1
    }
  ],
  "laps": [
    {
      "id": 60497303903,
      "resource_state": 2,
      "name": "Lap 1",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 344,
      "moving_time": 344,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 1000.0,
      "average_speed": 2.91,
      "max_speed": 3.2,
      "lap_index": 1,
      "split": 1,
      "start_index": 0,
      "end_index": 231,
      "total_elevation_gain": 5.2,
      "average_cadence": 85.4,
      "device_watts": true,
      "average_watts": 334.6,
      "average_heartrate": 143.0,
      "max_heartrate": 151.0,
      "pace_zone": 2
    },
    {
      "id": 60497303992,
      "resource_state": 2,
      "name": "Lap 2",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 352,
      "moving_time": 352,
      "start_date": "2026-01-06T10:08:50Z",
      "start_date_local": "2026-01-06T11:08:50Z",
      "distance": 1000.0,
      "average_speed": 2.84,
      "max_speed": 3.14,
      "lap_index": 2,
      "split": 2,
      "start_index": 232,
      "end_index": 584,
      "total_elevation_gain": 5.8,
      "average_cadence": 85.3,
      "device_watts": true,
      "average_watts": 330.6,
      "average_heartrate": 149.7,
      "max_heartrate": 155.0,
      "pace_zone": 2
    },
    {
      "id": 60497304014,
      "resource_state": 2,
      "name": "Lap 3",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 175,
      "moving_time": 175,
      "start_date": "2026-01-06T10:14:43Z",
      "start_date_local": "2026-01-06T11:14:43Z",
      "distance": 500.0,
      "average_speed": 2.86,
      "max_speed": 3.24,
      "lap_index": 3,
      "split": 3,
      "start_index": 585,
      "end_index": 759,
      "total_elevation_gain": 0.0,
      "average_cadence": 85.3,
      "device_watts": true,
      "average_watts": 323.9,
      "average_heartrate": 152.9,
      "max_heartrate": 156.0,
      "pace_zone": 2
    },
    {
      "id": 60497304036,
      "resource_state": 2,
      "name": "Lap 4",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 110,
      "moving_time": 110,
      "start_date": "2026-01-06T10:17:38Z",
      "start_date_local": "2026-01-06T11:17:38Z",
      "distance": 400.0,
      "average_speed": 3.64,
      "max_speed": 4.18,
      "lap_index": 4,
      "split": 4,
      "start_index": 760,
      "end_index": 870,
      "total_elevation_gain": 0.0,
      "average_cadence": 88.2,
      "device_watts": true,
      "average_watts": 398.7,
      "average_heartrate": 167.5,
      "max_heartrate": 172.0,
      "pace_zone": 3
    },
    {
      "id": 60497304061,
      "resource_state": 2,
      "name": "Lap 5",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 127,
      "moving_time": 127,
      "start_date": "2026-01-06T10:19:29Z",
      "start_date_local": "2026-01-06T11:19:29Z",
      "distance": 400.0,
      "average_speed": 3.15,
      "max_speed": 3.78,
      "lap_index": 5,
      "split": 5,
      "start_index": 871,
      "end_index": 998,
      "total_elevation_gain": 0.0,
      "average_cadence": 85.8,
      "device_watts": true,
      "average_watts": 351.6,
      "average_heartrate": 165.9,
      "max_heartrate": 172.0,
      "pace_zone": 2
    },
    {
      "id": 60497304077,
      "resource_state": 2,
      "name": "Lap 6",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 110,
      "moving_time": 110,
      "start_date": "2026-01-06T10:21:37Z",
      "start_date_local": "2026-01-06T11:21:37Z",
      "distance": 400.0,
      "average_speed": 3.64,
      "max_speed": 3.98,
      "lap_index": 6,
      "split": 6,
      "start_index": 999,
      "end_index": 1108,
      "total_elevation_gain": 0.0,
      "average_cadence": 88.1,
      "device_watts": true,
      "average_watts": 400.2,
      "average_heartrate": 170.6,
      "max_heartrate": 173.0,
      "pace_zone": 3
    },
    {
      "id": 60497304094,
      "resource_state": 2,
      "name": "Lap 7",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 126,
      "moving_time": 126,
      "start_date": "2026-01-06T10:23:27Z",
      "start_date_local": "2026-01-06T11:23:27Z",
      "distance": 400.0,
      "average_speed": 3.17,
      "max_speed": 3.6,
      "lap_index": 7,
      "split": 7,
      "start_index": 1109,
      "end_index": 1235,
      "total_elevation_gain": 0.0,
      "average_cadence": 86.4,
      "device_watts": true,
      "average_watts": 361.9,
      "average_heartrate": 170.0,
      "max_heartrate": 174.0,
      "pace_zone": 2
    },
    {
      "id": 60497304118,
      "resource_state": 2,
      "name": "Lap 8",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 110,
      "moving_time": 110,
      "start_date": "2026-01-06T10:25:34Z",
      "start_date_local": "2026-01-06T11:25:34Z",
      "distance": 400.0,
      "average_speed": 3.64,
      "max_speed": 3.98,
      "lap_index": 8,
      "split": 8,
      "start_index": 1236,
      "end_index": 1345,
      "total_elevation_gain": 0.0,
      "average_cadence": 87.7,
      "device_watts": true,
      "average_watts": 403.2,
      "average_heartrate": 172.9,
      "max_heartrate": 176.0,
      "pace_zone": 3
    },
    {
      "id": 60497304142,
      "resource_state": 2,
      "name": "Lap 9",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 128,
      "moving_time": 128,
      "start_date": "2026-01-06T10:27:24Z",
      "start_date_local": "2026-01-06T11:27:24Z",
      "distance": 400.0,
      "average_speed": 3.13,
      "max_speed": 3.76,
      "lap_index": 9,
      "split": 9,
      "start_index": 1346,
      "end_index": 1473,
      "total_elevation_gain": 2.0,
      "average_cadence": 85.9,
      "device_watts": true,
      "average_watts": 363.5,
      "average_heartrate": 170.3,
      "max_heartrate": 176.0,
      "pace_zone": 2
    },
    {
      "id": 60497304173,
      "resource_state": 2,
      "name": "Lap 10",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 110,
      "moving_time": 110,
      "start_date": "2026-01-06T10:29:32Z",
      "start_date_local": "2026-01-06T11:29:32Z",
      "distance": 400.0,
      "average_speed": 3.64,
      "max_speed": 4.04,
      "lap_index": 10,
      "split": 10,
      "start_index": 1474,
      "end_index": 1584,
      "total_elevation_gain": 0.0,
      "average_cadence": 87.8,
      "device_watts": true,
      "average_watts": 402.8,
      "average_heartrate": 171.6,
      "max_heartrate": 174.0,
      "pace_zone": 3
    },
    {
      "id": 60497304197,
      "resource_state": 2,
      "name": "Lap 11",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 127,
      "moving_time": 127,
      "start_date": "2026-01-06T10:31:23Z",
      "start_date_local": "2026-01-06T11:31:23Z",
      "distance": 400.0,
      "average_speed": 3.15,
      "max_speed": 3.58,
      "lap_index": 11,
      "split": 11,
      "start_index": 1585,
      "end_index": 1711,
      "total_elevation_gain": 0.0,
      "average_cadence": 85.5,
      "device_watts": true,
      "average_watts": 365.5,
      "average_heartrate": 171.9,
      "max_heartrate": 176.0,
      "pace_zone": 2
    },
    {
      "id": 60497304218,
      "resource_state": 2,
      "name": "Lap 12",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 110,
      "moving_time": 110,
      "start_date": "2026-01-06T10:33:30Z",
      "start_date_local": "2026-01-06T11:33:30Z",
      "distance": 400.0,
      "average_speed": 3.64,
      "max_speed": 4.02,
      "lap_index": 12,
      "split": 12,
      "start_index": 1712,
      "end_index": 1822,
      "total_elevation_gain": 0.0,
      "average_cadence": 87.8,
      "device_watts": true,
      "average_watts": 406.9,
      "average_heartrate": 173.9,
      "max_heartrate": 177.0,
      "pace_zone": 3
    },
    {
      "id": 60497304247,
      "resource_state": 2,
      "name": "Lap 13",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 127,
      "moving_time": 127,
      "start_date": "2026-01-06T10:35:21Z",
      "start_date_local": "2026-01-06T11:35:21Z",
      "distance": 400.0,
      "average_speed": 3.15,
      "max_speed": 3.48,
      "lap_index": 13,
      "split": 13,
      "start_index": 1823,
      "end_index": 1949,
      "total_elevation_gain": 0.0,
      "average_cadence": 85.2,
      "device_watts": true,
      "average_watts": 365.4,
      "average_heartrate": 172.0,
      "max_heartrate": 176.0,
      "pace_zone": 2
    },
    {
      "id": 60497304268,
      "resource_state": 2,
      "name": "Lap 14",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 109,
      "moving_time": 109,
      "start_date": "2026-01-06T10:37:28Z",
      "start_date_local": "2026-01-06T11:37:28Z",
      "distance": 400.0,
      "average_speed": 3.67,
      "max_speed": 4.08,
      "lap_index": 14,
      "split": 14,
      "start_index": 1950,
      "end_index": 2059,
      "total_elevation_gain": 0.0,
      "average_cadence": 87.5,
      "device_watts": true,
      "average_watts": 413.0,
      "average_heartrate": 173.6,
      "max_heartrate": 178.0,
      "pace_zone": 4
    },
    {
      "id": 60497304291,
      "resource_state": 2,
      "name": "Lap 15",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 127,
      "moving_time": 127,
      "start_date": "2026-01-06T10:39:18Z",
      "start_date_local": "2026-01-06T11:39:18Z",
      "distance": 400.0,
      "average_speed": 3.15,
      "max_speed": 3.62,
      "lap_index": 15,
      "split": 15,
      "start_index": 2060,
      "end_index": 2187,
      "total_elevation_gain": 2.0,
      "average_cadence": 85.4,
      "device_watts": true,
      "average_watts": 370.4,
      "average_heartrate": 172.6,
      "max_heartrate": 175.0,
      "pace_zone": 2
    },
    {
      "id": 60497304308,
      "resource_state": 2,
      "name": "Lap 16",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 110,
      "moving_time": 110,
      "start_date": "2026-01-06T10:41:26Z",
      "start_date_local": "2026-01-06T11:41:26Z",
      "distance": 400.0,
      "average_speed": 3.64,
      "max_speed": 4.12,
      "lap_index": 16,
      "split": 16,
      "start_index": 2188,
      "end_index": 2297,
      "total_elevation_gain": 0.0,
      "average_cadence": 87.3,
      "device_watts": true,
      "average_watts": 401.2,
      "average_heartrate": 174.5,
      "max_heartrate": 180.0,
      "pace_zone": 3
    },
    {
      "id": 60497304328,
      "resource_state": 2,
      "name": "Lap 17",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 127,
      "moving_time": 127,
      "start_date": "2026-01-06T10:43:16Z",
      "start_date_local": "2026-01-06T11:43:16Z",
      "distance": 400.0,
      "average_speed": 3.15,
      "max_speed": 3.9,
      "lap_index": 17,
      "split": 17,
      "start_index": 2298,
      "end_index": 2425,
      "total_elevation_gain": 0.0,
      "average_cadence": 85.3,
      "device_watts": true,
      "average_watts": 364.7,
      "average_heartrate": 172.6,
      "max_heartrate": 178.0,
      "pace_zone": 2
    },
    {
      "id": 60497304349,
      "resource_state": 2,
      "name": "Lap 18",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 109,
      "moving_time": 109,
      "start_date": "2026-01-06T10:45:24Z",
      "start_date_local": "2026-01-06T11:45:24Z",
      "distance": 400.0,
      "average_speed": 3.67,
      "max_speed": 4.0,
      "lap_index": 18,
      "split": 18,
      "start_index": 2426,
      "end_index": 2535,
      "total_elevation_gain": 0.0,
      "average_cadence": 87.7,
      "device_watts": true,
      "average_watts": 401.8,
      "average_heartrate": 174.6,
      "max_heartrate": 179.0,
      "pace_zone": 4
    },
    {
      "id": 60497304373,
      "resource_state": 2,
      "name": "Lap 19",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 127,
      "moving_time": 127,
      "start_date": "2026-01-06T10:47:14Z",
      "start_date_local": "2026-01-06T11:47:14Z",
      "distance": 400.0,
      "average_speed": 3.15,
      "max_speed": 3.48,
      "lap_index": 19,
      "split": 19,
      "start_index": 2536,
      "end_index": 2662,
      "total_elevation_gain": 0.0,
      "average_cadence": 85.1,
      "device_watts": true,
      "average_watts": 369.4,
      "average_heartrate": 172.0,
      "max_heartrate": 177.0,
      "pace_zone": 2
    },
    {
      "id": 60497304399,
      "resource_state": 2,
      "name": "Lap 20",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 89,
      "moving_time": 90,
      "start_date": "2026-01-06T10:49:21Z",
      "start_date_local": "2026-01-06T11:49:21Z",
      "distance": 151.93,
      "average_speed": 1.69,
      "max_speed": 3.22,
      "lap_index": 20,
      "split": 20,
      "start_index": 2663,
      "end_index": 2751,
      "total_elevation_gain": 0.0,
      "average_cadence": 59.6,
      "device_watts": true,
      "average_watts": 166.9,
      "average_heartrate": 153.7,
      "max_heartrate": 170.0,
      "pace_zone": 1
    },
    {
      "id": 60497304430,
      "resource_state": 2,
      "name": "Lap 21",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 376,
      "moving_time": 376,
      "start_date": "2026-01-06T10:50:50Z",
      "start_date_local": "2026-01-06T11:50:50Z",
      "distance": 1000.0,
      "average_speed": 2.66,
      "max_speed": 2.94,
      "lap_index": 21,
      "split": 21,
      "start_index": 2752,
      "end_index": 3128,
      "total_elevation_gain": 5.0,
      "average_cadence": 84.6,
      "device_watts": true,
      "average_watts": 312.8,
      "average_heartrate": 150.8,
      "max_heartrate": 157.0,
      "pace_zone": 1
    },
    {
      "id": 60497304457,
      "resource_state": 2,
      "name": "Lap 22",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 366,
      "moving_time": 366,
      "start_date": "2026-01-06T10:57:07Z",
      "start_date_local": "2026-01-06T11:57:07Z",
      "distance": 1000.0,
      "average_speed": 2.73,
      "max_speed": 3.1,
      "lap_index": 22,
      "split": 22,
      "start_index": 3129,
      "end_index": 3494,
      "total_elevation_gain": 3.8,
      "average_cadence": 84.5,
      "device_watts": true,
      "average_watts": 324.6,
      "average_heartrate": 150.5,
      "max_heartrate": 155.0,
      "pace_zone": 1
    },
    {
      "id": 60497304482,
      "resource_state": 2,
      "name": "Lap 23",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 91,
      "moving_time": 91,
      "start_date": "2026-01-06T11:03:14Z",
      "start_date_local": "2026-01-06T12:03:14Z",
      "distance": 265.73,
      "average_speed": 2.92,
      "max_speed": 3.16,
      "lap_index": 23,
      "split": 23,
      "start_index": 3494,
      "end_index": 3494,
      "total_elevation_gain": 0.0,
      "average_cadence": 84.7,
      "device_watts": true,
      "average_watts": 326.0,
      "average_heartrate": 147.5,
      "max_heartrate": 150.0,
      "pace_zone": 2
    }
  ],
  "best_efforts": [
    {
      "id": 71440973971,
      "resource_state": 2,
      "name": "400m",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 110,
      "moving_time": 110,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 400,
      "pr_rank": null,
      "achievements": [],
      "start_index": 2060,
      "end_index": 2170
    },
    {
      "id": 71440973972,
      "resource_state": 2,
      "name": "1/2 mile",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 237,
      "moving_time": 237,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 805,
      "pr_rank": null,
      "achievements": [],
      "start_index": 1227,
      "end_index": 1464
    },
    {
      "id": 71440973973,
      "resource_state": 2,
      "name": "1K",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 293,
      "moving_time": 293,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 1000,
      "pr_rank": null,
      "achievements": [],
      "start_index": 1107,
      "end_index": 1400
    },
    {
      "id": 71440973967,
      "resource_state": 2,
      "name": "1 mile",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 476,
      "moving_time": 476,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 1609,
      "pr_rank": null,
      "achievements": [],
      "start_index": 1742,
      "end_index": 2218
    },
    {
      "id": 71440973968,
      "resource_state": 2,
      "name": "2 mile",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 955,
      "moving_time": 955,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 3219,
      "pr_rank": null,
      "achievements": [],
      "start_index": 1744,
      "end_index": 2699
    },
    {
      "id": 71440973969,
      "resource_state": 2,
      "name": "5K",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 1482,
      "moving_time": 1482,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 5000,
      "pr_rank": null,
      "achievements": [],
      "start_index": 1161,
      "end_index": 2643
    },
    {
      "id": 71440973970,
      "resource_state": 2,
      "name": "10K",
      "activity": {
        "id": 16954650847,
        "visibility": "everyone",
        "resource_state": 1
      },
      "athlete": {
        "id": 81055898,
        "resource_state": 1
      },
      "elapsed_time": 3224,
      "moving_time": 3224,
      "start_date": "2026-01-06T10:03:04Z",
      "start_date_local": "2026-01-06T11:03:04Z",
      "distance": 10000,
      "pr_rank": null,
      "achievements": [],
      "start_index": 0,
      "end_index": 3224
    }
  ],
  "gear": {
    "id": "g23642256",
    "primary": false,
    "name": "Adidas EVO SL Czarne",
    "nickname": "Czarne",
    "resource_state": 2,
    "retired": false,
    "distance": 429862,
    "converted_distance": 429.9
  },
  "photos": {
    "primary": null,
    "count": 0
  },
  "stats_visibility": [
    {
      "type": "heart_rate",
      "visibility": "everyone"
    },
    {
      "type": "pace",
      "visibility": "everyone"
    },
    {
      "type": "power",
      "visibility": "everyone"
    },
    {
      "type": "speed",
      "visibility": "everyone"
    },
    {
      "type": "calories",
      "visibility": "everyone"
    }
  ],
  "hide_from_home": false,
  "embed_token": "a2dd13e648afbc124026e533f1579597c226f670",
  "similar_activities": {
    "effort_count": 2,
    "average_speed": 2.9977316171785398,
    "min_average_speed": 2.9438389197009887,
    "mid_average_speed": 2.9998987500660426,
    "max_average_speed": 3.058135135135135,
    "pr_rank": 1,
    "frequency_milestone": null,
    "trend": {
      "speeds": [2.9438389197009887, 2.9998987500660426],
      "current_activity_index": 1,
      "min_speed": 2.9438389197009887,
      "mid_speed": 2.9998987500660426,
      "max_speed": 3.058135135135135,
      "direction": 1
    },
    "resource_state": 2
  },
  "available_zones": ["heartrate", "pace", "power"]
}


```
---

#### 4) Kudos — `GET /activities/{id}/kudos`

```json
[
  {
    "resource_state": 2,
    "firstname": "Mal",
    "lastname": "C."
  },
  {
    "resource_state": 2,
    "firstname": "Wiesława",
    "lastname": "C."
  },
  {
    "resource_state": 2,
    "firstname": "Filip",
    "lastname": "C."
  },
  {
    "resource_state": 2,
    "firstname": "Alicja",
    "lastname": "Ł."
  },
  {
    "resource_state": 2,
    "firstname": "Kacper",
    "lastname": "K."
  },
  {
    "resource_state": 2,
    "firstname": "Ola",
    "lastname": "Ł."
  },
  {
    "resource_state": 2,
    "firstname": "Karolina",
    "lastname": "C."
  },
  {
    "resource_state": 2,
    "firstname": "Michal",
    "lastname": "H."
  },
  {
    "resource_state": 2,
    "firstname": "Łukasz",
    "lastname": "C."
  }
]

```


---

#### 5) Zones — `GET /activities/{id}/zones`

```json
[
  {
    "score": 115.0,
    "distribution_buckets": [
      { "min": 0, "max": 133, "time": 40.0 },
      { "min": 134, "max": 147, "time": 378.0 },
      { "min": 148, "max": 160, "time": 1358.0 },
      { "min": 161, "max": 166, "time": 131.0 },
      { "min": 167, "max": -1, "time": 1793.0 }
    ],
    "type": "heartrate",
    "resource_state": 3,
    "sensor_based": true,
    "points": 59,
    "custom_zones": true
  },
  {
    "score": 5,
    "distribution_buckets": [
      { "min": 0, "max": 2.828, "time": 967.0 },
      { "min": 2.828, "max": 3.285, "time": 1641.0 },
      { "min": 3.285, "max": 3.66, "time": 947.0 },
      { "min": 3.66, "max": 3.909, "time": 145.0 },
      { "min": 3.909, "max": 4.159, "time": 0.0 },
      { "min": 4.159, "max": -1, "time": 0.0 }
    ],
    "type": "pace",
    "resource_state": 3,
    "sensor_based": true
  },
  {
    "distribution_buckets": [
      { "min": 0, "max": 0, "time": 0.0 },
      { "min": 0, "max": 50, "time": 0.0 },
      { "min": 50, "max": 100, "time": 0.0 },
      { "min": 100, "max": 150, "time": 41.0 },
      { "min": 150, "max": 200, "time": 40.0 },
      { "min": 200, "max": 250, "time": 5.0 },
      { "min": 250, "max": 300, "time": 125.0 },
      { "min": 300, "max": 350, "time": 1668.0 },
      { "min": 350, "max": 400, "time": 1199.0 },
      { "min": 400, "max": 450, "time": 609.0 },
      { "min": 450, "max": -1, "time": 13.0 }
    ],
    "type": "power",
    "resource_state": 3,
    "sensor_based": true
  }
]

```

---

### 6) Segment details — `GET /segments/{id}`

```json
{
  "id": 38484972,
  "resource_state": 3,
  "name": "666m na AWW (South)",
  "activity_type": "Run",
  "distance": 669.1,
  "average_grade": -0.2,
  "maximum_grade": 3.5,
  "elevation_high": 123.4,
  "elevation_low": 121.6,
  "start_latlng": [51.112615, 17.091152],
  "end_latlng": [51.106638, 17.090862],
  "elevation_profile": "https://d3o5xota0a1fcr.cloudfront.net/v6/charts/IYTDEYUO6VGURC4K4CJIYGOVHSKW6TNW6MSRYXIZVZKYWGKGZ4I6TNV2VQTJOTAJLSFQHLWZU5AENHH5LXMRA===",
  "elevation_profiles": {
    "light_url": "https://d3o5xota0a1fcr.cloudfront.net/v6/charts/IYTDEYUO6VGURC4K4CJIYGOVHSKW6TNW6MSRYXIZVZKYWGKGZ4I6TNV2VQTJOTAJLSFQHLWZU5AENHH5LXMRA===",
    "dark_url": "https://d3o5xota0a1fcr.cloudfront.net/v6/charts/N52F6COMCCSSY6ZAXKMRNHWKWAO6MKQQTD5MEBUNOY5OGGJLGBPQXAA7VPRMUIIQM77DZUG4WE4X577ZXPPUQ==="
  },
  "climb_category": 0,
  "city": "Wrocław",
  "state": "Lower Silesian Voivodeship",
  "country": "Poland",
  "private": false,
  "hazardous": false,
  "starred": false,
  "created_at": "2025-01-22T17:37:46Z",
  "updated_at": "2025-01-22T17:37:46Z",
  "total_elevation_gain": 0.0,
  "map": {
    "id": "s38484972",
    "polyline": "y|}vHubigBv@Eb@ItBD\\Ip@L`GH^Dl@@hBN~@@^CZFn@?PBR?p@EpARp@?TBp@@",
    "resource_state": 3
  },
  "effort_count": 2908,
  "athlete_count": 785,
  "star_count": 2,
  "athlete_segment_stats": {
    "pr_elapsed_time": 194,
    "pr_date": "2026-01-06",
    "pr_visibility": "everyone",
    "pr_activity_id": 16954650847,
    "pr_activity_visibility": "everyone",
    "effort_count": 14
  },
  "xoms": {
    "kom": "1:56",
    "qom": "2:45",
    "overall": "1:56",
    "destination": {
      "href": "strava://segments/38484972/leaderboard",
      "type": "overall",
      "name": "All-Time"
    }
  },
  "local_legend": {
    "athlete_id": 120923595,
    "title": "Dawid Domagała",
    "profile": "https://graph.facebook.com/6642398219124853/picture?height=256&width=256",
    "effort_description": "24 efforts in the last 90 days",
    "effort_count": "24",
    "effort_counts": {
      "overall": "24 efforts",
      "female": "8 efforts"
    },
    "destination": "strava://segments/38484972/local_legend?categories%5B%5D=overall"
  }
}
```

---

