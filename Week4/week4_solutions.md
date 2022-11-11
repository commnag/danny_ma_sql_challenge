**Schema (PostgreSQL v13)**


    

---

**Query #1**
1. What is the unique count and total amount for each transaction type?

    SELECT txn_type, COUNT(txn_amount) as unique_count, SUM(txn_amount) as total_amount 
    FROM data_bank.customer_transactions
    GROUP BY txn_type;

| txn_type   | unique_count | total_amount |
| ---------- | ------------ | ------------ |
| purchase   | 1617         | 806537       |
| deposit    | 2671         | 1359168      |
| withdrawal | 1580         | 793003       |

---
**Query #2**
2. What is the average total historical deposit counts and amounts for all customers?

    SELECT COUNT(*) as deposit_counts, AVG(txn_amount) as average_amounts
    FROM data_bank.customer_transactions
    WHERE txn_type = 'deposit';

| deposit_counts | average_amounts      |
| -------------- | -------------------- |
| 2671           | 508.8611007113440659 |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2GtQz4wZtuNNu7zXH5HtV4/3)
