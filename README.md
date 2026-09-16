# 🛍️ Customer Shopping Behavior Analysis
### From raw transactions to boardroom-ready insight — a full-stack analytics walkthrough

![Python](https://img.shields.io/badge/Python-pandas-3776AB?style=flat-square&logo=python&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?style=flat-square&logo=powerbi&logoColor=black)
![Status](https://img.shields.io/badge/Status-Complete-2ea44f?style=flat-square)

---

## 📖 Table of Contents
- [Business Context](#-business-context)
- [Dataset](#️-dataset)
- [Approach](#-approach)
- [Data Preparation](#-data-preparation-python--pandas)
- [SQL Analysis](#-sql-analysis-10-business-questions)
- [Power BI Dashboard](#-power-bi-dashboard)
- [Key Findings & Business Impact](#-key-findings--business-impact)
- [Recommendations](#-recommendations)
- [Tech Stack](#️-tech-stack)
- [Repository Structure](#-repository-structure)
- [What I'd Extend Next](#-what-id-extend-next)
- [Connect](#-connect)

---

## 🎯 Business Context

A retail team sitting on 3,900 transaction records doesn't need another chart — it needs answers to questions that change decisions: *Who should we target for subscriptions? Which products are quietly bleeding margin through discounts? Is our loyalty base actually loyal, or just large?*

This project treats the dataset as a stakeholder would hand it over — messy, uncleaned, and unopinionated — and builds a complete pipeline to answer those questions, end to end: **Python for preparation → SQL for structured business logic → Power BI for the story stakeholders actually see.**

## 🗃️ Dataset

| Attribute | Detail |
|---|---|
| Volume | 3,900 transactions · 18 fields |
| Demographics | Age, Gender, Location, Subscription Status |
| Purchase details | Item, Category, Amount, Season, Size, Color |
| Behavioral signals | Discount usage, Purchase frequency, Review rating, Shipping type, Purchase history |
| Data quality issue | 37 missing values in `Review Rating` |

## 🧭 Approach

The project follows a deliberate three-stage pipeline rather than jumping straight to a dashboard — each stage exists to catch a different class of problem before it reaches the visual layer:

1. **Clean and engineer in Python** — where messy, inconsistent data actually gets fixed
2. **Interrogate in SQL** — where business questions get formal, reproducible answers
3. **Visualize in Power BI** — where the answers become something a non-technical stakeholder can filter and explore themselves

## 🔧 Data Preparation (Python / pandas)

- **Structural audit first** — `df.info()`, `df.describe(include='all')`, and a null-value scan before touching a single row, to avoid "fixing" a problem that wasn't actually there
- **Category-aware imputation** — the 37 missing `Review Rating` values were filled with each product **category's own median**, not a global one, so a missing rating on a $200 coat isn't quietly replaced by the median rating of a $15 sock
- **Redundancy check, not assumption** — rather than assuming `discount_applied` and `promo_code_used` were duplicates, `(df['discount_applied'] == df['promo_code_used']).all()` confirmed it *before* dropping the column
- **Feature engineering that earns its place in SQL later:**
  - `age_group` — quartile-based bins (`Young Adult`, `Adult`, `Middle-aged`, `Senior`) so age becomes a usable segment, not just a raw number
  - `purchase_frequency_days` — mapped text labels like *Fortnightly*, *Bi-Weekly*, *Every 3 Months* into a single numeric day-count scale, unlocking frequency as something that can be averaged, ranked, or compared
- **Column standardization** — full snake_case rename (`Purchase Amount (USD)` → `purchase_amount`) for clean SQL compatibility downstream
- Cleaned DataFrame loaded into **MySQL via SQLAlchemy**, handing off from pandas to a query-able relational table

## 🗄️ SQL Analysis (10 Business Questions)

Each query was written to answer a specific stakeholder question — not just to demonstrate syntax:

| # | Business Question | SQL Technique |
|---|---|---|
| 1 | Revenue split — male vs. female customers | `GROUP BY` + `SUM` |
| 2 | Which discount users *still* spend above average? | Correlated subquery |
| 3 | Top 5 products by average review rating | `ORDER BY` + `LIMIT` |
| 4 | Standard vs. Express shipping — spend comparison | Conditional filtering |
| 5 | Do subscribers actually spend more? | Aggregation + multi-metric comparison |
| 6 | Top 5 most discount-dependent products | Conditional percentage calculation |
| 7 | Segment customers: New / Returning / Loyal | CTE + `CASE` logic |
| 8 | Top 3 products *within each* category | `ROW_NUMBER() OVER (PARTITION BY ...)` |
| 9 | Are repeat buyers more likely to subscribe? | Filtered aggregation |
| 10 | Revenue contribution by age group | `GROUP BY` + ranked aggregation |

**Sample — surfacing the top product per category without a self-join:**
```sql
WITH item_counts AS (
    SELECT category, item_purchased,
           COUNT(customer_id) AS total_orders,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY COUNT(customer_id) DESC) AS item_rank
    FROM customer
    GROUP BY category, item_purchased
)
SELECT item_rank, category, item_purchased, total_orders
FROM item_counts
WHERE item_rank <= 3;
```

## 📊 Power BI Dashboard

An interactive, slicer-driven dashboard so the answers above aren't locked in static query output — anyone can filter by **gender, category, subscription status, or shipping type** and watch the numbers recompute live.

**Headline metrics:**
- 👥 **3.9K** customers analyzed
- 💵 **$59.76** average purchase amount
- ⭐ **3.75** average review rating
- 🔁 Only **27%** of customers are active subscribers

Panels include revenue and sales by category, revenue and sales by age group, and subscription-status breakdown — built to answer "so what should we *do*" at a glance, not just "here's what happened."

## 💡 Key Findings & Business Impact

| Finding | What It Means |
|---|---|
| **3,116 of 3,900 customers are "Loyal"** (>10 prior purchases), but only 958 repeat buyers (>5 purchases) are subscribed | Loyalty and subscription are two different behaviors — a large loyal base isn't automatically converting into recurring revenue |
| **Male customers generated $157.9K vs. $75.2K from female customers** | Revenue is heavily gender-skewed — worth investigating whether this reflects category mix, basket size, or marketing reach |
| **Hats, Sneakers, and Coats carry the highest discount dependency (~48–50%)** | These products may be structurally underpriced or over-promoted — a margin risk hiding in plain sight |
| **Young Adults lead revenue ($62.1K)** despite Loyal customers spanning every age bracket | Acquisition and retention likely need *different* playbooks by age segment, not one blanket strategy |
| **Express shipping customers spend marginally more** ($60.48 vs. $58.46 average) | A small but real signal that shipping speed correlates with basket value — worth testing as a conversion lever |

## 📌 Recommendations

- **Close the loyalty–subscription gap** — target the 2,518 repeat buyers not yet subscribed with a conversion-focused offer
- **Audit discount-dependent SKUs** (Hat, Sneakers, Coat, Sweater, Pants) — confirm the discounts are driving incremental volume, not just subsidizing sales that would've happened anyway
- **Segment marketing spend by age group** rather than treating the customer base as one cohort
- **Test express shipping as an upsell**, not just a fulfillment option, given its correlation with higher spend

## 🛠️ Tech Stack

`Python` · `pandas` · `MySQL` · `SQLAlchemy` · `Power BI` · `DAX` · `Jupyter Notebook`

## 📁 Repository Structure

customer-shopping-behavior-analysis/
--├── README.md
--├── customer_shopping_behavior.csv
--├── Customer-shopping-behaviour.ipynb
--├── Customer Shopping Behaviour Analysis.sql
--├── Customer Behaviour Analysis.pbix
--└── Customer-Shopping-Behavior-Analysis.pptx

## 🔭 What I'd Extend Next

- A cohort-based churn model to flag at-risk "Loyal" customers before they lapse
- A/B test design for the express-shipping-as-upsell hypothesis
- Category-level price elasticity analysis to replace discount guesswork with data

## 📬 Connect

- **Portfolio:** [sindhuportfolio-ab4p.vercel.app](https://sindhuportfolio-ab4p.vercel.app/)
- **GitHub:** [github.com/sindhumanoharan1210](https://github.com/sindhumanoharan1210)
- **LinkedIn:** [linkedin.com/in/sindhumanoharan](https://www.linkedin.com/in/sindhumanoharan/)

---

*Sindhu M —  Data & BI Analyst*

