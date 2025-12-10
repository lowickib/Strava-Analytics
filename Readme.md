## 4. Power BI Dashboard Overview

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

➡️ A detailed page-by-page description can be found here [here](docs/04_power_bi_dashboard.md)
