# Store Sales Performance Dashboard | PostgreSQL → Power BI

End-to-end business intelligence project that connects live retail data from a **PostgreSQL** database to **Power BI**, transforms and models it for scale, and delivers an executive-ready sales dashboard with a full DAX measure library.

> 📌 **Current scope:** January 2025 data (12 stores, 4 states, 2 sales channels, 6 product categories). The data model is deliberately built to receive new monthly extracts without rework — see [Data Modeling](#3-data-modeling) below.


---

## Project Overview

**Business question:** How is the store network performing, where is revenue concentrated, and where is margin being won or lost — broken down by region, store, category, channel, and time?

**What I did:** Connected directly to a PostgreSQL source, built a repeatable Power Query ETL pipeline, designed a scalable data model anchored on a proper Date dimension, wrote 25+ DAX measures (including time-intelligence measures that aren't useful yet on one month of data but will activate automatically as new months load), and built a two-page interactive dashboard covering sales, profit, discounting, returns, and channel performance.

**Why it matters:** This mirrors a real recurring reporting cycle — a business that receives a fresh data extract every month and needs a dashboard that doesn't need to be rebuilt each time. The project is structured around that constraint from the ground up, not bolted on afterward.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Source database | PostgreSQL |
| Connectivity | Power BI native PostgreSQL connector |
| ETL / transformation | Power Query (M) |
| Data modeling | Power BI Data Model (star schema) |
| Calculations | DAX |
| Visualization | Power BI Desktop |

---

## Dashboard Preview

**Page 1 — Sales Performance Overview:** daily trend, weekday vs. weekend, top/bottom stores, profit by store.

![Dashboard overview page](./images/dashboard_overview.png)

**Page 2 — Regional & Category Deep Dive:** region and state performance, channel split, category profitability, store-type volume — with slicers for state and region.

![Dashboard regional view](./images/dashboard_regional_view.png)

---

## Project Workflow

### 1. Data Extraction (PostgreSQL → Power BI)
- Connected Power BI Desktop directly to the PostgreSQL instance using the built-in PostgreSQL connector (**Get Data → Database → PostgreSQL database**), authenticating with the database credentials and selecting **Import** mode.
- Pulled the raw `store_performance` table (one row per store per day) containing transactions, units sold, gross sales, discounts, returns, net sales, COGS, profit, and average ticket — covering 12 stores across 4 states for January 2025.
- Kept the connection parameterized (server/database name as Power BI parameters) so the same report can be pointed at a staging vs. production database without editing every query.

### 2. Data Transformation & Cleaning (Power Query)
Cleaning and shaping was done entirely in Power Query before the data reached the model layer, so the model itself stays fast and simple:
- **Type enforcement** — explicitly set data types on every column (Date, whole number, decimal, text) since PostgreSQL/import defaults aren't always what Power BI infers.
- **Null handling** — identified and addressed rows with missing `Discounts` / `Returns` values (2 of 360 rows) rather than letting them silently break aggregations; documented as a known data-quality flag rather than guessed at.
- **Column standardization** — trimmed and cleaned text fields (`StoreName`, `City`, `Region`, `Category`) to avoid duplicate categories from whitespace/casing inconsistencies.
- **Derived columns removed from the fact table** — calculated fields like `DayOfWeek` were re-derived from the Date dimension instead of trusting the source column, so the model has a single source of truth for date logic (see below).
- **Query folding preserved** where possible, so filtering/grouping steps push back down to PostgreSQL instead of running in Power BI's engine — keeping refreshes fast as more months are appended.

### 3. Data Modeling
Built as a **star schema**, not a single flat table, specifically because this report is designed to keep growing month over month:

- **Fact table:** `Sales` — one row per store, per day, holding the numeric measures (transactions, units, gross sales, discounts, returns, net sales, COGS, profit).
- **Date dimension:** a dedicated calculated `Date` table spanning a full year before and after the data range (not just January), so the model doesn't need to be touched when February, March, etc. are loaded. Marked as the official **Date Table** in Power BI, which is what makes built-in time-intelligence functions (`TOTALYTD`, `SAMEPERIODLASTYEAR`, `DATEADD`, etc.) work correctly. Includes Year, Quarter, Month, Week, Day-of-Week, Weekend flag, and relative-period flags.
- **Relationship:** single-direction, one-to-many from `Date[Date] → Sales[Date]`.
- This structure means: **when next month's extract lands, it's a refresh, not a rebuild.** No new relationships, no re-written measures, no broken visuals — the Date table and every DAX time-intelligence measure just start returning real values instead of one-month placeholders.

### 4. DAX Measures
All measures are written to a dedicated **Measures** table (not scattered across model tables) and organized into logical groups for maintainability:

| Group | Examples |
|---|---|
| Core KPIs | `Total Net Sales`, `Total Profit`, `Profit Margin %`, `Total Transactions`, `Average Ticket Size` |
| Cost & risk | `Total Discounts`, `Discount Rate %`, `Total Returns`, `Return Rate %` |
| Time intelligence | `Net Sales MTD`, `Net Sales YTD`, `Net Sales PY`, `Net Sales YoY %`, `Net Sales 7D Avg`, `Net Sales 30D Avg` |
| Ranking & share | `Store Rank`, `% of Total Sales by Category` |
| Segment comparisons | `Weekday Sales`, `Weekend Sales`, `Online Sales Share %` |

Every measure uses `DIVIDE()` instead of the `/` operator to handle blank/zero denominators safely, and every time-intelligence measure is written against the marked Date table so it will resolve correctly the moment multi-month data is available — no rewriting needed later.


---

## Key Business Insights

## Headline Numbers

| Metric | Value |
|---|---|
| Gross Sales | **$2.54M** |
| Net Sales | **$2.32M** |
| Total Profit | **$769.31K** |
| Profit Margin | **33.16%** |
| Total Transactions | **71.46K** |
| Average Ticket Size | **$32.46** |
| Units Sold | **186.77K** |
| Total Discounts | **$179.50K** (7.07% discount rate) |
| Total Returns | **$39.35K** (1.55% return rate) |

> Returns amount to **$39,350, resulting in a low return rate of just 1.55%**. So, returns aren’t really an issue for us right now.


- **First off, we hit **$2.32 million in net sales** with a solid 33% profit margin. While that sounds good, the real insight lies in figuring out where that revenue is coming from. on 71,460 transactions.
- **Interestingly, one region alone accounts for 54% of our total revenue**. Sure, it's great news for now, but it feels a bit risky for the future if we don’t expand our reach elsewhere.
- **Revenue is concentrated:** the Southwest region alone accounts for 54% of net sales; New Mexico is the top-performing state on profit.
- When it comes to product categories, **grocery sales** topped the charts. However, it also had the lowest profit margins compared to other categories. Apparel and Health actually outperformed grocery in terms of profitability. So sometimes, just because something sells well doesn’t mean it’s your top money-maker.
- **Online sales made up only 21% of revenue** — but had a higher average order value than in-store. That's not a checkout problem; that's a visibility problem.
- **Saturday and Wednesday are the strongest sales days**; Monday is the softest — a potential staffing/scheduling signal.
- **Discounting costs roughly 4.5x more than returns** as a share of gross sales (7.1% vs. 1.6%) — the bigger margin lever is promotional discipline, not returns management.


---
# 📊 Store Performance — Dashboard Insights

A deeper dive into what my 30-day store performance analysis is telling me — where the money's coming from, what's actually profitable, and where the opportunities and risks are hiding.

---

## Table of Contents

- [Revenue Breakdown by Region and State](#revenue-breakdown-by-region-and-state)
- [Sales Channels: In-Store vs Online](#sales-channels-in-store-vs-online)
- [Category Insights: Sales vs Profitability](#category-insights-sales-vs-profitability)
- [Where Am I Getting Volume From?](#where-am-i-getting-volume-from)
- [Store-Level Insights](#store-level-insights)
- [Weekly Trends](#weekly-trends)
- [Key Takeaways for Stakeholders](#key-takeaways-for-stakeholders)

---

## Revenue Breakdown by Region and State

When I dive deeper into where the money is coming from, it's clear the **Southwest region is really driving the ship** with **$1.26 million** in net sales — this accounts for **over half of my total revenue**, echoing the point that one region makes up **54%** of earnings. Specifically, the **South** contributed **$690K** while the **Mountain** region brought in **$380K**.

Breaking it down further by state profits:
- **New Mexico** leads with **$280K**
- **Texas** follows closely at **$230K**
- **Arizona** and **Colorado** each tied at **$130K**

The takeaway here is pretty straightforward: I'm heavily reliant on **one region (the Southwest)** and **one state (New Mexico)** for my profits. It looks like growth opportunities in other areas are still waiting to be tapped rather than being outright weaknesses.

---

## Sales Channels: In-Store vs Online

Now let me compare my sales channels — **In-store has generated $1.84 million** versus **online sales at just $480K**. That means online sales only account for around **21%** of my total revenue, which aligns perfectly with earlier observations.

This backs up the idea that it's more about **visibility than a problem with the checkout process**; online sales may be a smaller piece of the pie now, but there's definitely room to expand that without needing to fix any broken systems.

---

## Category Insights: Sales vs Profitability

When I look at performance by category, it tells quite a different story.

- **Grocery** stands out with the **highest net sales** at around **$1 million** but has a **lower profit margin** compared to other categories.
- On the flip side, **Health** may have lower sales volume but boasts the **highest profit margin** — nearly hitting **40%**. It might not bring in as much cash overall, but it's certainly more efficient.
- Categories like **Home Goods, Apparel, Electronics, and Sports** fall somewhere in between regarding both sales and margins, with some fluctuations here and there.
- Interestingly enough, **Grocery also experiences the highest return volume**, while returns drop off in categories that sell less overall.

So what does this mean? It directly supports the point — **just because something sells well doesn't mean it earns well too**. Grocery attracts customers, but **Health and Apparel tend to be much more profitable per dollar spent**.

---

## Where Am I Getting Volume From?

Looking at my store types, I found that:
- **Urban stores** sold an impressive **126K units** — more than double what **suburban stores** managed (**53K**)
- **Rural stores** barely made it past **10K units**

Clearly, **urban locations are driving most of my volume**.

---

## Store-Level Insights

Now let me get into specifics about store performance:

**Top 5 stores by sales:**
1. Rio Grande Market — $243K
2. Riverwalk Goods — $238K
3. Valley Sun Retail — $234K
4. Lone Star Central — $227K
5. High Desert Outlet — $224K

**Profit by store (highest to lowest):**
- **Rio Grande Market** leads with **$80K** profit
- Followed closely by **Lone Star Central** at **$79K**
- **Valley Sun Retail** at **$78K**
- **Riverwalk Goods** comes next with **$76K**
- **Front Range Mart** adds another solid performance at **$75K**

Interestingly enough though: **Enchanted Circle Store significantly lags behind** all others — it only pulls in about **a third of what my top store makes in profit** — definitely something I plan to examine further regarding potential issues like location or staffing challenges.

Another thing worth noting is that store rankings for sales and profits don't always align perfectly; for example, **Lone Star Central ranks fourth in terms of sales but second in profits** — which serves as a good reminder that **high revenue doesn't automatically mean high profitability**.

---

## Weekly Trends

Shifting gears to weekly patterns, I found some intriguing insights:
- **Saturdays ($360K)** and **Wednesdays ($356K)** are my top sales days — right on par with earlier observations.
- Overall **weekday sales ($1.609M) massively eclipse weekend figures ($711K)** — even though Saturday stands as the best single day for sales, weekdays collectively pull **more than twice as much revenue** compared to weekends.
- When it comes to **returns**: they build steadily from **Monday through midweek** (peaking on **Wednesday/Thursday**), then dip slightly on **Friday** before climbing again leading into the weekend — this could be worth cross-referencing against staffing or fulfillment patterns for those days.

So here's what I'd suggest: when planning staffing or inventory needs, I should focus more on **weekday coverage overall** while ensuring I still have enough support on **Saturday**, since it remains my busiest day by far.

---

## Key Takeaways for Stakeholders

To sum everything up from my analysis of these dashboards:

1. **Revenue is concentrated** primarily within **one region (Southwest)**, **one state (New Mexico)**, and largely from **in-store channels** — which may work efficiently now but poses risks long-term.
2. **What sells best (Grocery) doesn't necessarily earn best (Health/Apparel)** — so managing margins is as crucial as tracking volume.
3. **My online presence appears underutilized rather than ineffective** — a real opportunity for growth moving forward.

---
*Source: 30-day store performance analysis across regions, states, categories, and channels.*



## Data Quality Notes

Transparency matters as much as polish in a real BI project:
- 2 of 360 store-day rows were missing a `Discounts` or `Returns` value. These were treated as $0 in all aggregations rather than dropped or estimated, and flagged for follow-up with the source system rather than silently corrected.
- The dataset currently covers a single month, so month-over-month and year-over-year DAX measures are built and functional but not yet meaningful — they will activate automatically as more monthly extracts are loaded.

---

## What I'd Do With More Time / More Data

- Split `Store`, `Category`, and `Region` out into their own dimension tables once multiple months of data justify a fuller star schema (currently kept lean for a single fact table + Date dimension).
- Add a `Budget`/`Target` table to support variance-to-plan reporting.
- Automate the PostgreSQL → Power BI refresh on a schedule via Power BI Service / Gateway once this moves beyond Desktop.

---

## How to Reproduce This Project

1. Restore the `store_performance` table into a PostgreSQL instance.
2. Open Power BI Desktop → **Get Data → PostgreSQL database** → point to your instance.
3. Apply the Power Query steps documented in `/PowerQuery` (type changes, null handling, text cleaning).
4. Load the `Date` dimension calculated table (DAX script included in `/DAX/date_dimension.dax`) and mark it as the Date Table.
5. Build the relationship `Date[Date] → Sales[Date]`.
6. Import the measures from `/DAX/measures.dax` into a dedicated Measures table.
7. Open `Dashboard.pbix` for the finished report, or rebuild the visuals using the layout described above.

---

## About Me

I'm a data analyst focused on turning raw operational data into dashboards that stakeholders actually trust and act on — not just charts, but the modeling and documentation discipline behind them. This project was built to demonstrate a realistic, recurring reporting workflow end to end: source connection, cleaning, scalable modeling, DAX, and storytelling.

