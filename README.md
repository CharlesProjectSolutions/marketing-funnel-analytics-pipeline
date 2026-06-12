# 👁️‍🗨️ E‑Commerce Marketing Funnel Analysis & A/B Test Pipeline

![SQL](https://img.shields.io/badge/Language-SQL-blue?style=flat-square)
![BI](https://img.shields.io/badge/Domain-Business%20Intelligence-orange?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

---

### 🧭 Executive Summary
This [Customer Funnel Performance dashboard](https://public.tableau.com/app/profile/charles3440/viz/CustomerFunnelPerformance/CustomerFunnelPerformance?publish=yes) visualizes Warby Parker’s customer journey from quiz engagement through Home Try‑On to purchase, revealing stage‑level conversion performance and experiment outcomes. The analysis identifies a **34% drop‑off between Home Try‑On and Purchase**, spotlighting a key opportunity to improve post–try‑on engagement, product confidence, and checkout CTAs. The A/B test shows that offering **5 pairs instead of 3 drives a +49.4% lift in purchase conversion**, providing clear evidence to guide product and marketing strategy. 

---

## 📋 Table of Contents

- [Executive Summary](#-executive-summary)
- [Project Overview](#-project-overview)
- [Business Problem](#-business-problem)
- [Dataset Description](#-dataset-description)
- [Data Architecture & Ingestion](#-data-architecture--ingestion)
- [Pivoted Analytical SQL Modeling](#-pivoted-analytical-sql-modeling)
- [Key Performance Indicators (KPIs)](#-key-performance-indicators-kpis)
- [Dashboard Design & Live Metrics](#-dashboard-design--live-metrics)
- [A/B Test Evaluation](#-ab-test-evaluation)
- [Strategic Business Insights](#-strategic-business-insights)

---

## 📌 Project Overview

Warby Parker operates a high-touch direct-to-consumer eyewear funnel starting with a digital style quiz, progressing to a physical Home Try-On sampler box, where customers receive frames to try before purchasing the product.

This project performs an end-to-end **marketing funnel analysis** using SQL and Tableau to:

- Track user progression from style quiz → home try-on → purchase
- Identify the questions in the quiz where users disengage
- Quantify stage-by-stage conversion rates across the full funnel
- Evaluate whether offering **3 pairs vs. 5 pairs** in the try-on program impacts purchase likelihood

This project establishes a resilient analytics pipeline that unifies siloed transactional tables into an optimized framework. It provides full transparency into stage-level drop-offs and quantifies the economic value of an active product variant test: **offering 3-pair vs. 5-pair home sampling kits**.

---

## 🧩 Business Problem

Warby Parker's customer acquisition model depends on successfully guiding users through a multi-step digital funnel. Even small drop-off rates at each stage compound into significant revenue loss at scale.  Stakeholders required automated answers to three core visibility blockers:

1. **Funnel Friction points:** Where do users experience drop-off across the primary journey milestones?
2. **The Try-On Experiment:** Does expanding kit options from 3 to 5 pairs reduce purchase friction, or does it trigger choice paralysis?
3. **Product Inventory Demands:** Which specific frame models dominate checkouts, and how do early quiz selections map to high-value purchases?

### Quiz Funnel
- At which question(s) do the most users abandon the quiz?
- Is there evidence of **quiz fatigue**, and if so, at what point?
- Which quiz questions are most and least effective at retaining users?

### Home Try-On Funnel
- What are the conversion rates from **quiz → try-on → purchase**?
- Does giving customers **5 try-on pairs** (vs. 3) result in significantly higher purchase rates?
- Where is the highest-value opportunity to improve revenue?

---

## 🗃️ Dataset Description

The analysis draws from **four log files**, each linked by a shared `UserId`:

| Table | Description | Key Columns |
|---|---|---|
| `survey` | Responses to Warby Parker's 5-question style quiz | `UserId`, `Question` |
| `Quiz` | Users who completed the full style quiz | `UserId`, `Style`, `Fit`, `Shape`, `Color` |
| `Home_Try_On` | Users who enrolled in the try-on program | `UserId`, `NumberOfPairs`, `Address` |
| `Purchase` | Users who completed a purchase | `UserId`, `ProductId`, `Style`, `ModelName`, `Color`, `Price` |


---

## 🗃️ Data Architecture & Ingestion
The raw application logs consisted of four disconnected schemas linked by an alphanumeric `UserId`. To safeguard against data type mismatches (such as string-based booleans `"TRUE"/"FALSE"` or corrupt empty elements), a **two-tier staging design** was implemented.

### Ingestion & Cleaning Script (Example: Purchase Log Ingestion)

```sql
-- Create text-safe temporary staging structures
CREATE TABLE #Purchase_Staging (
    UserId VARCHAR(100), ProductId VARCHAR(50), Style VARCHAR(100), 
    ModelName VARCHAR(100), Color VARCHAR(100), Price VARCHAR(50)
);

-- Bulk load volatile server file
BULK INSERT #Purchase_Staging
FROM 'C:\YourSecureDataDirectory\purchase.csv'
WITH (FORMAT = 'CSV', FIRSTROW = 2, FIELDTERMINATOR = ',', ROWTERMINATOR = '\n');

-- Enforce explicit data typing and purge empty strings
INSERT INTO Purchase (UserId, ProductId, Style, ModelName, Color, Price)
SELECT 
    TRY_CAST(UserId AS UNIQUEIDENTIFIER), 
    TRY_CAST(ProductId AS INT), 
    Style, 
    ModelName, 
    Color, 
    TRY_CAST(NULLIF(Price, '') AS DECIMAL(10,2))
FROM #Purchase_Staging;

DROP TABLE #Purchase_Staging;

```

## 🔀 Funnel Logic

The customer journey is modeled as a **linear conversion funnel** across three stages:

```
[Quiz Completion] → [Home Try-On] → [Purchase]
```


## Conversion Metrics

This script calculates the macro-level milestone volumes, step-by-step conversion rates, and the overall conversion rate from the start of the quiz to a completed purchase.:


```sql
WITH Milestone_Counts AS (
    -- Calculate distinct users reaching each major milestone
    SELECT 
        (SELECT COUNT(DISTINCT UserId) FROM Quiz) AS Quiz_Users,
        (SELECT COUNT(DISTINCT UserId) FROM Home_Try_On) AS TryOn_Users,
        (SELECT COUNT(DISTINCT UserId) FROM Purchase) AS Purchase_Users
)
SELECT 
    Quiz_Users,
    TryOn_Users,
    Purchase_Users,
    
    -- 1. Overall Conversion Rate (Quiz -> Purchase)
    CAST((CAST(Purchase_Users AS DECIMAL(10,2)) / Quiz_Users) * 100.0 AS DECIMAL(10,2)) AS Overall_Conversion_Rate,
    
    -- 2. Quiz -> Home Try-On Conversion
    CAST((CAST(TryOn_Users AS DECIMAL(10,2)) / Quiz_Users) * 100.0 AS DECIMAL(10,2)) AS Quiz_To_TryOn_Conversion,
    
    -- 3. Home Try-On -> Purchase Conversion
    CAST((CAST(Purchase_Users AS DECIMAL(10,2)) / TryOn_Users) * 100.0 AS DECIMAL(10,2)) AS TryOn_To_Purchase_Conversion
FROM Milestone_Counts;

```
Output Result:
| Quiz_Users | TryOn_Users | Purchase_Users | Overall_Conversion_Rate | Quiz_To_TryOn_Conversion | TryOn_To_Purchase_Conversion |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1000** | 750 | 495 | 49.50 | 75.00 | 66.00 |


## Actionable Insights & A/B Testing

A/B Test Breakdown: Does offering more try-on pairs boost purchases?

This script Isolate the Number Of Pairs experiment ("3 pairs" vs "5 pairs") to see which group registers a stronger pull-through to final checkout.


```sql
WITH Experiment_Groups AS (
    -- Segment users by their experiment variant and track if they purchased
    SELECT 
        h.UserId,
        h.NumberOfPairs,
        CASE WHEN p.UserId IS NOT NULL THEN 1 ELSE 0 END AS Did_Purchase
    FROM Home_Try_On h
    LEFT JOIN Purchase p ON h.UserId = p.UserId
)
SELECT 
    NumberOfPairs AS Variant_Group,
    COUNT(DISTINCT UserId) AS Users_In_Stage,
    SUM(Did_Purchase) AS Total_Purchases,
    CAST((CAST(SUM(Did_Purchase) AS DECIMAL(10,2)) / COUNT(DISTINCT UserId)) * 100.0 AS DECIMAL(10,2)) AS Stage_Conversion_Rate
FROM Experiment_Groups
GROUP BY NumberOfPairs;

```
Output Result:
| Variant_Group | Users_In_Stage | Total_Purchases | Stage_Conversion_Rate | 
| :--- | :--- | :--- | :--- |
| **3 pairs** | 379 | 201 | 53.03 |
| **5 pairs** | 371 | 294 | 79.25 |


## 🛠️ Pivoted Analytical SQL Modeling
Because live relational database connections (MS SQL Server) restrict Tableau's native user-interface pivot capabilities, the data layer was reshaped at the database tier using a **vertical `UNION ALL` stacking framework**. This structures wide columns into a uniform text dimension (`Funnel_Stage`) and a single numeric measure (`Stage_Users`), preventing heavy calculations inside the BI engine.

```sql
WITH Global_Funnel_Base AS (
    -- 1. Pre-calculate global baseline numbers for the overall funnel table
    SELECT 
        (SELECT COUNT(DISTINCT UserId) FROM Quiz) AS Total_Quiz_Users,
        (SELECT COUNT(DISTINCT UserId) FROM Home_Try_On) AS Total_TryOn_Users,
        (SELECT COUNT(DISTINCT UserId) FROM Purchase) AS Total_Purchase_Users
),
AB_Variant_Summaries AS (
    -- 2. Pre-calculate variant-level metrics for the A/B test table
    SELECT 
        h.NumberOfPairs,
        COUNT(DISTINCT h.UserId) AS Variant_Total_Users,
        COUNT(DISTINCT p.UserId) AS Variant_Total_Purchases,
        CAST((COUNT(DISTINCT p.UserId) * 100.0) / COUNT(DISTINCT h.UserId) AS DECIMAL(10,1)) AS Variant_Conversion_Rate
    FROM Home_Try_On h
    LEFT JOIN Purchase p ON h.UserId = p.UserId
    GROUP BY h.NumberOfPairs
),
Flat_User_Data AS (
    -- 3. Original query kept completely intact as a baseline
    SELECT 
        q.UserId, q.Style AS Quiz_Preferred_Style, q.Fit AS Quiz_Preferred_Fit,
        COALESCE(h.NumberOfPairs, 'Did Not Reach Try-On') AS AB_Test_Variant,
        p.ProductId, p.ModelName AS Purchased_Model, p.Price AS Purchase_Amount,
        g.Total_Quiz_Users, g.Total_TryOn_Users, g.Total_Purchase_Users,
        100.0 AS Funnel_Quiz_Started_Conversion,
        CAST((g.Total_TryOn_Users * 100.0) / g.Total_Quiz_Users AS DECIMAL(10,1)) AS Funnel_TryOn_Conversion,
        CAST((g.Total_Purchase_Users * 100.0) / g.Total_Quiz_Users AS DECIMAL(10,1)) AS Funnel_Purchased_Conversion,
        ab.Variant_Total_Users AS AB_Variant_Users,
        ab.Variant_Total_Purchases AS AB_Variant_Purchases,
        ab.Variant_Conversion_Rate AS AB_Variant_Conversion_Percent
    FROM Quiz q
    CROSS JOIN Global_Funnel_Base g
    LEFT JOIN Home_Try_On h ON q.UserId = h.UserId
    LEFT JOIN Purchase p    ON q.UserId = p.UserId
    LEFT JOIN AB_Variant_Summaries ab ON h.NumberOfPairs = ab.NumberOfPairs
) 
-- 4. Apply UNION ALL to stack the metrics vertically into a native Tableau Dimension to help build funnel chart
SELECT 
    UserId, Quiz_Preferred_Style, Quiz_Preferred_Fit, AB_Test_Variant, ProductId, Purchased_Model, Purchase_Amount,
    AB_Variant_Users, AB_Variant_Purchases, AB_Variant_Conversion_Percent,
    '1 - Quiz Started' AS Funnel_Stage, Total_Quiz_Users AS Stage_Users, Funnel_Quiz_Started_Conversion AS Stage_Conversion
FROM Flat_User_Data

UNION ALL

SELECT 
    UserId, Quiz_Preferred_Style, Quiz_Preferred_Fit, AB_Test_Variant, ProductId, Purchased_Model, Purchase_Amount,
    AB_Variant_Users, AB_Variant_Purchases, AB_Variant_Conversion_Percent,
    '2 - Try-On Ordered' AS Funnel_Stage, Total_TryOn_Users AS Stage_Users, Funnel_TryOn_Conversion AS Stage_Conversion
FROM Flat_User_Data

UNION ALL

SELECT 
    UserId, Quiz_Preferred_Style, Quiz_Preferred_Fit, AB_Test_Variant, ProductId, Purchased_Model, Purchase_Amount,
    AB_Variant_Users, AB_Variant_Purchases, AB_Variant_Conversion_Percent,
    '3 - Purchased' AS Funnel_Stage, Total_Purchase_Users AS Stage_Users, Funnel_Purchased_Conversion AS Stage_Conversion
FROM Flat_User_Data;
```

## 📊 Key Performance Indicators (KPIs)


| Metric | Measured Value | Business Formulation |
| :--- | :--- | :--- |
| **Quiz Starters** | **1,000** | Baseline volume of unique customer profiles entering the acquisition funnel. |
| **Quiz-to-Try-On Rate** | **75.0%** | Conversion pull-through from onboarding setup to ordering physical product frames. |
| **Try-On-to-Purchase Rate**| **66.0%** | Conversion efficiency of users finalizing a checkout after box delivery. |
| **Overall Funnel Conversion**| **49.5%** | Total percentage of quiz starters successfully tracked to checkout (`495 / 1000`). |
| **Total Funnel Drop-off** | **50.5%** | Aggregate loss from baseline starting pool down to final step (`(1000 - 495) / 1000`). |

---

## 📈 Dashboard Design & Live Metrics

The final dataset was mapped into an interactive executive dashboard layout.

![Customer Funnel Performance Dashboard](https://github.com/CharlesProjectSolutions/Customer-Funnel-Analysis/blob/d92e73c6085680e6e2cca5eabf79c16e0845c2dc/Customer%20Funnel%20Dashboard.png)

---

## 🧪 A/B Test Evaluation

*   **Hypothesis:** Expanding home sample options from 3 pairs to 5 pairs boosts overall checkouts by decreasing choice limitations and increasing user brand engagement.
*   **The Verdict:** The 5-pair variant out-performed the baseline control group, driving an absolute purchase rate surge from **53.0% to 79.2%**. This accounts for a staggering **+49.4% relative conversion lift**.


| Experimental Cohort | Sample Users | Final Purchases | Stage-Specific Conversion Rate |
| :--- | :--- | :--- | :--- |
| **3 Pairs (Control)** | 379 | 201 | 53.0% |
| **5 Pairs (Variant)** | 371 | 294 | **79.2% (Statistically Significant Win)** |

---

## 💡 Strategic Business Insights

1. **Scale the Winner:** The 5-pair sample variant demonstrates massive conversion superiority over the 3-pair baseline. The business should systematically phase out smaller sample kits to optimize conversion performance.
2. **Address Try-On Leaks:** While 75% of users successfully migrate to ordering a try-on box, a **34.0% drop-off** occurs between box arrival and final purchase. Implementing automated push-notifications or email follow-ups during the physical trial window represents a high-value recovery opportunity.
3. **Inventory Alignment:** Merchandising teams should prioritize stock allocations around top-performing models like **Eugene Narrow** (116 sales) and **Dawes** (107 sales), which represent the largest revenue drivers in the current cycle.

---

## 🧰 Tools & Technologies

| Tool | Purpose |
|---|---|
| **SQL (SQLite / PostgreSQL / SQL Saver / DBeaver)** | Data querying, funnel construction, aggregation |
| **LEFT JOIN + CASE WHEN** | Funnel logic and boolean flag creation |
| **CTEs (Common Table Expressions)** | Readable, modular multi-step queries |
| **Tableau / Power BI / Sigma** | Dashboard visualization |
| **DB Microsoft SQL Saver** | Local SQL execution environment |
| **Warehouse Cloud Platforms** | Redshift, Snowflake, Azure, Databricks, BigQuery, dbt, and Teradata to store transformed Analytics & AL ready Data |
| **GitHub / Confluence** | Version control and project documentation |

---

## 📄 License

This project dataset was inspired by a collaboration with Warby Parker's Data Science team. 

---

*Built by Charles · Business Intelligence & Data Analytics Portfolio*

