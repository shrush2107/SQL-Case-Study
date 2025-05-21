# Foodie-Fi SQL Analysis


## 1. How many customers has Foodie-Fi ever had?

```sql
SELECT COUNT(DISTINCT customer_id) AS total_customers
FROM foodie_fi.subscriptions;
```

### Explanation:
- The `subscriptions` table holds records of customer plans.
- To count unique customers, we use `COUNT(DISTINCT customer_id)`.
- This tells us the total number of unique users who have ever subscribed to any plan (trial, basic, pro, etc.).

## 2. Monthly distribution of trial plan `start_date` values

```sql
SELECT 
    TO_CHAR(start_date, 'YYYY-MM') AS year_month,
    COUNT(DISTINCT customer_id) AS trial_starts
FROM foodie_fi.subscriptions
JOIN foodie_fi.plans ON subscriptions.plan_id = plans.plan_id
WHERE plan_name = 'trial'
GROUP BY year_month
ORDER BY year_month;
```

### Explanation:
- We’re focusing only on trial plans using `WHERE plan_name = 'trial'`.
- `TO_CHAR(start_date, 'YYYY-MM')` extracts the year and month for grouping.
- `COUNT(DISTINCT customer_id)` counts the number of unique trial starts per month.
- Useful to visualize adoption trends over time.

## 3. Plan start_date values after 2020, grouped by plan name

```sql
WITH temp AS (
    SELECT 
        TO_CHAR(start_date, 'YYYY') AS plan_year,
        plan_name,
        COUNT(DISTINCT customer_id) AS cust_count
    FROM foodie_fi.subscriptions
    JOIN foodie_fi.plans ON subscriptions.plan_id = plans.plan_id
    GROUP BY plan_year, plan_name
)
SELECT 
    plan_name,
    cust_count
FROM temp
WHERE CAST(plan_year AS INTEGER) > 2020;
```

### Explanation:
- The goal is to see how many customers subscribed to each plan after 2020.
- We convert the start date to `plan_year`.
- Count unique customers for each `plan_name`.
- Use a WITH clause (common table expression) for readability.
- Final filter: only include years after 2020.

## 4. Customer count and percentage of churned users

```sql
WITH churned AS (
    SELECT COUNT(DISTINCT customer_id) AS churn_count
    FROM foodie_fi.subscriptions
    JOIN foodie_fi.plans ON subscriptions.plan_id = plans.plan_id
    WHERE plan_name = 'churn'
),
total AS (
    SELECT COUNT(DISTINCT customer_id) AS total_count
    FROM foodie_fi.subscriptions
)
SELECT 
    ROUND((churn_count * 100.0 / total_count), 1) AS churn_percentage,
    churn_count
FROM churned, total;
```

### Explanation:
- First CTE (`churned`) counts how many customers churned.
- Second CTE (`total`) gets the total customer base.
- The final SELECT calculates the percentage of customers who churned.
- `ROUND(..., 1)` keeps the result to 1 decimal place.

## 5. Customers who churned straight after their free trial

```sql
WITH ranked_plans AS (
    SELECT 
        DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY start_date) AS plan_rank,
        customer_id,
        plans.plan_id
    FROM foodie_fi.subscriptions
    JOIN foodie_fi.plans ON subscriptions.plan_id = plans.plan_id
)
SELECT 
    ROUND(
        100.0 * COUNT(DISTINCT customer_id) / 
        (SELECT COUNT(DISTINCT customer_id) FROM foodie_fi.subscriptions)
    ) AS cust_churn_percentage,
    COUNT(DISTINCT customer_id) AS churn_count
FROM ranked_plans 
WHERE plan_rank = 2 AND plan_id = 4;
```

### Explanation:
- Uses `DENSE_RANK()` to identify the second plan each customer subscribed to.
- Filters those whose second plan was churn (`plan_id = 4`).
- Calculates how many churned right after the trial and what percentage of the total user base they represent.
- Assumes everyone starts with a trial.

## 6. Number and percentage of customer plans after the trial

```sql
WITH ranked_plans AS (
    SELECT 
        DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY start_date) AS plan_rank,
        customer_id,
        plans.plan_id,
        plan_name
    FROM foodie_fi.subscriptions
    JOIN foodie_fi.plans ON subscriptions.plan_id = plans.plan_id
)
SELECT 
    plan_name,
    COUNT(DISTINCT customer_id) AS count_cust,
    ROUND(
        100.0 * COUNT(DISTINCT customer_id) / 
        (SELECT COUNT(DISTINCT customer_id) FROM foodie_fi.subscriptions),
        2
    ) AS percentage
FROM ranked_plans
WHERE plan_rank = 2
GROUP BY plan_name
ORDER BY count_cust DESC;
```

### Explanation:
- Analyzes the second plan customers chose after their trial.
- Groups by `plan_name` and counts how many customers moved to each.
- Calculates the percentage of each path.
- Helps analyze customer transition behavior after trial.
