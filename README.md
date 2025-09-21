# plsql-window-functions-Iza-Kuradusenge-Emma-Lise
PL/SQL Window Functions Assignment — AUCA 2025.
 ## Step 1: Problem Definition
**Business Context:** TuneWave, a regional music streaming platform - Content Strategy Team.

**Data challenge:** From January through December 2024, TuneWave logged millions of song plays, however, it does not have a detailed understanding of the top genres and artists in each region, or the monthly variation in listening behaviors. Also, user retention is an issue as many listeners stop engaging with the service after a few months.

**Expected outcome:** Identify the top 5 artists for each region and quarter, calculate cumulative monthly streaming totals, recognize month-over-month changes in active listeners, and classify listeners into quartiles based on their engagement behavior to improve playlist development and targeted campaigns for promotions.
## Step 2: Success Criteria

We will use the following 5 measurable goals to guide our PL/SQL window function analysis for TuneWave:

1. **Top 5 artists per region and quarter → RANK()**  
   Identify the artists that have the highest streaming in each of the regions each quarter to know which content to promote, and curate playlists with.

2. **Running monthly streaming totals → SUM() OVER()**  
   Determine how many streams each region or artist has in cumulative total over the months for each quarter to see trends and growth.

3. **Monthly active users month-over-month → LAG()/LEAD()**  
   Queue listening counts month-over-month to identify growth, decline, or a seasonal pattern.

4. **Listener engagement quartiles → NTILE(4)**  
   Cluster listeners into 4 groups based on the total streams each year to find targeted campaigns around top listeners and low engagement.

5. **3-month moving averages of streams per artist → AVG() OVER()**  
   Calculate rolling 3-month averages to quantify the ups and downs, and to find trend data on streams per artist.
   ## Step 3: Database Schema

We designed three related tables for the TuneWave music streaming platform. The tables are connected by foreign keys, ensuring referential integrity.

| Table Name | Purpose            | Key Columns                                                      | Example Row                                     |
|------------|--------------------|------------------------------------------------------------------|------------------------------------------------|
| listeners  | Customer info      | listener_id (PK), name, region                                   | 1001, Alice Uwase, Kigali                      |
| songs      | Song catalog       | song_id (PK), title, artist, genre                               | 2001, Umutima, The Ben, Afrobeat               |
| streams    | Streaming records  | stream_id (PK), listener_id (FK → listeners), song_id (FK → songs), stream_date, duration | 3001, 1001, 2001, 2025-01-15, 180 |

### ER Diagram
![ER Diagram](https://raw.githubusercontent.com/Emmalise1/plsql-window-functions-Iza-Kuradusenge-Emma-Lise/4eafdad53a9e71bcba39df65d908fa7820329d5a/images/plsql-window-functions-Iza%20Kuradusenge-Emma%20Lise-Er-diagram.PNG) 

# Step 4: Window Functions Implementation 

We implemented four categories of PL/SQL window functions in Oracle SQL Developer. Each section below shows the **query**, a **screenshot** of results, and a 2–3 sentence **interpretation**.

---

### 1) Ranking — `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `PERCENT_RANK()`

**Query (Ranking Functions):**
```sql
 Ranking listeners by total streaming duration (descending)
SELECT
  t.listener_id,
  t.name,
  t.total_duration,
  ROW_NUMBER()   OVER (ORDER BY t.total_duration DESC) AS row_num,
  RANK()         OVER (ORDER BY t.total_duration DESC) AS rnk,
  DENSE_RANK()   OVER (ORDER BY t.total_duration DESC) AS dense_rnk,
  PERCENT_RANK() OVER (ORDER BY t.total_duration DESC) AS pct_rank
FROM (
  SELECT l.listener_id, l.name, SUM(s.duration) AS total_duration
  FROM listeners l
  JOIN streams s ON l.listener_id = s.listener_id
  GROUP BY l.listener_id, l.name
) t
ORDER BY t.total_duration DESC; 
```

**Screenshot:**  
![Ranking Results](https://github.com/Emmalise1/plsql-window-functions-Iza-Kuradusenge-Emma-Lise/blob/main/Ranking.PNG?raw=true)

**Interpretation**
This query computes total streaming duration per listener and ranks them. ROW_NUMBER gives a strict ordering; RANK allows ties (with gaps); DENSE_RANK compresses gaps; PERCENT_RANK shows relative standing. You can use these results for identifying your top listeners for rewards, or a VIP program. 
### Aggregate — SUM(), AVG(), MIN(), MAX() (ROWS vs RANGE)
**SQL Query (Running Totals & Moving Average):**
```sql
WITH monthly AS (
  SELECT TRUNC(stream_date,'MM') AS month,
         SUM(duration) AS monthly_total
  FROM streams
  GROUP BY TRUNC(stream_date,'MM')
)
SELECT month,
       monthly_total,
       SUM(monthly_total) OVER (
         ORDER BY month
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total_rows,
       SUM(monthly_total) OVER (
         ORDER BY month
         RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total_range,
       AVG(monthly_total) OVER (
         ORDER BY month
         ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3m
FROM monthly
ORDER BY month;
```
**Screenshot:**
![Aggregate Results](https://github.com/Emmalise1/plsql-window-functions-Iza-Kuradusenge-Emma-Lise/blob/main/aggregate.PNG?raw=true)

**Interpretation**
This query provides total monthly figures, cumulative running totals, and a 3-month moving average. ROWS counts specific physical rows while RANGE groups rows according to values. The moving average presents general trends in streaming.

### Navigation — LAG(), LEAD() (growth % calculations)
**SQL Query (Month-over-Month Growth):**
```sql
WITH monthly AS (
  SELECT TRUNC(stream_date,'MM') AS month,
         SUM(duration) AS monthly_total
  FROM streams
  GROUP BY TRUNC(stream_date,'MM')
)
SELECT month,
       monthly_total,
       LAG(monthly_total) OVER (ORDER BY month) AS prev_month_total,
       LEAD(monthly_total) OVER (ORDER BY month) AS next_month_total,
       CASE
         WHEN LAG(monthly_total) OVER (ORDER BY month) IS NULL THEN NULL
         WHEN LAG(monthly_total) OVER (ORDER BY month) = 0 THEN NULL
         ELSE ROUND(
           (monthly_total - LAG(monthly_total) OVER (ORDER BY month)) * 100
           / LAG(monthly_total) OVER (ORDER BY month),
           2
         )
       END AS mom_pct_change
FROM monthly
ORDER BY month;
```
**Screenshot:**
![Navigation Results](https://raw.githubusercontent.com/Emmalise1/plsql-window-functions-Iza-Kuradusenge-Emma-Lise/2ca5845517775b101c322c58a8f0e5bd3f4d2d4b/navigation.PNG)

**Interpretation**
This query uses the LAG function to obtain the previous month's amount and calculates month-over-month growth. A positive amount indicates streaming growth and a negative amount indicates decline. Similarly, LEAD may also be used to look at the next month in order to forecast.

### Distribution — NTILE(4), CUME_DIST() (segmentation)
**SQL Query (Listener Segmentation):**
```sql
SELECT t.listener_id,
       t.name,
       t.total_duration,
       NTILE(4) OVER (ORDER BY t.total_duration DESC) AS quartile,
       CUME_DIST() OVER (ORDER BY t.total_duration DESC) AS cume_dist
FROM (
  SELECT l.listener_id,
         l.name,
         SUM(s.duration) AS total_duration
  FROM listeners l
  JOIN streams s ON l.listener_id = s.listener_id
  GROUP BY l.listener_id, l.name
) t
ORDER BY t.total_duration DESC;
```
**Screenshot:**
![Distribution Results](https://github.com/Emmalise1/plsql-window-functions-Iza-Kuradusenge-Emma-Lise/blob/main/distribution.PNG?raw=true000)

**Interpretation**
This segregates listeners into quartiles based on their total time streamed using NTILE(4). CUME_DIST calculates the proportion of listeners at or below a certain total time streamed. This is beneficial in identifying top listeners as well as those users who engage with your content the least in order to target them with messages.

# Step 5: GitHub Repository Structure

plsql-window-functions-[Emma Lise]-[Iza Kuradusenge]/
│
├── scripts/              # SQL queries
│   ├── ranking.sql
│   ├── aggregate.sql
│   ├── navigation.sql
│   └── distribution.sql
│
├── images/               # Screenshots
│   ├── ER_Diagram.png
│   ├── ranking.png
│   ├── aggregate.png
│   ├── navigation.png
│   └── distribution.png
│
├── README.md             # Project report
└── LICENSE (optional)

























