## Case Study #1 - Danny's Diner

#### 1. What is the total amount each customer spent at the restaurant?

```sql
SELECT
  s.customer_id,
  SUM(m.price) AS total_sales
FROM dannys_diner.sales s
  JOIN dannys_diner.menu m
    ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id
```
 customer_id | total_sales 
 ----------- | ----------- 
 A           | 76          
 B           | 74          
 C           | 36          


#### 2. How many days has each customer visited the restaurant?

```sql
SELECT
  s.customer_id,
  COUNT(DISTINCT s.order_date) AS unique_visits
FROM dannys_diner.sales s
GROUP BY s.customer_id
```
 customer_id | unique_visits 
 ----------- | ------------- 
 A           | 4             
 B           | 6             
 C           | 2             

#### 3. What was the first item from the menu purchased by each customer?

```sql
SELECT 
  s.customer_id,
  s.order_date,
  m.product_name
FROM dannys_diner.sales s
  JOIN dannys_diner.menu m
    ON s.product_id = m.product_id
WHERE 
  s.order_date IN
    (SELECT
      MIN(s.order_date)
    FROM dannys_diner.sales s
    GROUP BY
      s.customer_id
    )
ORDER BY
  s.customer_id
```	
 customer_id | order_date               | product_name 
 ----------- | ------------------------ | ------------ 
 A           | 2021-01-01T00:00:00.000Z | sushi        
 A           | 2021-01-01T00:00:00.000Z | curry        
 B           | 2021-01-01T00:00:00.000Z | curry        
 C           | 2021-01-01T00:00:00.000Z | ramen        
 C           | 2021-01-01T00:00:00.000Z | ramen        


#### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
SELECT
  m.product_name,
  COUNT(s.product_id) AS num_of_orders
FROM dannys_diner.menu m
  JOIN dannys_diner.sales s
    ON m.product_id = s.product_id
GROUP BY
  m.product_name
ORDER BY
  num_of_orders DESC
LIMIT 1
```
 product_name | num_of_orders 
 ------------ | ------------- 
 ramen        | 8               


#### 5. Which item was the most popular for each customer?

```sql
WITH order_count AS 
  (SELECT
      s.customer_id, 
      m.product_name, 
      COUNT(s.product_id) AS order_count,
      DENSE_RANK()
          OVER(
            PARTITION BY s.customer_id
            ORDER BY COUNT(s.customer_id) DESC) AS rank
  FROM dannys_diner.sales s
    JOIN dannys_diner.menu m
      ON s.product_id = m.product_id
  GROUP BY 
      s.customer_id, 
      m.product_name
  )
  
SELECT
  customer_id,
  product_name,
  order_count
FROM order_count
``` 	
 customer_id | product_name | order_count 
 ----------- | ------------ | ----------- 
 A           | ramen        | 3           
 B           | ramen        | 2           
 B           | curry        | 2           
 B           | sushi        | 2           
 C           | ramen        | 3           


#### 6. Which item was purchased first by the customer after they became a member?

```sql
WITH member_timeline AS 
  (SELECT 
      mb.customer_id,
      mb.join_date,
      s.order_date,
      s.product_id,
      DENSE_RANK()
    	OVER (PARTITION BY s.customer_id
        ORDER BY s.order_date) AS order_rank
  FROM dannys_diner.members mb
	  JOIN dannys_diner.sales s
      ON mb.customer_id = s.customer_id
  WHERE
      s.order_date >= mb.join_date
 ORDER BY
      mb.customer_id,
      s.order_date
)

SELECT
  mt.customer_id,
  mt.join_date,
  mt.order_date,
  m.product_name
FROM member_timeline mt
  JOIN dannys_diner.menu m
    ON mt.product_id = m.product_id
WHERE
  mt.order_rank = 1
ORDER BY
  mt.customer_id
```
 customer_id | join_date                | order_date               | product_name 
 ----------- | ------------------------ | ------------------------ | ------------ 
 A           | 2021-01-07T00:00:00.000Z | 2021-01-07T00:00:00.000Z | curry        
 B           | 2021-01-09T00:00:00.000Z | 2021-01-11T00:00:00.000Z | sushi        


#### 7. Which item was purchased just before the customer became a member?

```sql
WITH premember_timeline AS 
(SELECT
    mb.customer_id,
    mb.join_date,
    s.order_date,
    s.product_id,
    DENSE_RANK()
    	OVER (PARTITION BY s.customer_id
        ORDER BY s.order_date DESC) AS order_rank
 FROM dannys_diner.members mb
   JOIN dannys_diner.sales s
     ON mb.customer_id = s.customer_id
 WHERE
    s.order_date < mb.join_date
 ORDER BY
    mb.customer_id,
    s.order_date DESC
)

SELECT
  pmt.customer_id,
  pmt.join_date,
  pmt.order_date,
  m.product_name
FROM premember_timeline pmt
  JOIN dannys_diner.menu m
    ON pmt.product_id = m.product_id
WHERE
  pmt.order_rank = 1
ORDER BY
  pmt.customer_id
```
 customer_id | join_date                | order_date               | product_name 
 ----------- | ------------------------ | ------------------------ | ------------ 
 A           | 2021-01-07T00:00:00.000Z | 2021-01-01T00:00:00.000Z | sushi        
 A           | 2021-01-07T00:00:00.000Z | 2021-01-01T00:00:00.000Z | curry        
 B           | 2021-01-09T00:00:00.000Z | 2021-01-04T00:00:00.000Z | sushi        


#### 8. What is the total items and amount spent for each member before they became a member?

```sql
SELECT
  items.customer_id,
  items.total_items,
  total.total_spent
FROM
      (SELECT
          mb.customer_id,
          COUNT(s.order_date) as total_items
      FROM dannys_diner.members mb
          JOIN dannys_diner.sales s
            ON mb.customer_id = s.customer_id
      WHERE
          s.order_date < mb.join_date
      GROUP BY
          mb.customer_id
      ) items
  JOIN
      (SELECT
          mb.customer_id,
          SUM(m.price) AS total_spent
      FROM dannys_diner.members mb
          JOIN dannys_diner.sales s
            ON mb.customer_id = s.customer_id
          JOIN dannys_diner.menu m
            ON s.product_id = m.product_id
      WHERE
          s.order_date < mb.join_date
      GROUP BY
          mb.customer_id
      ORDER BY
          mb.customer_id
      ) total
  ON items.customer_id = total.customer_id
```
 customer_id | total_items | total_spent 
 ----------- | ----------- | ----------- 
 A           | 2           | 25          
 B           | 3           | 40          
  
#### 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

```sql
WITH ep AS 
(SELECT
  s.customer_id,
  m.product_name,
  m.price,
  CASE WHEN m.product_id = 1 THEN 20
       ELSE 10 
        END eligible_points,
  (m.price * (CASE WHEN m.product_id = 1 THEN 20
                   ELSE 10 
                    END)) 
        AS points_earned
FROM dannys_diner.sales s
  JOIN dannys_diner.menu m
    ON s.product_id = m.product_id
ORDER BY
	s.customer_id
)

SELECT
  customer_id,
  SUM(points_earned) AS total_points
FROM ep
GROUP BY
  customer_id
```
 customer_id | total_points 
 ----------- | ------------ 
 A           | 860          
 B           | 940          
 C           | 360          


#### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH jan_member_pts AS 
(SELECT
    s.customer_id,
    mb.join_date,
    s.order_date,
    m.product_name,
    m.price,
    (m.price * (CASE WHEN m.product_id = 1 THEN 20
                     ELSE 10 END) * 
               (CASE WHEN m.product_id != 1 AND 
                          s.order_date BETWEEN mb.join_date AND mb.join_date + 6 THEN 2
                     ELSE 1 END))
        AS points_earned
FROM dannys_diner.sales s
  JOIN dannys_diner.members mb
    ON s.customer_id = mb.customer_id
  JOIN dannys_diner.menu m
    ON s.product_id = m.product_id
WHERE
  s.order_date BETWEEN '2021-01-01' AND '2021-01-31'
ORDER BY
  s.customer_id,
  s.order_date
)

SELECT
  customer_id,
  SUM(points_earned) AS january_points
FROM jan_member_pts
GROUP BY
  customer_id
```
 customer_id | january_points 
 ----------- | -------------- 
 A           | 1370           
 B           | 820       

#### Bonus Question: Join All The Things!  
(Re)create the table with customer id, order date, product name, product price, and Y/N on whether the customer was a member when placing the order.

```sql
SELECT
  s.customer_id,
  s.order_date,
  m.product_name,
  m.price,
  CASE WHEN s.order_date >= mb.join_date THEN 'Y'
       ELSE 'N' 
        END AS member
FROM dannys_diner.sales s
  LEFT JOIN dannys_diner.menu m
         ON s.product_id = m.product_id
  LEFT JOIN dannys_diner.members mb
         ON s.customer_id = mb.customer_id
ORDER BY
  s.customer_id,
  s.order_date 
```
 customer_id | order_date               | product_name | price | member 
 ----------- | ------------------------ | ------------ | ----- | ------ 
 A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      
 A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      
 A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      
 A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      
 A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      
 A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      
 B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      
 B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      
 B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      
 B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      
 B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      
 B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      
 C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      
 C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      
 C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      

#### Bonus Question: Rank All The Things!  
Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.

```sql
WITH member_orders AS 
(SELECT
    s.customer_id,
    s.order_date,
    m.product_name,
    m.price,
    CASE WHEN s.order_date >= mb.join_date THEN 'Y'
         ELSE 'N' 
          END AS member
FROM dannys_diner.sales s
  LEFT JOIN dannys_diner.menu m
         ON s.product_id = m.product_id
  LEFT JOIN dannys_diner.members mb
         ON s.customer_id = mb.customer_id
ORDER BY
    s.customer_id,
    s.order_date
)

SELECT
  *,
  CASE WHEN member = 'N' THEN NULL
       ELSE RANK()
              OVER (PARTITION BY customer_id, member
                    ORDER BY order_date)
        END AS ranking
FROM member_orders
```
 customer_id | order_date               | product_name | price | member | ranking 
 ----------- | ------------------------ | ------------ | ----- | ------ | ------- 
 A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |         
 A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         
 A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      | 1       
 A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      | 2       
 A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       
 A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       
 B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         
 B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |         
 B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |         
 B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      | 1       
 B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      | 2       
 B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      | 3       
 C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         
 C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         
 C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |         
