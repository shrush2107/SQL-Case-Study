
# 🍕 Pizza Runner - SQL Analysis

## A. Pizza Metrics

### 1️⃣ How many pizzas were ordered?

```sql
SELECT COUNT(pizza_id) AS order_count 
FROM pizza_runner.customer_orders;
```



### 2️⃣ How many unique customer orders were made?

```sql
SELECT COUNT(DISTINCT order_id) AS unique_order_count 
FROM pizza_runner.customer_orders;
```


### 3️⃣ How many successful orders were delivered by each runner?

```sql
SELECT runner_id, COUNT(order_id)  
FROM pizza_runner.runner_orders 
WHERE cancellation IS NULL 
GROUP BY runner_id;
```

  
### 4️⃣ How many of each type of pizza was delivered?

```sql
SELECT pizza_id, COUNT(pizza_id) 
FROM pizza_runner.runner_orders 
JOIN pizza_runner.customer_orders ON runner_orders.order_id = customer_orders.order_id 
WHERE cancellation IS NULL 
GROUP BY pizza_id;
```

  
### 5️⃣ How many Vegetarian and Meatlovers were ordered by each customer?

```sql
SELECT customer_id, 
    SUM(CASE WHEN pizza_id = 1 THEN 1 ELSE 0 END) AS MeatLovers, 
    SUM(CASE WHEN pizza_id = 2 THEN 1 ELSE 0 END) AS Vegetarian 
FROM pizza_runner.customer_orders 
GROUP BY customer_id;
```

  
### 6️⃣ What was the maximum number of pizzas delivered in a single order?

```sql
SELECT runner_orders.order_id, COUNT(pizza_id) AS pizza_count 
FROM pizza_runner.runner_orders 
JOIN pizza_runner.customer_orders ON runner_orders.order_id = customer_orders.order_id 
WHERE cancellation IS NULL 
GROUP BY runner_orders.order_id 
ORDER BY pizza_count DESC 
LIMIT 1;
```


### 7️⃣ For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```sql
SELECT customer_orders.customer_id, 
    SUM(CASE WHEN exclusions != '' OR extras != '' THEN 1 ELSE 0 END) AS change_is_there, 
    SUM(CASE WHEN exclusions = '' AND extras = '' THEN 1 ELSE 0 END) AS no_change 
FROM pizza_runner.runner_orders  
JOIN pizza_runner.customer_orders ON runner_orders.order_id = customer_orders.order_id 
WHERE cancellation IS NULL 
GROUP BY customer_orders.customer_id 
ORDER BY customer_orders.customer_id;
```
  

### 8️⃣ How many pizzas were delivered that had both exclusions and extras?

```sql
SELECT SUM(CASE WHEN exclusions != '' AND extras != '' THEN 1 ELSE 0 END) AS exclusions_and_extras 
FROM pizza_runner.runner_orders  
JOIN pizza_runner.customer_orders ON runner_orders.order_id = customer_orders.order_id 
WHERE cancellation IS NULL;
```


### 9️⃣ What was the total volume of pizzas ordered for each hour of the day?

```sql
SELECT EXTRACT(HOUR FROM order_time) AS hour_of_day, 
    COUNT(order_id) AS pizza_count 
FROM pizza_runner.customer_orders 
GROUP BY EXTRACT(HOUR FROM order_time) 
ORDER BY hour_of_day;
```


### 🔟 What was the volume of orders for each day of the week?

```sql
SELECT TO_CHAR(order_time, 'Day') AS day_of_week, 
    COUNT(order_id) AS pizza_count 
FROM pizza_runner.customer_orders 
GROUP BY TO_CHAR(order_time, 'Day') 
ORDER BY day_of_week;
```

  

## B. Runner and Customer Experience

### 1️⃣ How many runners signed up for each 1 week period? (week starts 2021-01-01)

```sql
SELECT DATE_trunc('week', registration_date) AS week_start, 
    COUNT(runner_id) AS runner_count 
FROM pizza_runner.runners 
GROUP BY DATE_trunc('week', registration_date) 
ORDER BY week_start;
```

Note: date_part and extract may return week numbers (1-52/53) which can cause ambiguity across years; DATE_trunc('week', registration_date) ensures correct grouping.


### 2️⃣ What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

```sql
SELECT runner_id, 
    ROUND(AVG(EXTRACT(MINUTE FROM pickup_time - order_time))) AS avg_time_taken_to_reach_HQ_mins 
FROM pizza_runner.runner_orders 
JOIN pizza_runner.customer_orders ON runner_orders.order_id = customer_orders.order_id 
WHERE cancellation IS NULL 
GROUP BY runner_id;
```
Note: assumed that difference is 0-59 mins. Googled later to find out how to correct this flaw: EXTRACT(EPOCH FROM pickup_time - order_time) / 60  ->EPOCH converts to seconds then divide by 60 for minutes

### 3️⃣ Is there any relationship between the number of pizzas and how long the order takes to prepare?

```sql
WITH prep_data AS (
    SELECT
        customer_orders.order_id,
        COUNT(customer_orders.pizza_id) AS num_pizzas,
        ROUND(AVG(EXTRACT(MINUTE FROM pickup_time - order_time))) AS prep_time_minutes
    FROM pizza_runner.customer_orders 
    JOIN pizza_runner.runner_orders ON customer_orders.order_id = runner_orders.order_id
    WHERE runner_orders.cancellation IS NULL
    GROUP BY customer_orders.order_id, runner_orders.pickup_time, customer_orders.order_time
)
SELECT
    num_pizzas,
    ROUND(AVG(prep_time_minutes)) AS avg_prep_time
FROM prep_data
GROUP BY num_pizzas
ORDER BY num_pizzas;
```
Note: assumed that difference is 0-59 mins 

### 4️⃣ What was the average distance travelled for each customer?

```sql
SELECT customer_id, 
    ROUND(AVG(distance)) 
FROM pizza_runner.runner_orders 
JOIN pizza_runner.customer_orders ON runner_orders.order_id = customer_orders.order_id 
WHERE cancellation IS NULL 
GROUP BY customer_id 
ORDER BY customer_id;
```


### 5️⃣ What was the difference between the longest and shortest delivery times for all orders?

```sql
SELECT MAX(duration) - MIN(duration) AS diff_btw_delivery_time 
FROM pizza_runner.runner_orders;
```



### 6️⃣ What was the average speed for each runner for each delivery and do you notice any trend for these values?

```sql
SELECT *, 
    ROUND((distance / (duration * 1.0 / 60))) AS "speed km/hr" 
FROM pizza_runner.runner_orders 
WHERE cancellation IS NULL;
```



### 7️⃣ What is the successful delivery percentage for each runner?

```sql
WITH temp AS (
    SELECT runner_id, 
        SUM(CASE WHEN cancellation IS NULL THEN 1 ELSE 0 END) AS successful_delivery, 
        COUNT(order_id) AS total_delivery 
    FROM pizza_runner.runner_orders 
    GROUP BY runner_id
)
SELECT *, 
    ROUND((successful_delivery * 1.0 / total_delivery * 1.0) * 100, 2) AS "% successful" 
FROM temp 
ORDER BY runner_id;
```

