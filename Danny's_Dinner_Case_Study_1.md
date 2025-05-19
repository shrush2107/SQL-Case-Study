# 🍛🍣🍜 Case Study #1: Danny's Diner
<img src="https://8weeksqlchallenge.com/images/case-study-designs/1.png" alt="Image" width="500" height="520">

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

***

## Question and Solution

### Creating the schema:
````sql
CREATE SCHEMA dannys_diner;
use dannys_diner;

CREATE TABLE sales (
  customer_id VARCHAR(1),
  order_date DATE,
  product_id INTEGER
);

INSERT INTO sales
  (customer_id, order_date, product_id)
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  product_id INTEGER,
  product_name VARCHAR(5),
  price INTEGER
);

INSERT INTO menu
  (product_id, product_name, price)
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  customer_id VARCHAR(1),
  join_date DATE
);

INSERT INTO members
  (customer_id, join_date)
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');
````


### 1.What is the total amount each customer spent at the restaurant?
````sql 
SELECT 
    customer_id, CONCAT('$ ', SUM(price)) AS total_spend
FROM
    sales
        JOIN
    menu ON sales.product_id = menu.product_id
GROUP BY customer_id;
````
STEPS:
- The query joins the sales table with the menu table on the product_id.
- It groups the results by customer_id.
- It calculates the total amount spent for each customer.
- The CONCAT('$ ', SUM(price)) concatenates the dollar sign with the sum of prices for each customer.

### 2.How many days has each customer visited the restaurant?
````sql
SELECT 
    customer_id, COUNT(DISTINCT (order_date)) AS total_visits
FROM
    sales
GROUP BY customer_id;
````
STEPS:
- To determine the unique number of visits for each customer, utilize COUNT(DISTINCT `order_date`).
- Remember to include the DISTINCT keyword to ensure accurate counting, preventing duplicate inclusion of days. Without DISTINCT, multiple visits on the same day, like Customer A's two visits on '2021-01-07', would incorrectly inflate the count to 2 days instead of the correct count of 1 day.


### 3.What was the first item from the menu purchased by each customer?
````sql
WITH temp AS (
    SELECT 
        customer_id, order_date, product_name, 
        DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS purchase_rank 
    FROM 
        sales 
        JOIN 
        menu ON sales.product_id = menu.product_id
)
SELECT 
    customer_id, product_name 
FROM 
    temp 
WHERE 
    purchase_rank = 1;
````
STEPS:
- Create a temporary table temp using a Common Table Expression (CTE) named temp.
- Within the temp table, select columns customer_id, order_date, product_name, and calculates a ranking for each purchase made by a customer (purchase_rank). The DENSE_RANK() function is used, which assigns a rank to each row within a partition of the result set. Here, the partition is defined by the customer_id, and the rows are ordered by order_date.
- The sales table is joined with the menu table based on the product_id to obtain the product_name.
- Once the temp table is created, the main query selects customer_id and product_name from the temp table where the purchase_rank is equal to 1. This means it selects the first purchase made by each customer, as determined by the ordering of order_date.

### 4.What is the most purchased item on the menu and how many times was it purchased by all customers?
````sql
SELECT 
    product_name, COUNT(menu.product_id) AS purchase_count
FROM
    sales
        JOIN
    menu ON sales.product_id = menu.product_id
GROUP BY sales.product_id
ORDER BY purchase_count DESC
LIMIT 1;
````
STEPS:
- Perform a COUNT aggregation on the product_id column and ORDER BY the result in descending order using purchase_count field.
- Apply the LIMIT 1 clause to filter and retrieve the highest number of purchased items.

### 5.Which item was the most popular for each customer?
````sql
WITH customer_popular AS (
    SELECT 
        customer_id, product_name, COUNT(product_name) AS quantity_purchased 
    FROM 
        sales 
    JOIN 
        menu ON sales.product_id = menu.product_id 
    GROUP BY 
        customer_id , product_name
)
SELECT 
    customer_id, product_name, quantity_purchased 
FROM 
    (
        SELECT 
            customer_id, product_name, quantity_purchased, 
            DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY quantity_purchased DESC) AS popular_ranking 
        FROM 
            customer_popular
    ) AS popular_for_customer 
WHERE 
    popular_ranking = 1;
````

### 6.Which item was purchased first by the customer after they became a member?
````sql
WITH temp AS (
    SELECT 
        sales.customer_id, order_date, product_id, join_date, 
        DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS order_rank
    FROM 
        sales 
    JOIN 
        members ON sales.customer_id = members.customer_id 
    WHERE 
        sales.order_date > members.join_date
)
SELECT 
    customer_id, product_name 
FROM 
    temp 
JOIN 
    menu ON temp.product_id = menu.product_id 
WHERE 
    order_rank = '1';
````

### 7.Which item was purchased just before the customer became a member?
````sql
WITH temp AS (
    SELECT 
        sales.customer_id, order_date, product_id, join_date, 
        DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date DESC) AS order_rank
    FROM 
        sales 
    JOIN 
        members ON sales.customer_id = members.customer_id 
    WHERE 
        sales.order_date < members.join_date
)
SELECT 
    customer_id, product_name 
FROM 
    temp 
JOIN 
    menu ON temp.product_id = menu.product_id 
WHERE 
    order_rank = '1';
````

### 8.What is the total items and amount spent for each member before they became a member?
````sql
SELECT 
    sales.customer_id,
    COUNT(order_date) AS total_items,
    SUM(price) AS total_amount
FROM
    sales
        JOIN
    members ON sales.customer_id = members.customer_id
        JOIN
    menu ON sales.product_id = menu.product_id
WHERE
    sales.order_date < members.join_date
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
````
###  9.If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
````sql
WITH customerPoints AS (SELECT 
    customer_id,
    product_name,
    price,
    CASE
        WHEN product_name = 'sushi' THEN price * 10 * 2
        ELSE price * 10
    END AS points
FROM
    sales
        JOIN
    menu ON sales.product_id = menu.product_id)
SELECT 
    customer_id, SUM(points) AS total_points
FROM
    customerPoints
GROUP BY customer_id
ORDER BY customer_id;
````

### 10.In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?


## Bonus Questions
### 11. Recreate the table with: customer_id, order_date, product_name, price, member (Y/N)
````sql
SELECT 
    sales.customer_id, 
    sales.order_date, 
    menu.product_name, 
    menu.price,
    CASE
        WHEN sales.order_date >= members.join_date THEN 'Y'
        ELSE 'N'
    END AS member
FROM 
    sales 
JOIN 
    menu ON sales.product_id = menu.product_id 
LEFT JOIN 
    members ON sales.customer_id = members.customer_id 
ORDER BY 
    sales.customer_id, 
    sales.order_date;
````
STEPS:
