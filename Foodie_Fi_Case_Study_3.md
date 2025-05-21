# 🍕📺💡 Case Study #3: Foodie-Fi  
<img src="https://8weeksqlchallenge.com/images/case-study-designs/3.png" alt="Image" width="500" height="520">

> **Note:** All information about this case study is sourced from the official site: [8 Week SQL Challenge – Case Study #3](https://8weeksqlchallenge.com/case-study-3/)

***

## Business Task

Foodie-Fi is a subscription-based video streaming service offering a range of plans from a free trial to basic, pro, and annual packages. The business wants to understand customer behaviors across different subscription plans to improve product offerings and maximize conversions.

You have been given access to a sample database representing customer subscription history and plan metadata.

Danny (the analyst assisting Foodie-Fi) is tasked with uncovering insights about:

- Customer plan conversion timelines  
- Retention trends  
- Upgrade and churn behavior  
- Customer distribution across subscription types  
- Metrics to support pricing and marketing decisions


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


## 8. How many customers have upgraded to an annual plan in 2020?

```sql
SELECT COUNT(DISTINCT customer_id) AS annual_2020_customers
FROM foodie_fi.subscriptions
WHERE plan_id = 3 
  AND start_date <= '2020-12-31';
```

### Explanation:
- `plan_id = 3` represents the **annual plan**.
- Filters subscriptions that started **on or before** the end of 2020.
- Counts unique customers who upgraded to annual within that year.

---

## 9. How many days on average does it take for a customer to move to an annual plan from the day they join?

```sql
SELECT 
  ROUND(AVG(tannual.start_date - tstart.start_date)) AS avg_days_to_annual
FROM foodie_fi.subscriptions tstart
JOIN foodie_fi.subscriptions tannual 
  ON tstart.customer_id = tannual.customer_id
WHERE tstart.plan_id = 0   -- Trial
  AND tannual.plan_id = 3; -- Annual
```

### Explanation:
- Measures how long it takes customers to upgrade to the annual plan.
- Joins subscriptions on `customer_id` to pair each customer's trial and annual records.
- Calculates the date difference between trial start and annual upgrade.
- Aggregates the average number of days.

---

## 10. Count of customers who converted to annual within specific days (30-day window breakdown)

```sql
SELECT 
  FLOOR((tannual.start_date - tstart.start_date) / 30) AS bucket_number,
  COUNT(*) AS customer_count
FROM foodie_fi.subscriptions tstart
JOIN foodie_fi.subscriptions tannual 
  ON tstart.customer_id = tannual.customer_id
WHERE tstart.plan_id = 0   
  AND tannual.plan_id = 3  
GROUP BY bucket_number
ORDER BY bucket_number;
```

### Explanation:
- Tracks time taken (in days) for each customer to go from a trial to an annual plan.
- Helpful to create **bucketed conversion time windows** (e.g., within 7 days, 14 days, 30 days, etc.).
- Enables time-based segmentation of upgrade behavior.

---

## 11. How many customers downgraded from Pro Monthly to Basic Monthly in 2020?

```sql
SELECT COUNT(DISTINCT tstart.customer_id) AS downgraded_customers
FROM foodie_fi.subscriptions tstart
JOIN foodie_fi.subscriptions tnext 
  ON tstart.customer_id = tnext.customer_id
WHERE tstart.plan_id = 2                     
  AND tnext.plan_id = 1                      
  AND tnext.start_date > tstart.start_date   
  AND tnext.start_date <= '2020-12-31';
```

### Explanation:
- Identifies customers who **downgraded** their plan.
- `plan_id = 2` is Pro Monthly; `plan_id = 1` is Basic Monthly.
- Checks that the downgrade occurred **after** the upgrade (`tnext.start_date > tstart.start_date`).
- Restricts downgrades to those made in **2020** only.
