> Updated: Sep 2026

![E-commerce Web Analytics dashboard](assets/ecommerce-web-analytics-hero.png)

# 📊 E-commerce Performance Analysis | BigQuery SQL

This portfolio project uses SQL to analyse user behaviour, traffic quality, conversion, product performance, and revenue for the **Google Merchandise Store**. The goal is to translate web-analytics data into practical questions that an e-commerce and marketing team can act on.

---

## 📑 Table of Contents

1. [Background & Overview](#-background--overview)
2. [Dataset Description & Data Structure](#-dataset-description--data-structure)
3. [Approach](#-approach)
4. [SQL Analysis](#️-sql-analysis)
5. [Key Takeaways](#-key-takeaways)
6. [How to Run](#-how-to-run)

---

## 📌 Background & Overview

### Objective

E-commerce teams need to understand where visitors come from, whether they engage with the site, and which actions ultimately lead to purchases. This project answers ten business questions using Google Analytics session data in BigQuery.

- Track monthly visits, pageviews, and transactions.
- Evaluate traffic-source quality using bounce rate and conversion rate.
- Analyse revenue by source, period, and device.
- Compare purchaser behaviour with non-purchaser behaviour.
- Identify product cross-sell opportunities and funnel performance.
- Monitor weekly and cumulative revenue.

### Who is this project for?

- Data analysts and business analysts
- E-commerce, growth, and digital-marketing teams
- Product managers and decision-makers

---

## 📂 Dataset Description & Data Structure

### Data source

- **Source:** BigQuery public dataset: `bigquery-public-data.google_analytics_sample.ga_sessions_*`
- **Domain:** E-commerce web analytics (Google Merchandise Store)
- **Grain:** One row represents a Google Analytics session. Several fields, including `hits` and `hits.product`, are nested/repeated records.
- **Period analysed:** Primarily January–July 2017, depending on the business question.

### Table structure

This project uses one table family, partitioned by date: `ga_sessions_YYYYMMDD`. The following fields are the ones used in the analysis.

| Field | Data type | Description / use in this project |
|---|---:|---|
| `fullVisitorId` | STRING | Unique visitor identifier; used to identify buyers. |
| `date` | STRING | Session date in `YYYYMMDD` format. |
| `totals.visits` | INTEGER | Number of sessions with interaction events. |
| `totals.pageviews` | INTEGER | Number of pageviews within a session. |
| `totals.bounces` | INTEGER | Equals 1 for a bounced session; otherwise null. |
| `totals.transactions` | INTEGER | Number of e-commerce transactions within a session. |
| `trafficSource.source` | STRING | Traffic origin, such as Google, direct, or a referral site. |
| `device.deviceCategory` | STRING | Device category: desktop, mobile, or tablet. |
| `hits` | RECORD (repeated) | Hit-level information within each session. |
| `hits.eCommerceAction.action_type` | STRING | E-commerce action: product view (2), add to cart (3), purchase (6), etc. |
| `hits.product` | RECORD (repeated) | Product-level information associated with a hit. |
| `hits.product.v2ProductName` | STRING | Product name. |
| `hits.product.productQuantity` | INTEGER | Quantity purchased. |
| `hits.product.productRevenue` | INTEGER | Revenue in micros; divide by 1,000,000 for standard currency units. |

### Working with nested data

`hits` and `hits.product` are repeated fields. The queries use `UNNEST()` to turn them into rows before accessing hit- or product-level attributes. This is essential for revenue, product, and funnel analysis.

---

## 🧠 Approach

1. **Understand the schema** — identify session-level fields and nested hit/product fields.
2. **Prepare dates and filters** — use table suffixes and `PARSE_DATE()` / `FORMAT_DATE()` for the required periods.
3. **Build metrics** — aggregate visits, bounces, pageviews, transactions, product actions, and revenue.
4. **Analyse performance** — compare sources, devices, purchaser segments, product combinations, and funnel stages.
5. **Translate results into actions** — highlight how the metrics can support marketing, UX, and merchandising decisions.

---

## ⚒️ SQL Analysis

### Query 01 — Monthly traffic and transactions

**Business question:** What were total visits, pageviews, and transactions in January–March 2017?

```sql
SELECT
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
  SUM(totals.visits) AS visits,
  SUM(totals.pageviews) AS pageviews,
  SUM(totals.transactions) AS transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _TABLE_SUFFIX BETWEEN '0101' AND '0331'
GROUP BY 1
ORDER BY 1;
```

**Expected output (excerpt):**

| month | visits | pageviews | transactions |
|---|---:|---:|---:|
| 201701 | 64,694 | 257,708 | 713 |
| 201702 | 62,192 | 233,373 | 733 |
| 201703 | 69,931 | 259,522 | 993 |

March recorded the highest transaction count in the three-month period.

**Observation & finding:** Visits and pageviews recovered in March after declining in February, while transactions rose each month. This suggests improving commercial performance; the next step would be to identify which campaigns, products, or site changes contributed to March's uplift.

---

### Query 02 — Bounce rate by traffic source

**Business question:** Which July 2017 traffic sources generated visits, and how much of that traffic bounced?

```sql
SELECT
  trafficSource.source AS source,
  SUM(totals.visits) AS total_visits,
  SUM(totals.bounces) AS total_no_of_bounces,
  100.0 * SUM(totals.bounces) / SUM(totals.visits) AS bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY 1
ORDER BY total_visits DESC;
```

**Expected output (excerpt):**

| source | total_visits | total_no_of_bounces | bounce_rate |
|---|---:|---:|---:|
| google | 38,400 | 19,798 | 51.56% |
| (direct) | 19,891 | 8,606 | 43.27% |
| youtube.com | 6,351 | 4,238 | 66.73% |
| analytics.google.com | 1,972 | 1,064 | 53.96% |

**Observation & finding:** Google was the largest traffic source but more than half of its sessions bounced (51.56%). Direct traffic generated a sizeable volume with a materially lower bounce rate (43.27%), indicating more engaged visitors. YouTube had the highest bounce rate among the listed major sources (66.73%), so its targeting, landing page, and message-to-page consistency should be reviewed before scaling spend.

---

### Query 03 — Revenue by source, month, and week

**Business question:** How much June 2017 revenue did each traffic source generate by month and by week?

```sql
WITH month_data AS (
  SELECT
    'Month' AS time_type,
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS time,
    trafficSource.source AS source,
    SUM(product.productRevenue) / 1000000 AS revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST(hits) AS hit,
    UNNEST(hit.product) AS product
  WHERE product.productRevenue IS NOT NULL
  GROUP BY 1, 2, 3
),
week_data AS (
  SELECT
    'Week' AS time_type,
    FORMAT_DATE('%Y%W', PARSE_DATE('%Y%m%d', date)) AS time,
    trafficSource.source AS source,
    SUM(product.productRevenue) / 1000000 AS revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    UNNEST(hits) AS hit,
    UNNEST(hit.product) AS product
  WHERE product.productRevenue IS NOT NULL
  GROUP BY 1, 2, 3
)
SELECT * FROM month_data
UNION ALL
SELECT * FROM week_data
ORDER BY revenue DESC;
```

**Expected output (excerpt):**

| time_type | time | source | revenue |
|---|---:|---|---:|
| Month | 201706 | (direct) | 97,333.619695 |
| Week | 201724 | (direct) | 30,908.909927 |
| Week | 201725 | (direct) | 27,295.319924 |
| Month | 201706 | google | 18,757.179920 |

This view helps compare channel contribution at two time granularities and detect weekly shifts that a monthly total may hide.

**Observation & finding:** Direct traffic was the leading revenue source in the displayed June results, contributing 97.3K for the month. Its revenue was not evenly distributed by week—30.9K in week 24 versus 27.3K in week 25—so the team should investigate the marketing or customer-behaviour drivers behind larger weekly movements.

---

### Query 04 — Conversion rate by traffic source

**Business question:** Which traffic sources had the strongest conversion rate in 2017, among sources with at least 50 transactions?

```sql
SELECT
  trafficSource.source AS source,
  SUM(totals.visits) AS visits,
  SUM(totals.transactions) AS transactions,
  ROUND(100.0 * SUM(totals.transactions) / SUM(totals.visits), 2) AS conversion_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
GROUP BY 1
HAVING SUM(totals.transactions) >= 50
ORDER BY conversion_rate DESC;
```

**Expected output (excerpt):**

| source | visits | transactions | conversion_rate |
|---|---:|---:|---:|
| dfa | 2,728 | 72 | 2.64% |
| (direct) | 189,447 | 4,697 | 2.48% |
| google | 179,804 | 1,661 | 0.92% |

Filtering out very low-volume sources avoids making decisions based on unstable conversion rates.

**Observation & finding:** Among sources with at least 50 transactions, `dfa` had the highest conversion rate (2.64%), narrowly ahead of direct traffic (2.48%). Google delivered almost as many visits as direct traffic but converted at only 0.92%, suggesting an opportunity to improve campaign intent, audience segmentation, or Google-specific landing pages.

---

### Query 05 — Pageviews: purchasers vs. non-purchasers

**Business question:** In June and July 2017, did purchasers view more pages on average than non-purchasers?

```sql
WITH visitor_month AS (
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    fullVisitorId,
    SUM(IFNULL(totals.pageviews, 0)) AS total_pageviews,
    MAX(IF(totals.transactions >= 1, 1, 0)) AS made_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
  WHERE _TABLE_SUFFIX BETWEEN '0601' AND '0731'
  GROUP BY 1, 2
)
SELECT
  month,
  ROUND(AVG(IF(made_purchase = 1, total_pageviews, NULL)), 2) AS avg_pageviews_purchase,
  ROUND(AVG(IF(made_purchase = 0, total_pageviews, NULL)), 2) AS avg_pageviews_non_purchase
FROM visitor_month
GROUP BY 1
ORDER BY 1;
```

**Expected output:**

| month | avg_pageviews_purchase | avg_pageviews_non_purchase |
|---|---:|---:|
| 201706 | 94.02 | 316.87 |
| 201707 | 124.24 | 334.06 |

A meaningful gap between segments can indicate that browsing depth is associated with purchase intent and may guide onsite-content or recommendation improvements.

**Observation & finding:** The output shows higher average pageviews for the non-purchaser segment in both months. This does not mean more browsing causes non-purchase, but it may signal visitors struggling to find the right product or information. The team should examine product discovery, search, filtering, price clarity, and checkout hand-off for this segment.

---

### Query 06 — Average transactions per purchasing user

**Business question:** How many transactions did a purchasing user make on average in July 2017?

```sql
WITH buyer_transactions AS (
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    fullVisitorId,
    SUM(totals.transactions) AS transactions
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
  WHERE totals.transactions >= 1
  GROUP BY 1, 2
)
SELECT
  month,
  ROUND(AVG(transactions), 2) AS avg_transactions_per_purchasing_user
FROM buyer_transactions
GROUP BY 1;
```

**Expected output:**

| month | avg_total_transactions_per_user |
|---|---:|
| 201707 | 4.16 |

This metric distinguishes repeat purchasing behaviour from transaction volume alone.

**Observation & finding:** Purchasing users averaged 4.16 transactions in July. This is a strong repeat-purchase signal for a sample e-commerce store and supports testing retention tactics such as post-purchase recommendations, replenishment reminders, and loyalty offers.

---

### Query 07 — Revenue contribution by device

**Business question:** Which device categories contributed the most revenue in 2017?

```sql
WITH revenue_by_device AS (
  SELECT
    device.deviceCategory AS device,
    SUM(product.productRevenue) / 1000000 AS revenue_by_device
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits) AS hit,
    UNNEST(hit.product) AS product
  WHERE totals.transactions >= 1
    AND product.productRevenue IS NOT NULL
  GROUP BY 1
)
SELECT
  device,
  revenue_by_device,
  SUM(revenue_by_device) OVER () AS total_revenue,
  ROUND(100.0 * revenue_by_device / SUM(revenue_by_device) OVER (), 2) AS revenue_ratio
FROM revenue_by_device
ORDER BY revenue_ratio DESC;
```

**Expected output:**

| device | revenue_by_device | total_revenue | revenue_ratio |
|---|---:|---:|---:|
| desktop | 992,391.507057 | 1,029,315.866848 | 96.41% |
| mobile | 32,694.639844 | 1,029,315.866848 | 3.18% |
| tablet | 4,229.719947 | 1,029,315.866848 | 0.41% |

The result can prioritise device-specific UX testing, checkout optimisation, and campaign landing-page QA.

**Observation & finding:** Desktop accounted for 96.41% of revenue, while mobile contributed only 3.18%. Desktop should remain the primary optimisation focus, but the unusually low mobile share also warrants a mobile UX and checkout audit to rule out friction or tracking gaps.

---

### Query 08 — Cross-sell products for Vintage Henley buyers

**Business question:** What other products did customers buy after purchasing **YouTube Men's Vintage Henley** in July 2017?

```sql
WITH buyer_list AS (
  SELECT DISTINCT fullVisitorId
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
    UNNEST(hits) AS hit,
    UNNEST(hit.product) AS product
  WHERE product.v2ProductName = "YouTube Men's Vintage Henley"
    AND totals.transactions >= 1
    AND product.productRevenue IS NOT NULL
)
SELECT
  product.v2ProductName AS other_purchased_products,
  SUM(product.productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
  UNNEST(hits) AS hit,
  UNNEST(hit.product) AS product
JOIN buyer_list USING (fullVisitorId)
WHERE product.v2ProductName != "YouTube Men's Vintage Henley"
  AND product.productRevenue IS NOT NULL
  AND totals.transactions >= 1
GROUP BY 1
ORDER BY quantity DESC;
```

**Expected output (top products):**

| other_purchased_products | quantity |
|---|---:|
| Google Sunglasses | 20 |
| Google Women's Vintage Hero Tee Black | 7 |
| SPF-15 Slim & Slender Lip Balm | 6 |
| Google Women's Short Sleeve Hero Tee Red Heather | 4 |

The output supports product bundles, recommendation rules, and targeted cross-sell campaigns.

**Observation & finding:** Google Sunglasses was by far the most common co-purchase (20 units), compared with only seven units for the next product. This is a clear candidate for a product bundle, cart recommendation, or targeted offer to buyers of the Vintage Henley.

---

### Query 09 — Product funnel: view → add to cart → purchase

**Business question:** For each product in January–March 2017, what proportion of product views progressed to add-to-cart and purchase?

> **Note:** The project brief's sample output is aggregated by month, while the stated requirement asks for product-level analysis. The query below follows the stated requirement and returns one row per product per month; removing `product_name` from the `SELECT` and `GROUP BY` produces the monthly roll-up shown in the sample output.

```sql
WITH product_data AS (
  SELECT
    FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
    product.v2ProductName AS product_name,
    COUNTIF(hit.eCommerceAction.action_type = '2') AS product_views,
    COUNTIF(hit.eCommerceAction.action_type = '3') AS add_to_cart,
    COUNTIF(hit.eCommerceAction.action_type = '6'
      AND product.productRevenue IS NOT NULL) AS purchases
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits) AS hit,
    UNNEST(hit.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '0101' AND '0331'
    AND hit.eCommerceAction.action_type IN ('2', '3', '6')
  GROUP BY 1, 2
)
SELECT
  *,
  ROUND(100.0 * SAFE_DIVIDE(add_to_cart, product_views), 2) AS add_to_cart_rate,
  ROUND(100.0 * SAFE_DIVIDE(purchases, product_views), 2) AS purchase_rate
FROM product_data
ORDER BY month, product_views DESC;
```

**Expected output (overall monthly funnel from the project brief):**

| month | product_views | add_to_cart | purchases | add_to_cart_rate | purchase_rate |
|---|---:|---:|---:|---:|---:|
| 201701 | 25,787 | 7,342 | 2,143 | 28.47% | 8.31% |
| 201702 | 21,489 | 7,360 | 2,060 | 34.25% | 9.59% |
| 201703 | 23,549 | 8,782 | 2,977 | 37.29% | 12.64% |

A low add-to-cart rate may indicate a product-page issue; a large drop after add-to-cart may indicate price, shipping, or checkout friction.

**Observation & finding:** Both funnel rates improved month over month: add-to-cart increased from 28.47% to 37.29%, and purchase rate increased from 8.31% to 12.64%. The March performance provides a useful benchmark; the team should identify which products, promotions, or experience changes coincided with this improvement and replicate them where appropriate.

---

### Query 10 — Weekly and cumulative revenue

**Business question:** How did revenue accumulate week by week from May to July 2017?

```sql
WITH weekly_data AS (
  SELECT
    FORMAT_DATE('%Y-%W', PARSE_DATE('%Y%m%d', date)) AS week,
    SUM(product.productRevenue) / 1000000 AS weekly_revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
    UNNEST(hits) AS hit,
    UNNEST(hit.product) AS product
  WHERE _TABLE_SUFFIX BETWEEN '0501' AND '0731'
    AND product.productRevenue IS NOT NULL
  GROUP BY 1
)
SELECT
  week,
  weekly_revenue,
  SUM(weekly_revenue) OVER (
    ORDER BY week
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_revenue
FROM weekly_data
ORDER BY week;
```

**Expected output (excerpt):**

| week | weekly_revenue | cumulative_revenue |
|---|---:|---:|
| 2017-18 | 32,588.709909 | 32,588.709909 |
| 2017-19 | 34,741.409884 | 67,330.119793 |
| 2017-20 | 27,606.739887 | 94,936.859680 |
| 2017-21 | 20,198.539885 | 115,135.399565 |
| 2017-22 | 31,789.519858 | 146,924.919423 |
| 2017-23 | 19,751.869914 | 166,676.789337 |
| 2017-24 | 45,053.909903 | 211,730.699240 |
| 2017-25 | 28,583.259913 | 240,313.959153 |
| 2017-26 | 24,813.409910 | 265,127.369063 |
| 2017-27 | 20,437.739935 | 285,565.108998 |
| 2017-28 | 41,250.799849 | 326,815.908847 |
| 2017-29 | 57,111.459805 | 383,927.368652 |
| 2017-30 | 29,493.939812 | 413,421.308464 |

The running total makes it easier to monitor progress against a revenue target and identify unusually strong or weak weeks.

**Observation & finding:** Revenue was volatile across the period, ranging from 19.8K in week 23 to 57.1K in week 29. The cumulative total reached 413.4K by week 30. The sharp week-29 peak should be traced to specific sources, promotions, or product launches so the business can determine whether it is repeatable.

---

## 🔎 Key Takeaways

- Analyse both **volume and quality**: high-traffic channels are not necessarily the best-converting channels.
- Use **bounce rate and conversion rate together** when evaluating acquisition sources.
- Treat device performance as a product and UX signal, not only a reporting breakdown.
- Use the product funnel to identify where shoppers abandon the journey, then test improvements to product pages, cart, and checkout.
- Turn product co-purchase patterns into bundles and personalised recommendations.
- Track weekly revenue alongside cumulative revenue to manage performance throughout a campaign period.

---

## 🚀 How to Run

1. Open [BigQuery Console](https://console.cloud.google.com/bigquery).
2. Create a query in **GoogleSQL / Standard SQL**.
3. Copy a query from this README and run it against the public dataset.
4. Review query cost before execution; limit the table suffix/date range whenever possible.

> **Note:** This is a public sample dataset. Results may differ slightly if Google updates the underlying sample data.

---

### Skills demonstrated

`BigQuery SQL` · `CTEs` · `UNNEST` · `CASE` · `COUNTIF` · `SAFE_DIVIDE` · `Window Functions` · `Aggregation` · `E-commerce Analytics`
