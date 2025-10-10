# 🏦 Customer Transaction Analysis for Data Allocation Strategy in a E-Banking Company using SQL  

**Author:** Hoang Thi Hong Nhung  
**Date:** 2025-09-30  
**Tools Used:** SQL (BigQuery)

---

## 📑 Table of Contents  
1. [📌 Background & Overview](#-background--overview)  
2. [📂 Dataset Description & Data Structure](#-dataset-description--data-structure)  
3. [📊 SQL Analysis & Key Insights](#-sql-analysis--key-insights)  
4. [💡 Recommendations & Final Conclusion](#-recommendations--final-conclusion)  

---

## 📌 Background & Overview  

### 🧭 Situation  
Data Bank, a next-generation Neo-Bank — a digital-only financial institution with no physical branches. Unlike traditional banks, Data Bank not only handles customer transactions but also offers secure, distributed cloud data storage linked directly to customer account balances.

### ❓ Business Question  
How much data storage should be provisioned for customers under different allocation strategies?

Specifically, the project aims to answer:
1. How much data would be required under each allocation option?

- Option 1: Data allocated based on the previous month’s closing balance.

- Option 2: Data allocated based on the average balance in the last 30 days.

- Option 3: Data allocated and updated in real-time.

3. Which option provides the best balance between accuracy and efficiency?

### 👥 Target Audience

This analysis is designed for:
- Management Team: to plan data infrastructure and forecast costs.
- Data & BI Teams: to monitor storage demand and customer trends.
- Product Team: to align system performance with customer activity.

---

## 📂 Dataset Description & Data Structure  

### 🗄️ Data Source  
- **Source:** [Data Bank Case Study – 8 Week SQL Challenge](https://8weeksqlchallenge.com/case-study-4/)  
- **Size:** ~5,000+ transactions across multiple customers and nodes  

### 📊 Data Structure & Relationships  

#### Tables Used  
The dataset contains **3 related tables** forming a relational schema:

| Table Name | Description |
|-------------|--------------|
| `regions` | Contains information on geographic regions where Data Bank operates. |
| `customer_nodes` | Tracks where each customer’s data and funds are stored, including time-based reallocation. |
| `customer_transactions` | Contains all monetary activities — deposits, withdrawals, and purchases — per customer. |

**🔗 Relationships:**  
<img width="859" height="241" alt="image" src="https://github.com/user-attachments/assets/302688c3-bea1-48e7-a7d0-4bfd41fa8bd4" />

#### Table Schema & Data Snapshot  

<details>
  <summary>🗺️ Table 1: regions</summary>

| Column Name | Data Type | Description |
|--------------|------------|-------------|
| region_id | INT | Unique identifier for each region |
| region_name | STRING | Name of the geographic region |

***Sample Data***
*
| region_id | region_name |
|------------|--------------|
| 1 | Africa |
| 2 | America |
| 3 | Asia |

</details>

<details>
  <summary>🌐 Table 2: customer_nodes</summary>

| Column Name | Data Type | Description |
|--------------|------------|-------------|
| customer_id | INT | Unique identifier for each customer |
| region_id | INT | Region assigned to the customer |
| node_id | INT | Node ID where customer’s data and funds are stored |
| start_date | DATE | Allocation start date |
| end_date | DATE | Allocation end date (until reallocation) |

***Sample Data***

| customer_id | region_id | node_id | start_date | end_date |
|--------------|------------|----------|-------------|-----------|
| 1 | 3 | 4 | 2020-01-02 | 2020-01-03 |
| 2 | 3 | 5 | 2020-01-03 | 2020-01-17 |
| 3 | 5 | 4 | 2020-01-27 | 2020-02-18 |
| 4 | 5 | 4 | 2020-01-07 | 2020-01-19 |
| 5 | 3 | 3 | 2020-01-15 | 2020-01-23 |

</details>

<details>
  <summary>💳 Table 3: customer_transactions</summary>

| Column Name | Data Type | Description |
|--------------|------------|-------------|
| customer_id | INT | Unique identifier for each customer |
| txn_date | DATE | Date of transaction |
| txn_type | STRING | Type of transaction (`deposit`, `withdrawal`, or `purchase`) |
| txn_amount | FLOAT | Transaction amount |

***Sample Data***

| customer_id | txn_date | txn_type | txn_amount |
|--------------|-----------|-----------|-------------|
| 429 | 2020-01-21 | deposit | 82 |
| 155 | 2020-01-10 | deposit | 712 |
| 398 | 2020-01-01 | deposit | 196 |
| 255 | 2020-01-14 | deposit | 563 |
| 185 | 2020-01-29 | deposit | 626 |

</details>

## 📊 Analysis & Key Insights  

### A. Data Exploration

#### 🔹 Question 1: How many customers are allocated to each region?
  
**Query:**
```sql
WITH netamount AS (
  SELECT customer_id,
         txn_date,
         CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END AS net_amount
  FROM `data_bank.customer_transactions`
),
running_balance AS (
  SELECT customer_id,
         txn_date,
         SUM(net_amount) OVER (PARTITION BY customer_id ORDER BY txn_date) AS balance
  FROM netamount
)
SELECT * FROM running_balance;
```

**Result:**

| Region | Customer Count |
|--------|----------------|
| 🌏 Australia | 110 |
| 🌎 America | 105 |
| 🌍 Africa | 102 |
| 🌏 Asia | 95 |
| 🌍 Europe | 88 |

**Insight:**  Australia and America have the highest customer bases, while Europe has the lowest — suggesting higher data capacity may be needed in Oceania and America regions.

---
 #### 🔹 Question 2: Average Days Until Customer Reallocation?
 **Query:**
```sql
with customer_moves AS (
  SELECT
    customer_id,region_id,
    count(node_id) as node_count_1
  FROM `alpine-biplane-472213-d8.data_bank.customer_nodes`
  group by customer_id,region_id
  having count(node_id)>=2
)
SELECT 
  AVG(DATE_DIFF(end_date, start_date, DAY)) AS avg_days_until_reallocate
FROM `alpine-biplane-472213-d8.data_bank.customer_nodes` a 
inner join customer_moves b on a.customer_id = b.customer_id
WHERE EXTRACT(YEAR FROM end_date) != 9999;
```

**Result:**  
| avg_days_until_reallocate |
|--------|
| 14.6 | 

**Insight:**  
 On average, customers are moved to a different node every **~15 days**, reflecting a proactive data security measure to minimize risks of data breaches and ensure system resilience.

---
#### 🔹 Question 3: Median, 80th, and 95th Percentile of Reallocation Days by Region

**Query:**
```sql
WITH customer_moves AS (
  SELECT
    customer_id,
    region_id,
    COUNT(node_id)
  FROM `alpine-biplane-472213-d8.data_bank.customer_nodes`
  GROUP BY customer_id, region_id
  HAVING COUNT(node_id) >= 2
)
SELECT
  region_name,
  APPROX_QUANTILES(days_until_reallocate, 100)[OFFSET(50)] AS median_days,
  APPROX_QUANTILES(days_until_reallocate, 100)[OFFSET(80)] AS p80_days,
  APPROX_QUANTILES(days_until_reallocate, 100)[OFFSET(95)] AS p95_days
FROM (
  SELECT 
    a.customer_id,
    b.region_id,
    c.region_name,
    DATE_DIFF(end_date, start_date, DAY) AS days_until_reallocate
  FROM `alpine-biplane-472213-d8.data_bank.customer_nodes` a
  INNER JOIN customer_moves b 
    ON a.customer_id = b.customer_id AND a.region_id = b.region_id
  INNER JOIN `alpine-biplane-472213-d8.data_bank.regions` c 
    ON a.region_id = c.region_id
  WHERE EXTRACT(YEAR FROM end_date) != 9999
) a
GROUP BY region_name;
```

**Result**
| Region      | Median Days | P80 | P95 |
|--------------|-------------|-----|-----|
| Australia    | 15          | 23  | 28  |
| America      | 15          | 23  | 28  |
| Africa       | 15          | 24  | 28  |
| Asia         | 15          | 23  | 28  |
| Europe       | 15          | 24  | 28  |

**Insight:**  
- Median reallocation time is ~15 days across all regions.

- 80th–95th percentile ranges between 23–28 days.

- Suggests consistent reallocation cycle and stable customer movement pattern globally.

---
#### 💰 Question 4: Monthly Active Customers with Multiple Deposits and at Least One Purchase or Withdrawal

**Query:**
```sql
WITH customer_qualified AS (
  SELECT 
    customer_id,
    EXTRACT(YEAR FROM txn_date) AS year,
    EXTRACT(MONTH FROM txn_date) AS month,
    SUM(CASE WHEN LOWER(txn_type) = 'deposit' THEN 1 ELSE 0 END) AS deposit_count,
    SUM(CASE WHEN LOWER(txn_type) IN ('purchase', 'withdrawal') THEN 1 ELSE 0 END) AS purchase_withdrawal_count
  FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`
  GROUP BY customer_id, EXTRACT(YEAR FROM txn_date), EXTRACT(MONTH FROM txn_date)
)
SELECT 
  year, 
  month, 
  COUNT(customer_id) AS customer_count
FROM customer_qualified
WHERE deposit_count >= 1 
  AND purchase_withdrawal_count >= 1
GROUP BY year, month;
```

**Result**
| Year | Month | Customer Count |
|------|--------|----------------|
| 2020 | 1      | 294            |
| 2020 | 2      | 300            |
| 2020 | 3      | 330            |
| 2020 | 4      | 137            |

**Insight**
- Customer engagement peaked in March 2020 with 330 active users.  

- Significant drop in April (137), suggesting possible external or seasonal factors.  

- Consistent upward trend from January to March indicates growing transaction activity.

---

### B. Data Allocation Analysis

#### Option 1: Data allocated based on previous month’s closing balance

```sql
WITH month_list AS (
  SELECT FORMAT_DATE('%Y-%m', month) AS year_month
  FROM UNNEST(
    GENERATE_DATE_ARRAY(
      (SELECT MIN(DATE(txn_date)) FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`),
      (SELECT MAX(DATE(txn_date)) FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`),
      INTERVAL 1 MONTH
    )
  ) AS month
),
transactionvalue AS (
  SELECT 
    customer_id,
    FORMAT_DATE('%Y-%m', DATE(txn_date)) AS year_month,
    SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS transaction_value
  FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`
  GROUP BY customer_id, year_month
),
all_months AS (
  SELECT c.customer_id, m.year_month
  FROM (SELECT DISTINCT customer_id FROM transactionvalue) c
  CROSS JOIN month_list m
),
filled_months AS (
  SELECT a.customer_id, a.year_month, COALESCE(t.transaction_value, 0) AS transaction_value
  FROM all_months a
  LEFT JOIN transactionvalue t
    ON a.customer_id = t.customer_id AND a.year_month = t.year_month
),
closingbalance AS (
  SELECT 
    customer_id,
    year_month,
    SUM(transaction_value) OVER (PARTITION BY customer_id ORDER BY year_month ASC) AS closing_balance
  FROM filled_months
),
with_prev AS (
  SELECT 
    customer_id,
    year_month,
    closing_balance,
    LAG(closing_balance) OVER (PARTITION BY customer_id ORDER BY year_month) AS prev_closing_balance
  FROM closingbalance
),
opt1 AS (
  SELECT 
    year_month,
    SUM(CASE WHEN prev_closing_balance > 0 THEN prev_closing_balance ELSE 0 END) AS total_data_required_option1
  FROM with_prev
  GROUP BY year_month
)
SELECT 
  AVG(total_data_required_option1) AS avg_data_required_option1
FROM opt1
WHERE year_month != '2020-01';
```

**Result**
| Option | Average Data Required |
|---------|------------------------|
| 1       | 254.146               |

** Visualization**

<img width="545" height="374" alt="Image" src="https://github.com/user-attachments/assets/33d2d46e-2568-4216-a546-48a56db8dec8" />

 
 **Insight:** Option 1 requires ~254.1 units of data on average — efficient and stable since it relies on prior month balances.

---

#### Option 2: Data allocated based on 30-day rolling average balance
**Query:**
```sql
WITH day_list AS (
  SELECT DATE(day) AS txn_day
  FROM UNNEST(
    GENERATE_DATE_ARRAY(
      (SELECT MIN(DATE(txn_date)) FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`),
      (SELECT MAX(DATE(txn_date)) FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`),
      INTERVAL 1 DAY
    )
  ) AS day
),
daily_txn AS (
  SELECT 
    customer_id,
    DATE(txn_date) AS txn_day,
    SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS net_amount
  FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`
  GROUP BY customer_id, txn_day
),
all_days AS (
  SELECT c.customer_id, d.txn_day
  FROM (SELECT DISTINCT customer_id FROM daily_txn) c
  CROSS JOIN day_list d
),
filled_days AS (
  SELECT a.customer_id, a.txn_day, COALESCE(t.net_amount, 0) AS net_amount
  FROM all_days a
  LEFT JOIN daily_txn t
    ON a.customer_id = t.customer_id AND a.txn_day = t.txn_day
),
running_balance AS (
  SELECT 
    customer_id,
    txn_day,
    SUM(net_amount) OVER (PARTITION BY customer_id ORDER BY txn_day) AS balance
  FROM filled_days
),
avg_balance_30days AS (
  SELECT 
    customer_id,
    txn_day,
    AVG(balance) OVER (
      PARTITION BY customer_id
      ORDER BY txn_day
      ROWS BETWEEN 30 PRECEDING AND CURRENT ROW
    ) AS avg_balance_30d
  FROM running_balance
),
daily_data_need AS (
  SELECT txn_day, SUM(CASE WHEN avg_balance_30d > 0 THEN avg_balance_30d ELSE 0 END) AS total_data_required
  FROM avg_balance_30days
  GROUP BY txn_day
),
max_data_per_month AS (
  SELECT 
    EXTRACT(YEAR FROM txn_day) AS year,
    EXTRACT(MONTH FROM txn_day) AS month,
    txn_day AS peak_day,
    total_data_required,
    RANK() OVER (PARTITION BY EXTRACT(YEAR FROM txn_day), EXTRACT(MONTH FROM txn_day)
                 ORDER BY total_data_required DESC) AS rnk
  FROM daily_data_need
),
monthly_peak AS (
  SELECT year, month, peak_day, total_data_required AS max_data_required_in_month
  FROM max_data_per_month
  WHERE rnk = 1
)
SELECT AVG(max_data_required_in_month)
FROM monthly_peak
WHERE NOT (year = 2020 AND month = 1);
```

**Result**
| Option | Average Data Required |
|---------|------------------------|
| 2       | 254.293               |

**Visualization**

<img width="538" height="374" alt="Image" src="https://github.com/user-attachments/assets/0be2d0b2-4f28-4562-ad4f-e2a52a3c4714" />

#### Option 3 Data allocated based on realtime average balance

```sql
-- Create a list of all calendar days in the dataset
WITH day_list AS (
  SELECT 
    DATE(day) AS txn_day
  FROM UNNEST(
    GENERATE_DATE_ARRAY(
      (SELECT MIN(DATE(txn_date)) FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`),
      (SELECT MAX(DATE(txn_date)) FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`),
      INTERVAL 1 DAY
    )
  ) AS day
),

-- Aggregate daily transactions per customer
daily_txn AS (
  SELECT 
    customer_id,
    DATE(txn_date) AS txn_day,
    SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS net_amount
  FROM `alpine-biplane-472213-d8.data_bank.customer_transactions`
  GROUP BY customer_id, txn_day
),

-- Ensure every customer has all days
all_days AS (
  SELECT 
    c.customer_id,
    d.txn_day
  FROM (SELECT DISTINCT customer_id FROM daily_txn) c
  CROSS JOIN day_list d
),

-- Fill missing transactions with 0
filled_days AS (
  SELECT 
    a.customer_id,
    a.txn_day,
    COALESCE(t.net_amount, 0) AS net_amount
  FROM all_days a
  LEFT JOIN daily_txn t
    ON a.customer_id = t.customer_id
   AND a.txn_day = t.txn_day
),

-- Compute cumulative running balance (including no-txn days)
running_balance AS (
  SELECT 
    customer_id,
    txn_day,
    SUM(net_amount) OVER (
      PARTITION BY customer_id
      ORDER BY txn_day
    ) AS balance
  FROM filled_days
),

-- Compute daily total data requirement (sum of all positive balances)
daily_data AS (
  SELECT 
    txn_day,
    SUM(CASE WHEN balance > 0 THEN balance ELSE 0 END) AS total_data
  FROM running_balance
  GROUP BY txn_day
)

-- Find peak (max) data per month
monthly_max AS (
  SELECT 
    EXTRACT(YEAR FROM txn_day) AS year,
    EXTRACT(MONTH FROM txn_day) AS month,
    MAX(total_data) AS total_data_required_option3
  FROM daily_data
  GROUP BY year, month
),

-- Average required data (excluding first month)
SELECT 
  AVG(total_data_required_option3) AS avg_data_required_option3
FROM monthly_max
WHERE NOT (year = 2020 AND month = 1)
```
**Result:**
| Option | Average Data Required |
|---------|------------------------|
| 3       | 273.816               |


**Visualization:**

<img width="545" height="374" alt="Image" src="https://github.com/user-attachments/assets/027896b6-9f68-43db-b542-a3ea5bdea5d5" />

**Insight**
Real-time allocation (Option 3) consumes more data (~273.8), about 8% higher than periodic methods, offering the most responsiveness at a higher storage cost.


## 💡 Recommendations & Final Conclusion

#### ✅ Conclusion — Allocation Model Comparison

| **Option** | **Description** | **Data require** | **Data Volume Impact** | **Complexity** | **Strengths** | **Weaknesses** |
|-------------|------------------|------------------|------------------------|----------------|--------------|-------------|
| 🟩 **Option 1** | Data allocated based on **previous month’s closing balance** | `254.146` | ✅ *Low* — monthly snapshot only | ⭐ Simple | • Simple and stable — data only updated monthly.<br>• Lowest computational cost and easiest to maintain.<br>• Ideal for monthly planning and budget allocation. |• Not responsive to within-month fluctuations.<br>• Can be manipulated — users might top up large amounts near month-end to inflate their closing balance. |
| 🟨 **Option 2** | Data allocated based on **average balance in the last 30 days** | `254.293` | ⚖️ *Moderate* — rolling 30-day calculation | ⭐⭐⭐ High | • Reflects recent performance trends.<br>• Balances accuracy and stability better than monthly snapshots.<br>• Useful for rolling forecasts and near-term planning. | • Requires daily recalculation of 30-day rolling average → heavy compute load.<br>• Complex to implement and maintain.<br>• Slower query performance for large datasets. |
| 🟥 **Option 3** | Data allocated and updated **in real-time** | `273.816` | 🔺 *High* — continuous updates from transactions | ⭐⭐ Medium | • Highest accuracy and responsiveness.<br>• Best suited for live dashboards and operational monitoring.<br>• Captures immediate changes in balances. | • Very high data volume and storage demand.<br>• Continuous data updates increase infrastructure costs.<br>• Overkill for periodic or strategic analysis. |

#### ✅ Recommendation
- **Option 1** offers the **best balance between accuracy, simplicity, and cost efficiency**, making it ideal for monthly operational reporting.  
If higher granularity is required (e.g., weekly financial reviews).

- **Option 2** could be considered — but it requires heavier processing power.














