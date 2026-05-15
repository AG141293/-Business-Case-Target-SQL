# 🎯 Target Brazil — E-Commerce SQL Analysis (Google BigQuery)

> Analyzed **100,000+ Target Brazil e-commerce orders (2016–2018)** using Google BigQuery to uncover sales trends, delivery performance, freight costs, payment behavior, and customer distribution — driving data-backed business decisions.

---

## 📌 Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Database Setup](#database-setup)
- [Dataset Schema](#dataset-schema)
- [Analysis Breakdown](#analysis-breakdown)
  - [1. Initial Exploration](#1-initial-exploration)
  - [2. In-depth Exploration](#2-in-depth-exploration)
  - [3. Evolution of E-commerce Orders](#3-evolution-of-e-commerce-orders)
  - [4. Impact on Economy](#4-impact-on-economy)
  - [5. Sales, Freight & Delivery Time](#5-sales-freight--delivery-time)
  - [6. Payment Type Analysis](#6-payment-type-analysis)
- [Key Insights](#key-insights)

---

## 📖 Project Overview

This project performs end-to-end SQL analysis on Target's Brazil e-commerce dataset hosted on **Google BigQuery**. The dataset covers customer orders placed between **September 2016 and October 2018** and spans 8 relational tables including customers, orders, payments, products, sellers, reviews, and geolocation data.

The goal is to answer critical business questions around:
- E-commerce growth trends and seasonality in Brazil
- Customer purchase behavior (time of day, location)
- Delivery performance vs. estimated dates
- Freight cost distribution by state
- Payment type and installment preferences

---

## 🛠 Tech Stack

| Tool | Purpose |
|---|---|
| **Google BigQuery** | Cloud data warehouse and SQL execution |
| **Google Cloud Console** | Project and dataset management |
| **Standard SQL** | Querying, aggregation, joins, date functions |

---

## 🗄 Database Setup

- **Project ID:** `ankita-project-01`
- **Dataset ID:** `eCommerce_Requirements`
- **Region:** `asia-south1` (Mumbai)
- **Total Tables:** 8

### Steps to Recreate
1. Create a new project on [Google Cloud Console](https://console.cloud.google.com/)
2. Navigate to BigQuery and create a dataset named `eCommerce_Requirements`
3. Upload each CSV file as a separate table (Native table format)
4. Run the queries from each section below

---

## 📂 Dataset Schema

| Table | Description | Sample Size |
|---|---|---|
| `customers` | Customer ID, city, state, zip code | 10 records shown |
| `geolocation` | Zip code, latitude, longitude, city, state | 10 records shown |
| `order_items` | Order ID, product ID, seller ID, shipping limit date, price, freight | 10 records shown |
| `order_review` | Review ID, order ID, score, comment title, creation date | 10 records shown |
| `orders` | Order ID, customer ID, status, purchase/delivery timestamps | 10 records shown |
| `payments` | Order ID, payment sequence, type, installments, value | 10 records shown |
| `products` | Product ID, category, name, description, photos, weight, dimensions | 10 records shown |
| `sellers` | Seller ID, zip code, city, state | 10 records shown |

### Customers Table — Column Data Types
| Column | Data Type |
|---|---|
| customer_id | STRING |
| customer_unique_id | STRING |
| customer_zip_code_prefix | INT64 |
| customer_city | STRING |
| customer_state | STRING |

---

## 🔍 Analysis Breakdown

### 1. Initial Exploration

**1.1 Data Types of Columns**
```sql
SELECT table_name, column_name, data_type
FROM eCommerce_Requirements.INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'customers'
LIMIT 5;
```

**1.2 Time Period of the Data**
```sql
SELECT
  MIN(order_purchase_timestamp) AS first_order,
  MAX(order_purchase_timestamp) AS last_order
FROM `eCommerce_Requirements.orders`;
```
> **Result:** Data spans from `2016-09-04` to `2018-10-17`

**1.3 Cities and States of Customers Who Ordered**
```sql
SELECT DISTINCT
  cust.customer_city AS city,
  cust.customer_state AS state
FROM eCommerce_Requirements.customers AS cust
LEFT JOIN `eCommerce_Requirements.orders` AS o
  ON cust.customer_id = o.customer_id;
```

---

### 2. In-depth Exploration

**2.1 Growing Trend in E-commerce — Monthly Order Volume**
```sql
SELECT
  EXTRACT(YEAR FROM order_purchase_timestamp) AS year_,
  EXTRACT(MONTH FROM order_purchase_timestamp) AS month_,
  COUNT(DISTINCT order_id) AS volume
FROM `eCommerce_Requirements.orders`
WHERE order_status = 'delivered'
GROUP BY year_, month_
ORDER BY year_, month_;
```
> Orders grew from 1 in Sept 2016 to 3,546 by May 2017, showing strong upward momentum.

**2.2 Time of Day When Customers Buy**

Customers are classified into purchase windows: **Dawn (0–6), Morning (7–12), Afternoon (13–18), Night (19–23)** based on `order_purchase_timestamp`.

---

### 3. Evolution of E-commerce Orders

**3.1 Month-on-Month Orders by State**
```sql
SELECT X.MONTH, X.Year, X.Order_count, C.customer_state
FROM `eCommerce_Requirements.customers` C
JOIN (
  SELECT customer_id,
    EXTRACT(MONTH FROM order_purchase_timestamp) AS Month,
    EXTRACT(YEAR FROM order_purchase_timestamp) AS Year,
    COUNT(order_id) AS Order_Count
  FROM `eCommerce_Requirements.orders`
  GROUP BY Year, Month, customer_id
  ORDER BY Year, Month
) X ON X.customer_id = C.customer_id
GROUP BY X.Year, X.Month, X.Order_count, customer_state
LIMIT 1000;
```

**3.2 Customer Distribution Across Brazilian States**
```sql
SELECT customer_state, COUNT(DISTINCT customer_id)
FROM `eCommerce_Requirements.customers`
GROUP BY customer_state
ORDER BY customer_state;
```

---

### 4. Impact on Economy

**4.1 % Increase in Order Cost from 2017 to 2018 (Jan–Aug)**

Uses `payment_value` from the payments table, filtered to Jan–Aug for both years, then computes year-over-year percentage change.

**4.2 Mean & Sum of Price and Freight by Customer State**
```sql
SELECT
  cust.customer_state,
  ROUND(SUM(oi.price), 2)                        AS sum_prices,
  ROUND(AVG(oi.price), 2)                        AS avg_prices,
  ROUND(SUM(oi.freight_value), 2)                AS sum_freight,
  ROUND(AVG(oi.freight_value), 2)                AS avg_freight,
  ROUND(SUM(oi.price + oi.freight_value), 2)     AS price_charges,
  ROUND(AVG(oi.price + oi.freight_value), 2)     AS avg_price_charges
FROM `eCommerce_Requirements.customers` AS cust
JOIN `eCommerce_Requirements.orders` AS o
  ON cust.customer_id = o.customer_id
JOIN `eCommerce_Requirements.order_items` AS oi
  ON o.order_id = oi.order_id
GROUP BY cust.customer_state
ORDER BY cust.customer_state;
```

---

### 5. Sales, Freight & Delivery Time

**5.1 Days Between Purchasing, Delivering & Estimated Delivery**
```sql
SELECT
  DATE_DIFF(MAX(DATE(order_purchase_timestamp)),
            MIN(DATE(order_purchase_timestamp)), DAY)        AS purchasing,
  DATE_DIFF(MAX(DATE(order_delivered_customer_date)),
            MIN(DATE(order_delivered_customer_date)), DAY)   AS delivering,
  DATE_DIFF(MAX(DATE(order_estimated_delivery_date)),
            MIN(DATE(order_estimated_delivery_date)), DAY)   AS estimated_del
FROM `eCommerce_Requirements.orders`;
```
> **Result:** purchasing = 773 days | delivering = 736 days | estimated_del = 773 days

**5.2 Time to Delivery & Difference from Estimated**
```sql
SELECT
  DATE_DIFF(order_purchase_timestamp,
            order_delivered_customer_date, DAY)        AS time_to_delivery,
  DATE_DIFF(order_estimated_delivery_date,
            order_delivered_customer_date, DAY)        AS diff_estimated_delivery
FROM `eCommerce_Requirements.orders`;
```

**5.3 Average Freight, Delivery Time & Estimation Gap by State**
```sql
SELECT
  COUNT(e1.order_id)                                                          AS count_orders,
  AVG(freight_value)                                                          AS avg_freight,
  AVG(DATE_DIFF(order_purchase_timestamp,
                order_delivered_customer_date, DAY))                          AS time_to_delivery,
  AVG(DATE_DIFF(order_estimated_delivery_date,
                order_delivered_customer_date, DAY))                          AS diff_estimated_delivery
FROM `eCommerce_Requirements.orders` AS e1
JOIN `eCommerce_Requirements.order_items` AS e2 ON e1.order_id = e2.order_id
JOIN `eCommerce_Requirements.customers` AS c   ON e1.customer_id = c.customer_id
WHERE order_delivered_customer_date IS NOT NULL
GROUP BY c.customer_state;
```

**5.4 Top 5 States — Highest Average Freight (DESC)**
```sql
SELECT MAX(oi.freight_value) AS Freight_max, c.customer_state
FROM `eCommerce_Requirements.orders` AS o
INNER JOIN `eCommerce_Requirements.order_items` AS oi ON o.order_id = oi.order_id
INNER JOIN `eCommerce_Requirements.customers` AS c   ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY c.customer_state DESC
LIMIT 5;
```
> Top 5 states (DESC): TO (293.27), SP (339.59), SE (158.38), SC (375.28), RS (254.55)

**5.5 Top 5 States — Average Freight Value**
```sql
SELECT AVG(oi.freight_value) AS freight_avg, c.customer_state
FROM `eCommerce_Requirements.orders` AS o
INNER JOIN `eCommerce_Requirements.order_items` AS oi ON o.order_id = oi.order_id
INNER JOIN `eCommerce_Requirements.customers` AS c   ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY c.customer_state DESC
LIMIT 5;
```
> Results: TO (37.25), SP (15.15), SE (36.65), SC (21.47), RS (21.74)

**5.6 Top 5 States — Fastest/Slowest Delivery vs Estimate**
```sql
SELECT MIN(oi.freight_value) AS freight_min, c.customer_state
FROM `eCommerce_Requirements.orders` AS o
INNER JOIN `eCommerce_Requirements.order_items` AS oi ON o.order_id = oi.order_id
INNER JOIN `eCommerce_Requirements.customers` AS c   ON o.customer_id = c.customer_id
WHERE oi.freight_value > 0
GROUP BY c.customer_state
ORDER BY c.customer_state DESC
LIMIT 5;
```
> Results: TO (3.86), SP (0.01), SE (5.61), SC (0.04), RS (0.11)

---

### 6. Payment Type Analysis

**6.1 Month-over-Month Orders by Payment Type**
```sql
SELECT
  EXTRACT(MONTH FROM ord.order_purchase_timestamp) AS month,
  EXTRACT(YEAR  FROM ord.order_purchase_timestamp) AS year,
  pay.payment_type,
  COUNT(DISTINCT ord.order_id)                     AS total_orders
FROM `eCommerce_Requirements.orders`   AS ord
JOIN `eCommerce_Requirements.payments` AS pay ON ord.order_id = pay.order_id
GROUP BY pay.payment_type, month, year
ORDER BY year ASC, month ASC;
```
> Credit card dominates from Oct 2016 onwards; UPI and voucher usage also tracked monthly.

**6.2 Order Count by Number of Payment Installments**
```sql
SELECT
  pay.payment_installments AS total_installments,
  COUNT(DISTINCT ord.order_id) AS total_orders
FROM `eCommerce_Requirements.orders`   AS ord
JOIN `eCommerce_Requirements.payments` AS pay ON ord.order_id = pay.order_id
GROUP BY total_installments
ORDER BY pay.payment_installments;
```

| Installments | Orders |
|---|---|
| 0 | 2 |
| 1 | 49,060 |
| 2 | 12,389 |
| 3 | 10,443 |
| 4 | 7,088 |
| 5 | 5,234 |

> The majority of customers (49,060) opt for single-installment payments.

---

## 💡 Key Insights

- **E-commerce growth:** Orders surged from a single order in Sept 2016 to 3,546 in May 2017, confirming rapid e-commerce adoption in Brazil.
- **Seasonality:** Monthly volume peaked mid-year (May–August), suggesting seasonal buying patterns.
- **Delivery performance:** Average `time_to_delivery` is negative (purchase → delivery gap), and `diff_estimated_delivery` is positive — meaning most orders arrive ahead of the estimated date.
- **Freight costs:** States like **SC** and **SP** have the highest maximum freight values, while remote states show higher average freight due to distance.
- **Payment preference:** Over **49,000 orders** were paid in a single installment; credit card is the dominant payment method.
- **Customer concentration:** Most customers are concentrated in a few states (SP, MG, RJ), reflecting Brazil's urban economic geography.

---

## 📁 Project Structure

```
Target-SQL/
│
├── queries/
│   ├── 1_initial_exploration.sql
│   ├── 2_indepth_exploration.sql
│   ├── 3_ecommerce_evolution.sql
│   ├── 4_economy_impact.sql
│   ├── 5_delivery_analysis.sql
│   └── 6_payment_analysis.sql
│
├── data/                         # CSV files uploaded to BigQuery
│   ├── customers.csv
│   ├── geolocation.csv
│   ├── order_items.csv
│   ├── order_review.csv
│   ├── orders.csv
│   ├── payments.csv
│   ├── products.csv
│   └── sellers.csv
│
└── README.md
```

---

*Built on Google BigQuery as part of an e-commerce SQL analytics case study on Target Brazil's order ecosystem (2016–2018).*
