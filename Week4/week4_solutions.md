**Schema (PostgreSQL v13)**

---
<p align=center><b>A. Customer Nodes Exploration</b>

---

**Query #1** How many unique nodes are there on the Data Bank system?

    SELECT COUNT(DISTINCT(node_id)) as unique_nodes
    FROM data_bank.customer_nodes;

| unique_nodes |
| ------------ |
| 5            |

---
**Query #2** What is the number of nodes per region?

    SELECT region_id, COUNT(DISTINCT(node_id)) node_count
    FROM data_bank.customer_nodes
    GROUP BY region_id;

| region_id | node_count |
| --------- | ---------- |
| 1         | 5          |
| 2         | 5          |
| 3         | 5          |
| 4         | 5          |
| 5         | 5          |

---
**Query #3** How many customers are allocated to each region?

    SELECT region_id, COUNT(DISTINCT(customer_id)) as customer_count
    FROM data_bank.customer_nodes
    GROUP BY region_id;

| region_id | customer_count |
| --------- | -------------- |
| 1         | 110            |
| 2         | 105            |
| 3         | 102            |
| 4         | 95             |
| 5         | 88             |

---
**Query #4** How many days on average are customers reallocated to a different node?

    SELECT AVG(end_date - start_date) as average_days FROM data_bank.customer_nodes;

| average_days        |
| ------------------- |
| 416373.411714285714 |

---
**Query #5** What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

    SELECT region_id, PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY end_date - start_date) as median_percentile_50, PERCENTILE_CONT(0.8) WITHIN GROUP(ORDER BY end_date - start_date) as percentile_80, PERCENTILE_CONT(0.95) WITHIN GROUP(ORDER BY end_date - start_date) as percentile_95 FROM data_bank.customer_nodes
    GROUP BY region_id;

| region_id | median_percentile_50 | percentile_80 | percentile_95 |
| --------- | -------------------- | ------------- | ------------- |
| 1         | 17                   | 28            | 2914533.55    |
| 2         | 18                   | 27            | 2914534.3     |
| 3         | 17.5                 | 27            | 2914535.35    |
| 4         | 17                   | 27            | 2914538       |
| 5         | 18                   | 28            | 2914527       |

---
<p align=center><b>B. Customer Transactions</b>

---
**Query #1** What is the unique count and total amount for each transaction type?

    SELECT txn_type, COUNT(txn_amount) as unique_count, SUM(txn_amount) as total_amount 
    FROM data_bank.customer_transactions
    GROUP BY txn_type;

| txn_type   | unique_count | total_amount |
| ---------- | ------------ | ------------ |
| purchase   | 1617         | 806537       |
| deposit    | 2671         | 1359168      |
| withdrawal | 1580         | 793003       |

---
**Query #7** What is the average total historical deposit counts and amounts for all customers?

    SELECT COUNT(*) as deposit_counts, AVG(txn_amount) as average_amounts
    FROM data_bank.customer_transactions
    WHERE txn_type = 'deposit';

| deposit_counts | average_amounts      |
| -------------- | -------------------- |
| 2671           | 508.8611007113440659 |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2GtQz4wZtuNNu7zXH5HtV4/3)
