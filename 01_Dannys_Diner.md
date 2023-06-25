# Case Study Questions

## 1. What is the total amount each customer spent at the restaurant?
```sql
SELECT s.customer_id, SUM(m.price) as total
FROM w1_sales s
LEFT JOIN w1_menu m ON s.product_id=m.product_id
GROUP BY s.customer_id
```
|customer_id|total|
|---|---|
|A|76|
|B|74|
|C|36|

## 2. How many days has each customer visited the restaurant?
```sql
SELECT s.customer_id, COUNT(distinct s.order_date) as visit
FROM w1_sales s
GROUP BY s.customer_id
```
|customer_id|visit|
|---|---|
|A|4|
|B|6|
|C|2|

## 3. What was the first item from the menu purchased by each customer?
```sql
WITH order_cte AS (
SELECT s.customer_id, ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS rank, s.product_id, m.product_name
FROM w1_sales s
LEFT JOIN w1_menu m ON s.product_id=m.product_id
)

SELECT customer_id, product_name
FROM order_cte
WHERE rank=1
```
|customer_id|product_name|
|---|---|
|A|sushi|
|B|curry|
|C|ramen|

## 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT TOP 1 product_id, COUNT(customer_id) AS purchase
FROM w1_sales 
GROUP BY product_id
ORDER BY purchase DESC;
```
|product_id|purchase|
|---|---|
|3|8|

## 5. Which item was the most popular for each customer?
```sql
WITH pop_cte AS(
SELECT s.customer_id, COUNT(*) AS purchase, RANK() OVER(PARTITION BY s.customer_id ORDER BY COUNT(s.product_id) DESC) as rank, s.product_id
FROM w1_sales s
GROUP BY s.customer_id, s.product_id
)

SELECT p.customer_id, m.product_name
FROM pop_cte p, w1_menu m
WHERE p.product_id=m.product_id AND p.rank=1
```
|customer_id|product_name|
|---|---|
|A|ramen|
|B|sushi|
|B|curry|
|B|ramen|
|C|ramen|

## 6. Which item was purchased first by the customer after they became a member?
```sql
WITH member_cte AS (
    SELECT s.customer_id, ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) AS rank, me.product_name
    FROM w1_sales s, w1_members m, w1_menu me
    WHERE s.customer_id=m.customer_id AND s.product_id=me.product_id AND m.join_date<=s.order_date
)
SELECT customer_id, product_name 
From member_cte
WHERE rank=1;
```
|customer_id|product_name|
|---|---|
|A|curry|
|B|sushi|

## 7. Which item was purchased just before the customer became a member?
```sql
WITH member_cte AS (
    SELECT s.customer_id, ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY s.order_date DESC) AS rank, me.product_name
    FROM w1_sales s, w1_members m, w1_menu me
    WHERE s.customer_id=m.customer_id AND s.product_id=me.product_id AND m.join_date>s.order_date
)
SELECT customer_id, product_name 
From member_cte
WHERE rank=1;
```
|customer_id|product_name|
|---|---|
|A|sushi|
|B|sushi|

## 8. What is the total items and amount spent for each member before they became a member?
```sql
SELECT s.customer_id, COUNT(s.product_id) AS items, SUM(price) AS total_spent
FROM w1_sales s, w1_members m, w1_menu me
WHERE s.customer_id=m.customer_id AND s.product_id=me.product_id AND m.join_date>s.order_date
GROUP BY s.customer_id
```
|customer_id|items|total_spent|
|---|---|---|
|A|2|25|
|B|3|40|

## 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
SELECT s.customer_id, SUM(CASE WHEN s.product_id=1 THEN price*20 ELSE price*10 END) AS point
FROM w1_sales s, w1_menu m
WHERE s.product_id=m.product_id
GROUP BY s.customer_id
```
|customer_id|point|
|---|---|
|A|860|
|B|940|
|C|360|

## 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH date_cte AS(
    SELECT *,DATEADD(DAY, 6, join_date) AS double_date
    FROM w1_members
)

SELECT s.customer_id, SUM(CASE WHEN order_date>=join_date AND order_date<double_date THEN price*20
                               WHEN order_date<join_date AND s.product_id=1 THEN price*20
                               WHEN order_date>EOMONTH('2021-01-01') THEN price*0
                               ELSE price*10 END)
FROM w1_sales s, date_cte d, w1_menu m
WHERE s.customer_id=d.customer_id AND m.product_id=s.product_id
GROUP BY s.customer_id
```
|customer_id|(No column name)|
|---|---|
|A|1370|
|B|820|

# Bonus Questions

## 1. Join all the things
```sql
WITH date_cte AS(
    SELECT *,DATEADD(DAY, 6, join_date) AS double_date
    FROM w1_members
)

SELECT s.customer_id, order_date, product_name, price, CASE WHEN order_date>=join_date THEN 'Y' ELSE 'N' END AS member
FROM w1_sales s
LEFT JOIN date_cte d on s.customer_id=d.customer_id 
LEFT JOIN w1_menu m on m.product_id=s.product_id
```

## 2. Rank All The Things
```sql
WITH date_cte AS(
    SELECT *,DATEADD(DAY, 6, join_date) AS double_date
    FROM w1_members
),

q1_cte AS(
SELECT s.customer_id, order_date, product_name, price, CASE WHEN order_date>=join_date THEN 'Y' ELSE 'N' END AS member
FROM w1_sales s
LEFT JOIN date_cte d on s.customer_id=d.customer_id 
LEFT JOIN w1_menu m on m.product_id=s.product_id
)

SELECT *, CASE WHEN member='Y' THEN RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date) ELSE NULL END
FROM q1_cte
``` 