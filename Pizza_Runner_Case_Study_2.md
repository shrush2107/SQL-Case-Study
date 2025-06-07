
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

  
