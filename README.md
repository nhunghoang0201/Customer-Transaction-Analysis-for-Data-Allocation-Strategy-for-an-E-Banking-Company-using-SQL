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
**Data Bank** is a next-generation **Neo-Bank**, operating entirely digitally and linking customer balances with **cloud data storage** capacity. The bank wants to forecast total storage demand across customers to optimize infrastructure and reduce costs.  

### ⚙️ Problem  
Different data allocation can significantly impact both **storage cost** and **customer satisfaction**.  
The management team needs to understand **how much data storage would be required** under three different allocation strategies.  
| Option | Allocation Basis | Description |
|---------|------------------|--------------|
| **Option 1** | Previous month’s closing balance | Simple, fixed allocation once per month |
| **Option 2** | 30-day rolling average balance | Smooth adjustment based on past 30-day average |
| **Option 3** | Real-time balance | Instant data update after every transaction |

### ❓ Key Business Question  
> How much data would have been required for each allocation option on a monthly basis?  

---

## 📂 Dataset Description & Data Structure  

**Database:** `data_bank`  

| Table | Description | Key Fields |
|--------|--------------|-------------|
| **regions** | List of bank operating regions | `region_id`, `region_name` |
| **customer_nodes** | Tracks where each customer’s data and funds are stored | `customer_id`, `region_id`, `node_id`, `start_date`, `end_date` |
| **customer_transactions** | Records deposits, withdrawals, and purchases | `customer_id`, `txn_date`, `txn_type`, `txn_amount` |

<details>
  <summary>📸 Example Data Snapshot</summary>

**customer_transactions**
| customer_id | txn_date   | txn_type | txn_amount |
|--------------|------------|----------|-------------|
| 429 | 2020-01-21 | deposit | 82 |
| 155 | 2020-01-10 | deposit | 712 |
| 398 | 2020-01-01 | deposit | 196 |
| 185 | 2020-01-29 | deposit | 626 |

</details>

---

## 📊 SQL Analysis & Key Insights  

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
