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
1. How do customer balances evolve over time?
→ Calculate running and closing balances for each customer.

2. How much data would be required under each allocation option?
Option 1: Data allocated based on the previous month’s closing balance.
Option 2: Data allocated based on the average balance in the last 30 days.
Option 3: Data allocated and updated in real-time.

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

**Sample Data**
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

**Sample Data**
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

**Sample Data**
| customer_id | txn_date | txn_type | txn_amount |
|--------------|-----------|-----------|-------------|
| 429 | 2020-01-21 | deposit | 82 |
| 155 | 2020-01-10 | deposit | 712 |
| 398 | 2020-01-01 | deposit | 196 |
| 255 | 2020-01-14 | deposit | 563 |
| 185 | 2020-01-29 | deposit | 626 |

</details>

## 📊 Analysis & Key Insights  

### 🔹 Step 1: Compute Running Balance  

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
