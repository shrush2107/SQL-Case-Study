
# 🍕 Case Study #2: Pizza Runner  

<img src="https://8weeksqlchallenge.com/images/case-study-designs/2.png" alt="Data Bank" width="500" height="520">

**Note:** All information about this case study is sourced from the official site: [8 Week SQL Challenge – Case Study #2](https://8weeksqlchallenge.com/case-study-2/)

---

## Business Task  
Danny and his friends decided to create a pizza delivery service called **Pizza Runner**. While Danny was initially handling all the deliveries by himself, demand quickly grew, prompting him to recruit additional runners to help with order fulfillment.

Now that the business is growing, Danny wants to better understand the operational aspects of Pizza Runner. This includes analyzing delivery performance, runner engagement, and customer ordering behaviors. He needs insights that will help him optimize delivery processes, identify bottlenecks, and improve customer satisfaction.

Danny has shared a subset of the data due to privacy reasons, but he believes it contains enough relevant examples for you to write SQL queries that can help him gain deeper insights into the business. He also hopes to generate clean and readable summary datasets that his team can access without writing complex queries.

- How many successful orders were delivered?
- What are the average delivery times?
- How frequently are runners completing deliveries?
- What are the most popular pizza combinations?

These insights will play a key role in streamlining operations, tracking performance, and potentially launching loyalty offers or promotions.


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

## C.Ingredient Optimisation

### 1️⃣ What are the standard ingredients for each pizza?

```sql
WITH split_toppings AS (
  SELECT
    pizza_id,
    REGEXP_SPLIT_TO_TABLE(TRIM(toppings), '[,\s]+')::INTEGER AS topping_id
  FROM pizza_recipes
  WHERE toppings IS NOT NULL AND toppings <> ''
)
SELECT
  pn.pizza_name,
  STRING_AGG(pt.topping_name, ', ') AS toppings
FROM split_toppings st
JOIN pizza_names pn ON st.pizza_id = pn.pizza_id
JOIN pizza_toppings pt ON st.topping_id = pt.topping_id
GROUP BY pn.pizza_name;
```
Notes:
-- normalizing the table is a better way to go with.
-- string_agg(for readability) vs array_agg(for further processing)
-- REGEXP_SPLIT_TO_TABLE -> doesn't work on null or ""

### 2️⃣ What was the most commonly added extra?

```sql
WITH expanded_temp AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    REGEXP_SPLIT_TO_TABLE(TRIM(extras), '[,\s]+')::INTEGER AS extra_id
  FROM customer_orders
  WHERE extras IS NOT NULL AND extras <> ''
)
SELECT
  extra_id,
  COUNT(extra_id) AS topping_count,
  pt.topping_name
FROM expanded_temp et
JOIN pizza_toppings pt ON et.extra_id = pt.topping_id
GROUP BY extra_id, pt.topping_name
ORDER BY topping_count DESC
LIMIT 1;
```


### 3️⃣ What was the most common exclusion?

```sql
WITH expanded_temp AS (
  SELECT
    order_id,
    customer_id,
    pizza_id,
    order_time,
    REGEXP_SPLIT_TO_TABLE(TRIM(exclusions), '[,\s]+')::INTEGER AS exclusion_id
  FROM customer_orders
  WHERE exclusions IS NOT NULL AND exclusions <> ''
)
SELECT
  exclusion_id,
  COUNT(exclusion_id) AS topping_count,
  pt.topping_name
FROM expanded_temp et
JOIN pizza_toppings pt ON et.exclusion_id = pt.topping_id
GROUP BY exclusion_id, pt.topping_name
ORDER BY topping_count DESC
LIMIT 1;
```

## D. Pricing

### 1️⃣ If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?

```sql
SELECT 
    SUM(CASE 
        WHEN pn.pizza_name = 'Meatlovers' THEN 12
        ELSE 10
    END) AS total_revenue
FROM customer_orders c
JOIN runner_orders r ON c.order_id = r.order_id
JOIN pizza_names pn ON pn.pizza_id = c.pizza_id
WHERE r.cancellation IS NULL;
```

### 2️⃣ What if there was an additional $1 charge for any pizza extras?

``` sql 
WITH base_revenue AS (
    SELECT 
        CASE 
            WHEN pn.pizza_name = 'Meatlovers' THEN 12
            ELSE 10
        END AS price
    FROM customer_orders c
    JOIN runner_orders r ON c.order_id = r.order_id
    JOIN pizza_names pn ON pn.pizza_id = c.pizza_id
    WHERE r.cancellation IS NULL
),
extra_toppings AS (
    SELECT 
        REGEXP_SPLIT_TO_TABLE(TRIM(extras), '[,\s]+') AS topping_id
    FROM customer_orders c
    JOIN runner_orders r ON c.order_id = r.order_id
    WHERE r.cancellation IS NULL 
      AND extras IS NOT NULL 
      AND extras <> ''
)
SELECT 
    (SELECT SUM(price) FROM base_revenue) + COUNT(*) AS total_revenue
FROM extra_toppings;
```

### 3️⃣ If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

``` sql
WITH temp AS (
    SELECT
        r.order_id,
        r.distance,
        r.runner_id,
        r.pickup_time,
        r.duration,
        r.cancellation,
        c.customer_id,
        pn.pizza_name,
        distance * 0.3 AS delivery_fee,
        CASE 
            WHEN pn.pizza_name = 'Meatlovers' THEN 12
            ELSE 10
        END AS pizza_price
    FROM customer_orders c
    JOIN runner_orders r ON c.order_id = r.order_id
    JOIN pizza_names pn ON pn.pizza_id = c.pizza_id
    WHERE r.cancellation IS NULL
),

temp2 AS (
    SELECT 
        order_id,
        MAX(delivery_fee) AS per_order_fee
    FROM temp
    GROUP BY order_id
)

SELECT 
    (SELECT SUM(pizza_price) FROM temp) - 
    (SELECT SUM(per_order_fee) FROM temp2) AS net_profit;

```



