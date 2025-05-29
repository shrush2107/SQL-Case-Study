
# 📆🧠💰Case Study #4: Data Bank

<img src="https://8weeksqlchallenge.com/images/case-study-designs/4.png" alt="Data Bank" width="500" height="520">

> **Note:** All information about this case study is sourced from the official site: [8 Week SQL Challenge – Case Study #4](https://8weeksqlchallenge.com/case-study-4/)

---

##  Business Task

**Data Bank** is a digital banking provider offering customers access to financial transactions and node-based account routing. The business wants to better understand user behavior across their platform to enhance product offerings, improve customer experience, and support strategic decision-making.

You have been given access to two core datasets:
- `customer_nodes`: tracks customer movements between different node locations over time.
- `customer_transactions`: logs customer transactions, including deposits, withdrawals, and purchases.

As a data analyst at Data Bank, your mission is to uncover insights around:

-  **Node reallocation behavior**  
-  **Monthly transaction patterns**  
-  **Customer financial engagement**  
-  **Balance trends over time**  
-  **Identifying financially active customers**

The insights generated will be used to inform marketing strategies, detect customer churn, and support the development of features that promote healthy financial habits and retention.



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
- Look at each customer’s move-in dates (start_date) from a table.
- Use LEAD() to find the next move-in date for the same customer.
- Subtract the two dates to get the number of days between moves.
- Finally, average those numbers (ignores rows with no "next" date) and rounds it.

# Part B
## 1. **What is the unique count and total amount for each transaction type?**
```sql
SELECT txn_type, COUNT(customer_id), SUM(txn_amount)
FROM data_bank.customer_transactions
GROUP BY txn_type;
```
**Explanation**:
- Group the data by type of transaction (like “debit”, “credit”, etc.).
- Count how many transactions happened for each type.
- Add up the total amount of money for each type.

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
- Look only at deposit transactions.
- For each customer, count how many deposits they made and how much money they deposited.
- Then average those numbers across all customers.

## 3. **For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**
```sql
WITH monthly_activity AS (
  SELECT
    customer_id,
    DATE_TRUNC('month', txn_date)::DATE AS month,
    SUM(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS deposit_count,
    SUM(CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) AS purchase_count,
    SUM(CASE WHEN txn_type = 'withdraw' THEN 1 ELSE 0 END) AS withdraw_count
  FROM data_bank.customer_transactions
  GROUP BY customer_id, DATE_TRUNC('month', txn_date)::DATE
),
filtered_customers AS (
  SELECT
    month,
    customer_id
  FROM monthly_activity
  WHERE deposit_count > 1
    AND (purchase_count >= 1 OR withdraw_count >= 1)
)
SELECT
  month,
  COUNT(DISTINCT customer_id) AS qualifying_customers
FROM filtered_customers
GROUP BY month
ORDER BY month;

```
**Explanation**:
- First part look at each customer's monthly transactions and counts: How many deposits, How many purchases, How many withdrawals
- Then it filter to keep only the customers who: Made more than 1 deposit, and Made at least 1 purchase or withdrawal in that month
- Finally, it count how many unique customers met that condition in each month, and shows that number per month.

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
- Assign a sign to each transaction: Deposits stay positive. Withdrawals and purchases become negative.
- Sum up signed transactions per customer per month - gives the net gain or loss for each month.
- Add up those monthly net amounts over time (month by month) for each customer - like a running total.


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

- Assign a sign to each transaction: Deposits stay positive. Withdrawals and purchases become negative.
- Sum up signed transactions per customer per month - gives the net gain or loss for each month.
- Add up those monthly net amounts over time (month by month) for each customer - like a running total.
- Sum the total amount of the transcations done for that month.
- Add previous month’s balance using LAG() so I can compare this month vs last month.
- Filter based on the criteria 
- Count the total of customers who meet the criteria and calculate the percentage.