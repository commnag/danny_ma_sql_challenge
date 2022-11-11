-- A. Customer Nodes Exploration

-- 1. How many unique nodes are there on the Data Bank system?
SELECT COUNT(DISTINCT(node_id)) as unique_nodes
FROM data_bank.customer_nodes;

-- 2. What is the number of nodes per region?
SELECT region_id, COUNT(DISTINCT(node_id)) node_count
FROM data_bank.customer_nodes
GROUP BY region_id; 

-- 3. How many customers are allocated to each region?
SELECT region_id, COUNT(DISTINCT(customer_id)) as customer_count
FROM data_bank.customer_nodes
GROUP BY region_id;

-- 4. How many days on average are customers reallocated to a different node?
SELECT AVG(end_date - start_date) as average_days FROM data_bank.customer_nodes;

-- 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
SELECT region_id, PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY end_date - start_date) as median_percentile_50, PERCENTILE_CONT(0.8) WITHIN GROUP(ORDER BY end_date - start_date) as percentile_80, PERCENTILE_CONT(0.95) WITHIN GROUP(ORDER BY end_date - start_date) as percentile_95 FROM data_bank.customer_nodes
GROUP BY region_id;


-- B. Customer Transactions

-- 1. What is the unique count and total amount for each transaction type?
SELECT txn_type, COUNT(txn_amount) as unique_count, SUM(txn_amount) as total_amount 
FROM data_bank.customer_transactions
GROUP BY txn_type;

-- 2. What is the average total historical deposit counts and amounts for all customers?
SELECT COUNT(*) as deposit_counts, AVG(txn_amount) as average_amounts
FROM data_bank.customer_transactions
WHERE txn_type = 'deposit';

-- 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

-- 4. What is the closing balance for each customer at the end of the month?

-- 5. What is the percentage of customers who increase their closing balance by more than 5%?

