
# Case Study 4 – SQL Query Explanations

# Part A
## 1. **How many unique nodes are there on the Data Bank system?**
```sql
SELECT COUNT(DISTINCT node_id) AS unique_nodes
FROM data_bank.customer_nodes;
```
**Explanation**:
- Count of unique node IDs using `COUNT(DISTINCT node_id)`.


## 2. **What is the number of nodes per region?**
```sql
SELECT region_id, COUNT(DISTINCT node_id) AS unique_nodes
FROM data_bank.customer_nodes
GROUP BY customer_nodes.region_id;
```
**Explanation**:
- Grouped by region, counts unique node IDs for each.


## 3. **How many customers are allocated to each region?**
```sql
SELECT region_id, COUNT(customer_id) AS customer_count
FROM data_bank.customer_nodes
GROUP BY customer_nodes.region_id
ORDER BY customer_nodes.region_id;
```
**Explanation**:
- Groups by region, counts customers, orders by region ID.


## 4. **How many days on average are customers reallocated to a different node?**
```sql
WITH node_transitions AS (
  SELECT
    customer_id,
    start_date,
    LEAD(start_date) OVER (
      PARTITION BY customer_id 
      ORDER BY start_date
    ) AS next_start_date
  FROM data_bank.customer_nodes
)
SELECT
  ROUND(AVG(next_start_date - start_date)) AS avg_days_between_reallocations
FROM node_transitions
WHERE next_start_date IS NOT NULL;
```
**Explanation**:


# Part B
## 1. **What is the unique count and total amount for each transaction type?**
```sql
SELECT txn_type, COUNT(customer_id), SUM(txn_amount)
FROM data_bank.customer_transactions
GROUP BY txn_type;
```
**Explanation**:



## 2. **Average total historical deposit counts and amounts for all customers**
```sql
WITH deposit_summary AS (
  SELECT
    customer_id,
    COUNT(*) AS deposit_count,
    SUM(txn_amount) AS deposit_amount
  FROM customer_transactions
  WHERE txn_type = 'deposit'
  GROUP BY customer_id
)
SELECT
  ROUND(AVG(deposit_count), 2) AS avg_deposit_count,
  ROUND(AVG(deposit_amount), 2) AS avg_deposit_amount
FROM deposit_summary;
```
**Explanation**:


## 3. **For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**
```sql
```
**Explanation**:


## 4. **Closing balance for each customer at the end of the month**
```sql
WITH signed_txns AS (
  SELECT
    customer_id,
    DATE_TRUNC('month', txn_date)::DATE AS date_month,
    CASE
      WHEN txn_type = 'deposit' THEN txn_amount
      WHEN txn_type IN ('withdraw', 'purchase') THEN -txn_amount
      ELSE 0
    END AS signed_amount
  FROM data_bank.customer_transactions
),
monthly_sum AS (
  SELECT
    customer_id,
    date_month,
    SUM(signed_amount) AS net_monthly_amount
  FROM signed_txns
  GROUP BY customer_id, date_month
),
closing_balance AS (
  SELECT
    customer_id,
    date_month,
    SUM(net_monthly_amount) OVER (
      PARTITION BY customer_id
      ORDER BY date_month
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS closing_balance
  FROM monthly_sum
)
SELECT * FROM closing_balance
ORDER BY customer_id, date_month;
```
**Explanation**:


## 5. **Percentage of customers who increase their closing balance by >5%**
```sql
WITH signed_txns AS (
  SELECT
    customer_id,
    DATE_TRUNC('month', txn_date)::DATE AS date_month,
    CASE
      WHEN txn_type = 'deposit' THEN txn_amount
      WHEN txn_type IN ('withdraw', 'purchase') THEN -txn_amount
      ELSE 0
    END AS signed_amount
  FROM data_bank.customer_transactions
),
monthly_sum AS (
  SELECT
    customer_id,
    date_month,
    SUM(signed_amount) AS net_monthly_amount
  FROM signed_txns
  GROUP BY customer_id, date_month
),
closing_balance AS (
  SELECT
    customer_id,
    date_month,
    SUM(net_monthly_amount) OVER (
      PARTITION BY customer_id
      ORDER BY date_month
    ) AS closing_balance
  FROM monthly_sum
),
previous_balances AS (
  SELECT
    customer_id,
    date_month,
    closing_balance,
    LAG(closing_balance) OVER (
      PARTITION BY customer_id
      ORDER BY date_month
    ) AS previous_balance
  FROM closing_balance
),
filtered_customers AS (
  SELECT DISTINCT customer_id
  FROM previous_balances
  WHERE previous_balance IS NOT NULL
    AND previous_balance != 0
    AND closing_balance > previous_balance * 1.05
),
total_customers AS (
  SELECT COUNT(DISTINCT customer_id) AS total FROM data_bank.customer_transactions
)
SELECT
  ROUND(100.0 * COUNT(fc.customer_id) / MAX(tc.total), 2) AS percent_customers_with_gt5pct_increase
FROM filtered_customers fc, total_customers tc;
```
**Explanation**:

