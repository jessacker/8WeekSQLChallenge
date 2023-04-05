## Case Study Questions
### A. Enterprise Relationship Diagram
Using the associated DDL schema details to create an ERD for all the Clique Bait datasets.
![6CliqueBait_ERdiagram](https://user-images.githubusercontent.com/116126763/225055242-50811dd9-8edb-4404-9809-ac77c5af507e.PNG)

### B. Digital Analysis
Using the available datasets - answer the following questions using a single query for each one:

**1. How many users are there?**
```sql
SELECT
  COUNT(DISTINCT user_id)
FROM clique_bait.users
```
| count |
| ----- |
| 500   |

**2. How many cookies does each user have on average?**
```sql
SELECT
  ROUND(AVG(cookie_per_user)) AS avg_cookies_per_user
FROM
  (SELECT
     user_id,
     COUNT(cookie_id) AS cookie_per_user
  FROM clique_bait.users
  GROUP BY
     user_id
  ) cpu
```
| avg_cookies_per_user |
| -------------------- |
| 4                    |

**3. What is the unique number of visits by all users per month?**
```sql
SELECT
  EXTRACT('month' FROM event_time) AS month,
  COUNT (DISTINCT visit_id) AS unique_visits
FROM clique_bait.events
GROUP BY
  month
```
| month | unique_visits |
| ----- | ------------- |
| 1     | 876           |
| 2     | 1488          |
| 3     | 916           |
| 4     | 248           |
| 5     | 36            |

**4. What is the number of events for each event type?**
```sql
SELECT
  e.event_type,
  ei.event_name,
  COUNT(e.visit_id)
FROM clique_bait.events e
  JOIN clique_bait.event_identifier ei
    ON e.event_type = ei.event_type
GROUP BY
  e.event_type,
  ei.event_name
ORDER BY
  e.event_type
```
| event_type | event_name    | count |
| ---------- | ------------- | ----- |
| 1          | Page View     | 20928 |
| 2          | Add to Cart   | 8451  |
| 3          | Purchase      | 1777  |
| 4          | Ad Impression | 876   |
| 5          | Ad Click      | 702   |

**5. What is the percentage of visits which have a purchase event?**
```sql
SELECT
  ROUND((SELECT
           COUNT(visit_id) AS purchases
         FROM clique_bait.events
         WHERE
           event_type = 3
         GROUP BY
           event_type
         )::DECIMAL
      / COUNT (DISTINCT visit_id)
      * 100, 1) || '%'
    AS purchase_percent
FROM clique_bait.events
```
| purchase_percent |
| ---------------- |
| 49.9%            |

**6. What is the percentage of visits which view the checkout page but do not have a purchase event?**
```sql
SELECT
  ph.page_name,
  COUNT(DISTINCT e.visit_id) AS no_purchase_event_total,
  ROUND(COUNT(DISTINCT e.visit_id)::DECIMAL /
          (SELECT
             COUNT(DISTINCT visit_id)
           FROM clique_bait.events
          ) * 100, 1) || '%'
       AS percent_of_total_visits
FROM clique_bait.events e
  JOIN clique_bait.page_hierarchy ph
    ON e.page_id = ph.page_id
WHERE
  e.page_id = 12 AND
  e.visit_id NOT IN 
                (SELECT 
                   DISTINCT e.visit_id 
                 FROM clique_bait.events e 
                 WHERE 
                   e.event_type = 3)
GROUP BY
  ph.page_name
```
| page_name | no_purchase_event_total | percent_of_total_visits |
| --------- | ----------------------- | ----------------------- |
| Checkout  | 326                     | 9.1%                    |

**7. What are the top 3 pages by number of views?**
```sql
SELECT
  ph.page_name,
  COUNT(e.visit_id) AS views
FROM clique_bait.events e
  JOIN clique_bait.page_hierarchy ph
    ON e.page_id = ph.page_id
WHERE
  e.event_type = 1 -- page views only
GROUP BY
  ph.page_name
ORDER BY
  views DESC
LIMIT 3
```
| page_name    | views |
| ------------ | ----- |
| All Products | 3174  |
| Checkout     | 2103  |
| Home Page    | 1782  |

**8. What is the number of views and cart adds for each product category?**
```sql
SELECT
  ph.product_category,
  SUM(CASE WHEN e.event_type = 1 THEN 1 
           ELSE 0 END) 
        AS view_count,
  SUM(CASE WHEN e.event_type = 2 THEN 1 
           ELSE 0 END) 
        AS addcart_count
FROM clique_bait.events e
  JOIN clique_bait.page_hierarchy ph
    ON e.page_id = ph.page_id
WHERE
  ph.product_category IS NOT NULL
GROUP BY
  ph.product_category
ORDER BY
  ph.product_category
```
| product_category | view_count | addcart_count |
| ---------------- | ---------- | ------------- |
| Fish             | 4633       | 2789          |
| Luxury           | 3032       | 1870          |
| Shellfish        | 6204       | 3792          |

**9. What are the top 3 products by purchases?**
```sql
SELECT
  ph.page_name,
  SUM(CASE WHEN e.event_type = 2 THEN 1 ELSE 0 END) AS purchase_count
FROM clique_bait.events e
  JOIN clique_bait.page_hierarchy ph
    ON e.page_id = ph.page_id
WHERE
  e.visit_id IN
          (SELECT
             visit_id
           FROM clique_bait.events
           WHERE
             event_type = 3
          )
GROUP BY
  ph.page_name
ORDER BY
  purchase_count DESC
LIMIT 3
```
| page_name | purchase_count |
| --------- | -------------- |
| Lobster   | 754            |
| Oyster    | 726            |
| Crab      | 719            |

## C. Product Funnel Analysis
Using a single SQL query - create a new output table which has the following details:
* How many times was each product viewed?
* How many times was each product added to cart?
* How many times was each product added to a cart but not purchased (abandoned)?
* How many times was each product purchased?
```sql
CREATE TABLE IF NOT EXISTS product_funnel AS
(SELECT
   ph.page_name AS product_name,
   SUM(CASE WHEN e.event_type = 1 THEN 1 
            ELSE 0 END) AS view_count,
   SUM(CASE WHEN e.event_type = 2 THEN 1 
            ELSE 0 END) AS cartadd_count, 
   SUM(CASE WHEN e.event_type = 2 AND 
                 e.visit_id NOT IN 
                      (SELECT 
                         visit_id 
                       FROM clique_bait.events 
                       WHERE 
                         event_type = 3) THEN 1 
            ELSE 0 END) AS abandon_count,
   SUM(CASE WHEN e.event_type = 2 AND 
                 e.visit_id IN 
                      (SELECT 
                         visit_id 
                       FROM clique_bait.events 
                       WHERE 
                         event_type = 3) THEN 1 
            ELSE 0 END) AS purchase_count
FROM clique_bait.events e
  JOIN clique_bait.page_hierarchy ph
    ON e.page_id = ph.page_id
WHERE
  ph.product_category IS NOT NULL
GROUP BY
  ph.page_name
ORDER BY
  ph.page_name
```
| product_name   | view_count | cartadd_count | abandon_count | purchase_count |
| -------------- | ---------- | ------------- | ------------- | -------------- |
| Abalone        | 1525       | 932           | 233           | 699            |
| Black Truffle  | 1469       | 924           | 217           | 707            |
| Crab           | 1564       | 949           | 230           | 719            |
| Kingfish       | 1559       | 920           | 213           | 707            |
| Lobster        | 1547       | 968           | 214           | 754            |
| Oyster         | 1568       | 943           | 217           | 726            |
| Russian Caviar | 1563       | 946           | 249           | 697            |
| Salmon         | 1559       | 938           | 227           | 711            |
| Tuna           | 1515       | 931           | 234           | 697            |

Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.
```sql
CREATE TABLE IF NOT EXISTS product_category_funnel AS
(SELECT
   ph.product_category,
   SUM(CASE WHEN e.event_type = 1 THEN 1 
            ELSE 0 END) AS view_count,
   SUM(CASE WHEN e.event_type = 2 THEN 1 
            ELSE 0 END) AS cartadd_count,
   SUM(CASE WHEN e.event_type = 2 AND 
                 e.visit_id NOT IN 
                      (SELECT 
                         visit_id 
                       FROM clique_bait.events 
                       WHERE 
                         event_type = 3) THEN 1 
            ELSE 0 END) AS abandon_count,
   SUM(CASE WHEN e.event_type = 2 AND 
                 e.visit_id IN 
                      (SELECT 
                         visit_id 
                       FROM clique_bait.events 
                       WHERE 
                         event_type = 3 ) THEN 1 
            ELSE 0 END) AS purchase_count
FROM clique_bait.events e
  JOIN clique_bait.page_hierarchy ph
    ON e.page_id = ph.page_id
WHERE
  ph.product_category IS NOT NULL
GROUP BY
  ph.product_category
ORDER BY
  ph.product_category
)
```
| product_category | view_count | cartadd_count | abandon_count | purchase_count |
| ---------------- | ---------- | ------------- | ------------- | -------------- |
| Fish             | 4633       | 2789          | 674           | 2115           |
| Luxury           | 3032       | 1870          | 466           | 1404           |
| Shellfish        | 6204       | 3792          | 894           | 2898           |

Use your 2 new output tables - answer the following questions:

**1. Which product had the most views, cart adds and purchases?**
```sql
SELECT
  product_name,
  view_count
FROM product_funnel
ORDER BY
  view_count DESC
LIMIT 1;
```
| product_name | view_count     |
| ------------ | -------------- |
| Oyster       | 1568           |
```sql
SELECT
  product_name,
  cartadd_count
FROM product_funnel
ORDER BY
  cartadd_count DESC
LIMIT 1
```
| product_name | cartadd_count  |
| ------------ | -------------- |
| Lobster      | 968            |
```sql
SELECT
  product_name,
  purchase_count
FROM product_funnel
ORDER BY
  purchase_count DESC
LIMIT 1
```
| product_name | purchase_count |
| ------------ | -------------- |
| Lobster      | 754            |

**2. Which product was most likely to be abandoned?**
```sql
SELECT
  product_name,
  abandon_count
FROM product_funnel
ORDER BY
  abandon_count DESC
LIMIT 1
```
| product_name   | abandon_count |
| -------------- | ------------- |
| Russian Caviar | 249           |

**3. Which product had the highest view to purchase percentage?**
```sql
SELECT
  product_name,
  ROUND(purchase_count / view_count::DECIMAL * 100, 1) || '%' AS purchase_conversion
FROM product_funnel
ORDER BY
  purchase_conversion DESC
LIMIT 1
```
| product_name | purchase_conversion |
| ------------ | ------------------- |
| Lobster      | 48.7%               |

**4. What is the average conversion rate from view to cart add?**
```sql
SELECT
  ROUND((SUM(cartadd_count) / SUM(view_count)) * 100, 1) || '%'  AS avg_cartadd_conversion
FROM product_funnel
```
| avg_cartadd_conversion |
| ---------------------- |
| 60.9%                  |

**5. What is the average conversion rate from cart add to purchase?**
```sql
SELECT
  ROUND((SUM(purchase_count) / SUM(cartadd_count)) * 100, 1) || '%'  AS avg_purchase_conversion
FROM product_funnel
```
| avg_purchase_conversion |
| ----------------------- |
| 75.9%                   |

## D. Campaigns Analysis
Generate a table that has 1 single row for every unique visit_id record and has the following columns:
* `user_id`
* `visit_id`
* `visit_start_time`: the earliest event_time for each visit
* `page_views`: count of page views for each visit
* `cart_adds`: count of product cart add events for each visit
* `purchase`: 1/0 flag if a purchase event exists for each visit
* `campaign_name`: map the visit to a campaign if the visit_start_time falls between the start_date and end_date
* `impression`: count of ad impressions for each visit
* `click`: count of ad clicks for each visit
* (Optional column) `cart_products`: a comma separated text value with products added to the cart sorted by the order they were added to the cart (hint: use the sequence_number)

Use the subsequent dataset to generate at least 5 insights for the Clique Bait team - bonus: prepare a single A4 infographic that the team can use for their management reporting sessions, be sure to emphasise the most important points from your findings.

Some ideas you might want to investigate further include:
* Identifying users who have received impressions during each campaign period and comparing each metric with other users who did not have an impression event  
* Does clicking on an impression lead to higher purchase rates?  
* What is the uplift in purchase rate when comparing users who click on a campaign impression versus users who do not receive an impression? What if we compare them with users who just an impression but do not click?  
* What metrics can you use to quantify the success or failure of each campaign compared to eachother?
