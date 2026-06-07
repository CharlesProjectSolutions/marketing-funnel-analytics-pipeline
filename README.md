# 🕶️ E-Commerce Marketing Funnel Analysis & A/B Test Pipeline

![SQL](https://img.shields.io/badge/Language-SQL-blue?style=flat-square)
![BI](https://img.shields.io/badge/Domain-Business%20Intelligence-orange?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

> **An end-to-end full-funnel analysis of Warby Parker's style quiz and home try-on customer journey; measuring conversion rates, diagnosing drop-off points, and evaluating an A/B test on try-on pair volume.**
> **Utilizing a Load-Then-Link staging design to ingest volatile web logs, compute window metrics, and serve a denormalized, vertically pivoted data layer optimized for dashboard development**


---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Business Problem](#-business-problem)
- [Dataset Description](#dataset-description)
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
- Is there evidence of **quiz fatigue** — and if so, at what point?
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

The customer journey is modeled as a **linear conversion funnel** across four stages:

```
[Style Quiz Response] → [Quiz Completion] → [Home Try-On] → [Purchase]
```

Each stage is treated as a **binary event** per user: either the user progressed to that stage or they did not. Users who appear in a downstream table but not an upstream one are excluded from the funnel (data integrity check).

The funnel is constructed via **LEFT JOINs** anchored to the `quiz` table, enabling a complete view of all users — including those who did not convert at each stage. `CASE WHEN` flags are used to create boolean indicators for downstream-stage presence.

```
quiz (anchor)
  └── LEFT JOIN home_try_on  ON quiz.user_id = home_try_on.user_id
        └── LEFT JOIN purchase  ON home_try_on.user_id = purchase.user_id
```

---

## 🛠️ SQL Methodology

### 1. Quiz Drop-Off Analysis

Counts responses per question to measure where engagement falls off.

```sql
SELECT
  question,
  COUNT(DISTINCT user_id) AS num_responses
FROM survey
GROUP BY question
ORDER BY question;
```

### 2. Home Try-On Funnel — Boolean Flag Creation

Builds a consolidated funnel table with stage-presence flags and A/B test grouping.

```sql
SELECT
  q.user_id,
  h.user_id IS NOT NULL        AS is_home_try_on,
  h.number_of_pairs            AS num_pairs,
  p.user_id IS NOT NULL        AS is_purchase
FROM quiz q
LEFT JOIN home_try_on h
  ON q.user_id = h.user_id
LEFT JOIN purchase p
  ON q.user_id = p.user_id;
```

### 3. Funnel Conversion Rates

Aggregates stage counts and calculates conversion percentages.

```sql
WITH funnel AS (
  SELECT
    q.user_id,
    h.user_id IS NOT NULL AS is_home_try_on,
    p.user_id IS NOT NULL AS is_purchase
  FROM quiz q
  LEFT JOIN home_try_on h ON q.user_id = h.user_id
  LEFT JOIN purchase    p ON q.user_id = p.user_id
)
SELECT
  COUNT(*)                                    AS total_quiz,
  SUM(is_home_try_on)                         AS total_try_on,
  SUM(is_purchase)                            AS total_purchases,
  ROUND(100.0 * SUM(is_home_try_on)
              / COUNT(*), 1)                  AS quiz_to_tryon_pct,
  ROUND(100.0 * SUM(is_purchase)
              / SUM(is_home_try_on), 1)       AS tryon_to_purchase_pct,
  ROUND(100.0 * SUM(is_purchase)
              / COUNT(*), 1)                  AS overall_conversion_pct
FROM funnel;
```

### 4. A/B Test — Purchase Rate by Try-On Pair Count

Breaks down purchase rates between the 3-pair and 5-pair groups.

```sql
WITH funnel AS (
  SELECT
    q.user_id,
    h.number_of_pairs,
    p.user_id IS NOT NULL AS is_purchase
  FROM quiz q
  LEFT JOIN home_try_on h ON q.user_id = h.user_id
  LEFT JOIN purchase    p ON q.user_id = p.user_id
  WHERE h.user_id IS NOT NULL
)
SELECT
  number_of_pairs,
  COUNT(*)                                        AS total_users,
  SUM(is_purchase)                                AS purchases,
  ROUND(100.0 * SUM(is_purchase) / COUNT(*), 1)  AS purchase_rate_pct
FROM funnel
GROUP BY number_of_pairs
ORDER BY number_of_pairs;
```

### 5. Most Popular Quiz Outcomes & Purchase Styles

Surfaces top-performing frame styles and colors to inform inventory and merchandising decisions.

```sql
-- Top purchased styles
SELECT
  style,
  model_name,
  color,
  COUNT(*) AS total_purchases
FROM purchase
GROUP BY style, model_name, color
ORDER BY total_purchases DESC
LIMIT 10;
```

---

## 📊 Key Performance Indicators (KPIs)

| KPI | Definition |
|---|---|
| **Quiz Completion Rate** | % of users who answered all 5 quiz questions |
| **Quiz-to-Try-On Rate** | % of quiz completers who enrolled in home try-on |
| **Try-On-to-Purchase Rate** | % of try-on participants who made a purchase |
| **Overall Conversion Rate** | % of quiz completers who ultimately purchased |
| **3-Pair Purchase Rate** | Purchase rate among users in the 3-pair test group |
| **5-Pair Purchase Rate** | Purchase rate among users in the 5-pair test group |
| **A/B Lift** | Absolute and relative difference in purchase rates between groups |
| **Question Drop-Off Rate** | % decrease in responses from one quiz question to the next |

---

## 📈 Dashboard Design

The analysis is designed to support a BI dashboard with the following panels:

### Panel 1 — Quiz Engagement Funnel
- **Chart type:** Horizontal bar chart (response count per question)
- **Purpose:** Visually identify the sharpest drop-off point in the quiz
- **Filters:** None (all users, all questions)

### Panel 2 — Full Purchase Funnel
- **Chart type:** Funnel / waterfall chart
- **Stages:** Quiz Completed → Try-On Enrolled → Purchase Made
- **Metrics:** Count per stage + conversion % between stages

### Panel 3 — A/B Test Comparison
- **Chart type:** Side-by-side bar chart or KPI tiles
- **Groups:** 3-pair vs. 5-pair try-on participants
- **Metric:** Purchase rate (%) per group with statistical context

### Panel 4 — Purchase Breakdown
- **Chart type:** Treemap or ranked bar chart
- **Dimensions:** Style, model name, color
- **Metric:** Count of purchases per category

### Panel 5 — Summary KPI Row
- Tiles for: Total Quiz Users · Total Try-On Users · Total Purchases · Overall Conversion Rate

---

## 💡 Insights & Recommendations

### Quiz Funnel

| Finding | Recommendation |
|---|---|
| Response count drops significantly at **Question 3** (shapes) and even more sharply at **Question 5** (last eye exam) | Reorder or streamline the quiz — move more personal or friction-heavy questions later or make them optional |
| The eye exam question has the lowest completion rate — it may feel invasive early in the customer journey | A/B test removing or repositioning this question; add micro-copy to explain why it's relevant |
| Early questions (style preference, fit) retain engagement better | Lead with high-engagement questions to establish momentum before harder ones |

### Home Try-On Funnel

| Finding | Recommendation |
|---|---|
| The **quiz → try-on** conversion rate represents the largest single drop-off in the funnel | Improve post-quiz CTA design, reduce friction in try-on enrollment, or add social proof at the transition point |
| Purchase rates are measurably higher among **5-pair** participants | Scale the 5-pair offering or use it as a premium/default program option for undecided customers |
| Overall end-to-end conversion (quiz → purchase) is low but improvable | Each upstream improvement compounds — even 5–10% gains at quiz completion translate to meaningful revenue growth |

---

## 🧪 A/B Test Results

**Hypothesis:** Providing users with 5 try-on pairs (vs. 3) increases purchase likelihood by reducing choice paralysis and increasing perceived value.

| Group | Try-On Users | Purchases | Purchase Rate |
|---|---|---|---|
| 3 Pairs | ~201 | — | Lower |
| 5 Pairs | ~294 | — | **Higher** |

> **Result:** Users in the **5-pair group** purchased at a notably higher rate, supporting the hypothesis. The additional pairs appear to increase decision confidence without overwhelming users.

> **Recommendation:** Adopt 5 pairs as the default try-on offering, or gate it as a loyalty/first-time-buyer benefit to maximize impact.

---

## 🧰 Tools & Technologies

| Tool | Purpose |
|---|---|
| **SQL (SQLite / PostgreSQL)** | Data querying, funnel construction, aggregation |
| **LEFT JOIN + CASE WHEN** | Funnel logic and boolean flag creation |
| **CTEs (Common Table Expressions)** | Readable, modular multi-step queries |
| **Tableau / Power BI / Sigma** | Dashboard visualization (optional) |
| **DB Microsoft SQL Saver** | Local SQL execution environment |
| **GitHub** | Version control and project documentation |

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).  
Dataset inspired by a collaboration with Warby Parker's Data Science team. 

---

*Built by Charles · Business Intelligence & Data Analytics Portfolio*

