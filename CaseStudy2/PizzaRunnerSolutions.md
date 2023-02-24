## Case Study #2 - Pizza Runner

### Data Cleaning
In addition to standard cleaning (replacing blank spaces with nulls, removing characters and converting distance and durations into integers, etc), I also replaced 2020 with 2021 in the `order_time` and `pickup_time` columns in `customer_orders` and `runner_orders` tables respectively. When reading through the description, comparing the tables within the database, and viewing the sample tables in the Case Study's webpage, Pizza Runner is clearly shown to begin operations at the start of 2021. However, the insert statements input these dates as 2020. In practice, verification by the company owner or manager to confirm the correct dates would occur but, for this case study, it is assumed to be a typo (hey, we all do it at least once when January rolls around!) and thus was fixed in our cleaned tables.

`runner_orders`
```sql
DROP TABLE IF EXISTS runner_orders_clean;
CREATE TEMP TABLE runner_orders_clean AS 
  (SELECT
      order_id,
      runner_id,
      (CASE WHEN pickup_time ILIKE 'null'
              OR pickup_time LIKE '' THEN NULL
            WHEN pickup_time LIKE '2020%' THEN REPLACE(pickup_time, '2020', '2021')
            ELSE pickup_time
             END)::TIMESTAMP 
         AS pickup_time,
      (CASE WHEN distance ILIKE 'null'
              OR distance LIKE '' THEN NULL
            ELSE REGEXP_REPLACE(distance, '[a-z]+', '', 'g')
             END)::DECIMAL 
         AS distance,  
      (CASE WHEN duration ILIKE 'null'
              OR duration LIKE '' THEN NULL
            ELSE REGEXP_REPLACE(duration, '[a-z]+', '', 'g')
             END)::INTEGER 
         AS duration,
      CASE WHEN cancellation ILIKE 'null'
             OR cancellation LIKE '' THEN NULL
           ELSE cancellation
            END cancellation
   FROM pizza_runner.runner_orders
  )
```
`customer_orders`
```sql
DROP TABLE IF EXISTS customer_orders_clean;
CREATE TEMP TABLE customer_orders_clean AS 
  (SELECT
      order_id,
      customer_id,
      pizza_id,
      CASE WHEN exclusions ILIKE 'null'
             OR exclusions LIKE '' THEN NULL
           ELSE exclusions
            END exclusions,
      CASE WHEN extras ILIKE 'null'
             OR extras LIKE '' THEN NULL
           ELSE extras
            END extras,
      order_time + '1 year' AS clean_order_time
  FROM pizza_runner.customer_orders
  )
```

### A. Pizza Metrics

**1. How many pizzas were ordered?**
```SQL
SELECT
  COUNT(order_id) AS total_pizzas
FROM customer_orders_clean
```
| total_pizzas |
| ------------ |
| 14           |

**2. How many unique customer orders were made?**
```sql
SELECT
  COUNT(DISTINCT order_id) AS unique_orders
FROM customer_orders_clean
```
| unique_orders |
| ------------- |
| 10            |

**3. How many successful orders were delivered by each runner?**
```sql
SELECT
  runner_id,
  COUNT(order_id) AS delivered_total
FROM runner_orders_clean
WHERE
  distance IS NOT NULL
GROUP BY
  runner_id
ORDER BY
  runner_id
```
runner_id | delivered_total
--------- | ---------------
1         | 4
2         | 3
3         | 1

**4. How many of each type of pizza was delivered?**
```sql
WITH delivered_pizzas AS
  (SELECT
    order_id
  FROM runner_orders_clean
  WHERE
    distance IS NOT NULL
  )
SELECT
  pn.pizza_name,
  COUNT(dp.order_id) AS type_delivered
FROM customer_orders_clean coc
  JOIN pizza_runner.pizza_names pn
    ON coc.pizza_id = pn.pizza_id
  JOIN delivered_pizzas dp
    ON dp.order_id = coc.order_id
GROUP BY
  pn.pizza_name
```
pizza_name | type_delivered
---------- | --------------
Meatlovers | 9
Vegetarian | 3


**5. How many Vegetarian and Meatlovers were ordered by each customer?**
```sql
SELECT
  coc.customer_id,
  COUNT(CASE WHEN pn.pizza_name = 'Meatlovers' THEN 1
             ELSE NULL
              END) 
     AS meatlovers,
  COUNT(CASE WHEN pn.pizza_name = 'Vegetarian' THEN 1
             ELSE NULL
              END) 
     AS vegetarian
FROM customer_orders_clean coc
  JOIN pizza_runner.pizza_names pn
    ON coc.pizza_id = pn.pizza_id
GROUP BY
  coc.customer_id
ORDER BY
  coc.customer_id
```
customer_id | meatlovers | vegetarian
----------- | ---------- | ----------
101         | 2          | 1
102         | 2          | 1
103         | 3          | 1
104         | 3          | 0
105         | 0          | 1


**6. What was the maximum number of pizzas delivered in a single order?**
```sql
WITH pizza_count AS
  (SELECT
    coc.customer_id,
    roc.order_id,
    COUNT(roc.order_id) AS num_delivered
  FROM runner_orders_clean roc
    JOIN customer_orders_clean coc
      ON roc.order_id = coc.order_id
  WHERE
    distance IS NOT NULL
  GROUP BY
    coc.customer_id,
    roc.order_id
  ORDER BY
    coc.customer_id
  )
SELECT
  MAX(num_delivered) AS biggest_order
FROM pizza_count
```
| biggest_order |
| ------------- |
| 3             |


**7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**
```sql
WITH delivered_pizzas AS
  (SELECT
    order_id
  FROM runner_orders_clean
  WHERE
    distance IS NOT NULL
  )
SELECT
  coc.customer_id,
  COUNT(CASE WHEN exclusions IS NOT NULL
               OR extras IS NOT NULL THEN 1
             ELSE NULL
              END) 
     AS changed,
  COUNT(CASE WHEN exclusions IS NULL
              AND extras IS NULL THEN 1
             ELSE NULL
              END) 
     AS original
FROM customer_orders_clean coc
  JOIN delivered_pizzas dp
    ON dp.order_id = coc.order_id
GROUP BY
  coc.customer_id
ORDER BY
  coc.customer_id
```
customer_id | changed | original
----------- | ------- | --------
101         | 0       | 2
102         | 0       | 3
103         | 3       | 0
104         | 2       | 1
105         | 1       | 0

**8. How many pizzas were delivered that had both exclusions and extras?**
```sql
WITH delivered_pizzas AS
  (SELECT
    order_id
  FROM runner_orders_clean
  WHERE
    distance IS NOT NULL
  )
SELECT
  COUNT(CASE WHEN exclusions IS NOT NULL
              AND extras IS NOT NULL THEN 1
             ELSE NULL
              END) 
     AS fully_changed_pizzas
FROM customer_orders_clean coc
  JOIN delivered_pizzas dp
    ON dp.order_id = coc.order_id
```
| fully_changed_pizzas |
| -------------------- |
| 1                    |

**9. What was the total volume of pizzas ordered for each hour of the day?**
```sql
SELECT
  EXTRACT(hour FROM clean_order_time) as order_hour,
  COUNT(order_id) AS num_ordered
FROM customer_orders_clean
GROUP BY
  order_hour
ORDER BY
  order_hour
```
order_hour | num_ordered
---------- | -----------
11         | 1
13         | 3
18         | 3
19         | 1
21         | 3
23         | 3


**10. What was the volume of orders for each day of the week?**
```sql
SELECT
  TO_CHAR(clean_order_time, 'Day') as weekday_ordered,
  COUNT(order_id) AS num_by_weekday
FROM customer_orders_clean
GROUP BY
  weekday_ordered
ORDER BY
  num_by_weekday DESC
```
weekday_ordered | num_by_weekday
--------------- | --------------
Monday          | 5
Friday          | 5
Saturday        | 3
Sunday          | 1

### B. Runner and Customer Experience

**1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)**
```sql
SELECT
  CASE WHEN (EXTRACT('week' FROM registration_date)) = 53 THEN 1
       ELSE (EXTRACT('week' FROM registration_date) + 1)
        END registration_week,
  COUNT(registration_date) AS runner_signups
FROM pizza_runner.runners
GROUP BY
  registration_week
ORDER BY
  registration_week
```
registration_week | runner_signups
----------------- | --------------
1                 | 2
2                 | 1
3                 | 1

**2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**
```sql
WITH pickup_diff AS
  (SELECT DISTINCT
    roc.runner_id,
    coc.clean_order_time::time AS order_time,
    roc.pickup_time::time AS pickup_time,
    DATE_TRUNC('minute', AGE(roc.pickup_time, coc.clean_order_time) 
                + interval '30 seconds') 
       AS pickup_diff_minutes
  FROM runner_orders_clean roc
    JOIN customer_orders_clean coc
      ON roc.order_id = coc.order_id
  WHERE
    distance IS NOT NULL
  )
SELECT
  runner_id,
  DATE_TRUNC('minute', AVG(pickup_diff_minutes) + interval '30 seconds') AS avg_pickup_wait --round to nearest minute
FROM pickup_diff
GROUP BY
  runner_id
ORDER BY
  runner_id
```
 runner_id | avg_pickup_wait
 --------- | ---------------
 1         | 15 minutes
 2         | 20 minutes
 3         | 10 minutes

**3. Is there any relationship between the number of pizzas and how long the order takes to prepare?**
```sql
WITH prep_times AS
  (SELECT
    roc.order_id,
    COUNT(roc.order_id) AS pizzas_ordered,
    coc.clean_order_time::TIME,
    roc.pickup_time::TIME,
    AGE(roc.pickup_time, coc.clean_order_time) AS prep_time
  FROM runner_orders_clean roc
    JOIN customer_orders_clean coc
      ON roc.order_id = coc.order_id
  WHERE
    roc.distance IS NOT NULL
  GROUP BY
    roc.order_id,
    coc.clean_order_time::time,
    roc.pickup_time::time,
    prep_time
  )
SELECT
  pizzas_ordered,
  DATE_TRUNC('minutes', AVG(prep_time)) AS avg_prep_time
FROM prep_times
GROUP BY
  pizzas_ordered
ORDER BY
  pizzas_ordered
```
pizzas_ordered | avg_prep_time
-------------- | ---------------
1              | 12 minutes
2              | 18 minutes
3              | 29 minutes

Looking at only the above output, one may conclude that more pizzas per order result in longer prep time. However, we should then further investigate the amount of pizzas per order.

```sql
SELECT
  pizzas_ordered,
  COUNT(pizzas_ordered) AS num_of_orders
FROM
  (SELECT
    COUNT(roc.order_id) AS pizzas_ordered
  FROM runner_orders_clean roc
    JOIN customer_orders_clean coc
      ON roc.order_id = coc.order_id
  WHERE
    roc.distance IS NOT NULL
  GROUP BY
    roc.order_id
  ) op
GROUP BY
  pizzas_ordered
```
pizzas_ordered | num_of_orders
-------------- | -------------
3              | 1
2              | 2
1              | 5

As we can see with the table above, only one order was placed with 3 pizzas. It is therefore impossible to conclude whether a relationship exists between pizza total and prep time or if that particular 3-pizza order suffered from the runner taking longer to arrive at HQ. More data is necessary before any relationship correlations can be determined.

**4. What was the average distance traveled for each customer?**
```sql
WITH customer_distance AS
  (SELECT DISTINCT
    roc.order_id,
    coc.customer_id,
    roc.distance
  FROM runner_orders_clean roc
    JOIN customer_orders_clean coc
      ON roc.order_id = coc.order_id
  WHERE
    distance IS NOT NULL
  )
SELECT
  customer_id,
  ROUND(AVG(distance), 1) AS avg_distance
FROM customer_distance
GROUP BY
  customer_id
ORDER BY
  customer_id
```
customer_id | avg_distance
----------- | ------------
101         | 20
102         | 18.4
103         | 23.4
104         | 10
105         | 25


**5. What was the difference between the longest and shortest delivery times for all orders?**
```sql
SELECT
  MAX(duration) - MIN(duration) || ' minutes' AS duration_diff
FROM runner_orders_clean roc
```
| duration_diff |
| ------------- |
| 30 minutes    |

**6. What was the average speed for each runner for each delivery and do you notice any trend for these values?**
```sql
SELECT
  roc.runner_id,
  COUNT(roc.order_id) AS pizzas_ordered,
  ROUND((roc.distance/roc.duration) * 60) || ' kph' AS speed
FROM runner_orders_clean roc
  JOIN customer_orders_clean coc
    ON roc.order_id = coc.order_id
WHERE
  roc.distance IS NOT NULL
GROUP BY
  roc.runner_id,
  speed
ORDER BY
  roc.runner_id,
  pizzas_ordered DESC,
  speed DESC
```
 runner_id | pizzas_ordered | speed  
 --------- | -------------- | ------ 
 1         | 2              | 60 kph 
 1         | 2              | 40 kph 
 1         | 1              | 44 kph 
 1         | 1              | 38 kph 
 2         | 3              | 35 kph 
 2         | 1              | 94 kph 
 2         | 1              | 60 kph 
 3         | 1              | 40 kph 

Again, looking just at the above table, there does not appear to be any discernible trends in speed related to either `runner_id` or the number of pizzas in a delivery. Let's take this one step further and find the average speed for orders, grouped by the number of pizzas in the delivery.
```sql
WITH runner_speed AS
  (SELECT
    roc.runner_id,
    COUNT(roc.order_id) AS pizzas_ordered,
    ROUND((roc.distance/roc.duration) * 60) AS speed
  FROM runner_orders_clean roc
    JOIN customer_orders_clean coc
      ON roc.order_id = coc.order_id
  WHERE
    roc.distance IS NOT NULL
  GROUP BY
    roc.runner_id,
    speed
  )
SELECT
  pizzas_ordered,
  ROUND(AVG(speed)) || ' kph' AS avg_speed
FROM runner_speed
GROUP BY
  pizzas_ordered
```
 pizzas_ordered | avg_speed 
 -------------- | --------- 
 3              | 35 kph    
 2              | 50 kph    
 1              | 55 kph    

Here we'll find an interesting detail: while the speed for orders with either 1 or 2 pizzas in the delivery are similar, orders with 3 pizzas have a speed of nearly half! However, we need to remember that there was only one instance where a customer placed an order with 3 pizzas so, again, we cannot draw any conclusions based on what may be an outlier for that specific order especially as, if we go back to the first table, we see that runner 2 has wildly different speeds (60 kmh and 94(!)kmh) between two separate orders with only 1 pizza each.

**7. What is the successful delivery percentage for each runner?**
```sql
SELECT
  runner_id,
  (SUM(CASE WHEN pickup_time IS NULL THEN 0
            ELSE 1 
            END)::FLOAT / 
       COUNT(order_id)) * 100 
     AS success_percentage
FROM runner_orders_clean
GROUP BY
  runner_id
ORDER BY
  runner_id
```
runner_id | success_percentage
--------- | ------------------
1         | 100
2         | 75
3         | 50

### C. Ingredient Optimisation

**1. What are the standard ingredients for each pizza?**
```sql
SELECT
  recipes.pizza_name,
  STRING_AGG(ingredients.topping_name, ', ') AS toppings
FROM
  (SELECT
    pn.pizza_name,
    UNNEST(STRING_TO_ARRAY(pr.toppings, ','))::INTEGER AS toppings
  FROM pizza_runner.pizza_names pn
    JOIN pizza_runner.pizza_recipes pr
      ON pn.pizza_id = pr.pizza_id
  ) recipes
JOIN
  (SELECT
    topping_id,
    topping_name
  FROM pizza_runner.pizza_toppings
  ) ingredients
ON recipes.toppings = ingredients.topping_id
GROUP BY
  recipes.pizza_name
ORDER BY
  recipes.pizza_name
```
pizza_name | toppings
---------- | ---------------------------------------------------------------------
Meatlovers | Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
Vegetarian | Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce

**2. What was the most commonly added extra?**
```sql
SELECT
  ingredients.topping_name,
  COUNT(extras.topping_extras) AS extras_ordered
FROM
  (SELECT
    UNNEST(STRING_TO_ARRAY(extras, ','))::INTEGER AS topping_extras
  FROM customer_orders_clean
  WHERE 
    extras IS NOT NULL
  ) extras
JOIN
  (SELECT
    topping_id,
    topping_name
  FROM pizza_runner.pizza_toppings
  ) ingredients
ON extras.topping_extras = ingredients.topping_id
GROUP BY
  ingredients.topping_name
ORDER BY
  extras_ordered DESC
LIMIT 1
```
topping_name | extras_ordered
------------ | --------------
Bacon        | 4

**3. What was the most common exclusion?**
```sql
SELECT
  ingredients.topping_name,
  COUNT(exclusions.topping_exclusions) AS num_exclusions
FROM
  (SELECT
    UNNEST(STRING_TO_ARRAY(exclusions, ','))::INTEGER AS topping_exclusions
  FROM customer_orders_clean
  WHERE 
    exclusions IS NOT NULL
  ) exclusions
JOIN
  (SELECT
    topping_id,
    topping_name
  FROM pizza_runner.pizza_toppings
  ) ingredients
ON exclusions.topping_exclusions = ingredients.topping_id
GROUP BY
  ingredients.topping_name
ORDER BY
  num_exclusions DESC
LIMIT 1
```
topping_name | num_exclusions
------------ | --------------
Cheese       | 4

**4. Generate an order item for each record in the customers_orders table in the format of one of the following:**
* Meat Lovers
* Meat Lovers - Exclude Beef
* Meat Lovers - Extra Bacon
* Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
```sql
SELECT
  pizza_number,
  order_id,
  CONCAT (pizza_name, 
          CASE WHEN COUNT(exclusions) > 0 THEN ' - Exclude '
            	 ELSE '' 
                END,
            STRING_AGG(exclusions, ', '), 
          CASE WHEN COUNT (extras) > 0 THEN ' - Extra '
            	 ELSE '' 
                END,
          STRING_AGG(extras, ', ')) AS pizza_specifics
FROM
  (SELECT
    orders.pizza_number,
    orders.order_id,
    orders.pizza_name,
    CASE WHEN orders.extras IS NOT NULL 
          AND ingredients.topping_id IN 
              (SELECT UNNEST(STRING_TO_ARRAY(orders.extras, ','))::INTEGER) 
         THEN ingredients.topping_name 
          END 
      AS extras,
    CASE WHEN orders.exclusions IS NOT NULL
          AND ingredients.topping_id IN
              (SELECT UNNEST(STRING_TO_ARRAY(orders.exclusions, ','))::INTEGER)
         THEN ingredients.topping_name
          END 
      AS exclusions
   FROM  
    (SELECT
        ROW_NUMBER() 
          OVER (PARTITION BY coc.order_id) AS pizza_number,
        coc.order_id,
        pn.pizza_id,
        pn.pizza_name,
        coc.extras,
        coc.exclusions
     FROM customer_orders_clean coc
       JOIN pizza_runner.pizza_names pn
         ON coc.pizza_id = pn.pizza_id
     ORDER BY 
        coc.order_id
    ) orders
  JOIN
    (SELECT
        pr.pizza_id,
        pt.topping_id,
        pt.topping_name
     FROM pizza_runner.pizza_toppings pt,
     	  pizza_runner.pizza_recipes pr
    ) ingredients
  ON orders.pizza_id = ingredients.pizza_id
) order_specifics
GROUP BY
  pizza_number,
  order_id,
  pizza_name
ORDER BY
  order_id
  pizza_number
```
pizza_number | order_id | pizza_specifics
------------ | -------- | ---------------------------------------------------------------
1            | 1        | Meatlovers
1            | 2        | Meatlovers
1            | 3        | Vegetarian
2            | 3        | Meatlovers
1            | 4        | Meatlovers - Exclude Cheese
2            | 4        | Meatlovers - Exclude Cheese
3            | 4        | Vegetarian - Exclude Cheese
1            | 5        | Meatlovers - Extra Bacon
1            | 6        | Vegetarian
1            | 7        | Vegetarian - Extra Bacon
1            | 8        | Meatlovers
1            | 9        | Meatlovers - Exclude Cheese - Extra Bacon, Chicken
1            | 10       | Meatlovers
2            | 10       | Meatlovers - Exclude BBQ Sauce, Mushrooms - Extra Bacon, Cheese


**5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients**
* For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
```sql
WITH order_ingredients AS
  (SELECT
    pizza_number,
    order_id,
    pizza_name,
    CONCAT(CASE WHEN toppings = extras THEN '2x'
                ELSE NULL
                 END,
           CASE WHEN extras IS NOT NULL THEN extras
                WHEN toppings = exclusions THEN NULL
                ELSE toppings
                 END) toppings
  FROM
    (SELECT
      orders.pizza_number,
      orders.order_id,
      orders.pizza_name,
      CASE WHEN ingredients.topping_id IN
               (SELECT UNNEST(STRING_TO_ARRAY(ingredients.toppings, ','))::INTEGER)
           THEN ingredients.topping_name
            END AS toppings,
      CASE WHEN orders.extras IS NOT NULL
            AND ingredients.topping_id IN
               (SELECT UNNEST(STRING_TO_ARRAY(orders.extras, ','))::INTEGER)
           THEN ingredients.topping_name
            END AS extras,
      CASE WHEN orders.exclusions IS NOT NULL
            AND ingredients.topping_id IN
               (SELECT UNNEST(STRING_TO_ARRAY(orders.exclusions, ','))::INTEGER)
           THEN ingredients.topping_name
            END AS exclusions
    FROM
      (SELECT
        ROW_NUMBER()
          OVER (PARTITION BY coc.order_id) AS pizza_number,
        coc.order_id,
        pn.pizza_id,
        pn.pizza_name,
        coc.extras,
        coc.exclusions
      FROM customer_orders_clean coc
        JOIN pizza_runner.pizza_names pn
          ON coc.pizza_id = pn.pizza_id
      ORDER BY
        coc.order_id
      ) orders
  JOIN
    (SELECT
      pr.pizza_id,
      pr.toppings,
      pt.topping_id,
      pt.topping_name
    FROM pizza_runner.pizza_toppings pt,
         pizza_runner.pizza_recipes pr
    ) ingredients
  ON orders.pizza_id = ingredients.pizza_id
    ) order_specifics
  )
SELECT
  pizza_number,
  order_id,
  CONCAT(pizza_name, ': ',
         STRING_AGG(NULLIF(toppings, ''), ', ')
        ) AS order_ingredient_list
FROM order_ingredients
GROUP BY
  order_id,
  pizza_number,
  pizza_name
ORDER BY
  order_id,
  pizza_number
```
pizza_number | order_id | order_ingredient_list
------------ | -------- | -----------------------------------------------------------------------------------
1            | 1        | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
1            | 2        | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
1            | 3        | Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce
2            | 3        | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
1            | 4        | Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami
2            | 4        | Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami
3            | 4        | Vegetarian: Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce
1            | 5        | Meatlovers: 2xBacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
1            | 6        | Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce
1            | 7        | Vegetarian: Bacon, Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce
1            | 8        | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
1            | 9        | Meatlovers: 2xBacon, BBQ Sauce, Beef, 2xChicken, Mushrooms, Pepperoni, Salami
1            | 10       | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
2            | 10       | Meatlovers: 2xBacon, Beef, 2xCheese, Chicken, Pepperoni, Salami

**6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?**
```sql
SELECT
  topping_name,
  COUNT(CASE WHEN toppings = exclusions THEN NULL
             ELSE toppings
              END) + 
        COUNT(extras) 
     AS ingredient_totals
FROM
  (SELECT
      orders.pizza_number,
      orders.order_id,
      ingredients.topping_name,
      CASE WHEN ingredients.topping_id IN
              (SELECT UNNEST(STRING_TO_ARRAY(ingredients.toppings, ','))::INTEGER)
           THEN ingredients.topping_name
            END AS toppings,
      CASE WHEN orders.extras IS NOT NULL
            AND ingredients.topping_id IN
              (SELECT UNNEST(STRING_TO_ARRAY(orders.extras, ','))::INTEGER)
           THEN ingredients.topping_name
            END AS extras,
      CASE WHEN orders.exclusions IS NOT NULL
            AND ingredients.topping_id IN
              (SELECT UNNEST(STRING_TO_ARRAY(orders.exclusions, ','))::INTEGER)
           THEN ingredients.topping_name
            END AS exclusions
   FROM  
    (SELECT
        ROW_NUMBER() 
          OVER (PARTITION BY coc.order_id) AS pizza_number,
        coc.order_id,
     	pn.pizza_id,
        coc.extras,
        coc.exclusions
     FROM customer_orders_clean coc
       JOIN pizza_runner.pizza_names pn
         ON coc.pizza_id = pn.pizza_id
       JOIN runner_orders_clean roc
     	 ON coc.order_id = roc.order_id
     WHERE
     	roc.distance IS NOT NULL
     ORDER BY 
        coc.order_id
    ) orders
  JOIN
    (SELECT
        pr.pizza_id,
     	pr.toppings,
     	pt.topping_id,
        pt.topping_name
     FROM pizza_runner.pizza_toppings pt,
     	  pizza_runner.pizza_recipes pr
    ) ingredients
  ON orders.pizza_id = ingredients.pizza_id
  ) order_specifics
GROUP BY
  topping_name
ORDER BY
  ingredient_totals DESC
```
topping_name | ingredient_totals
------------ | -----------------
Bacon        | 12
Mushrooms    | 11
Cheese       | 10
Pepperoni    | 9
Chicken      | 9
Salami       | 9
Beef         | 9
BBQ Sauce    | 8
Tomato Sauce | 3
Onions       | 3
Tomatoes     | 3
Peppers      | 3

### D. Pricing and Ratings

**1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?**
```sql
SELECT
  SUM(CASE WHEN coc.pizza_id = 1 THEN 12
           WHEN coc.pizza_id = 2 THEN 10
           ELSE NULL
            END) 
    AS profit
FROM customer_orders_clean coc
  JOIN pizza_runner.pizza_names pn
    ON coc.pizza_id = pn.pizza_id
  JOIN runner_orders_clean roc
    ON coc.order_id = roc.order_id
WHERE 
  distance IS NOT NULL
```
| profit |
| ------ |
| 138    |

**2. What if there was an additional $1 charge for any pizza extras?**
* Add cheese is $1 extra
```sql
WITH pizza_prices AS
  (SELECT
    roc.order_id,
    pn.pizza_name,
    CASE WHEN coc.pizza_id = 1 THEN 12
         WHEN coc.pizza_id = 2 THEN 10
         ELSE NULL
          END pizza_price,
    (CARDINALITY(STRING_TO_ARRAY(coc.extras, ' ')) * 1) AS extras_price --multiply by additions charge
  FROM customer_orders_clean coc
    JOIN pizza_runner.pizza_names pn
      ON coc.pizza_id = pn.pizza_id
    JOIN runner_orders_clean roc
      ON coc.order_id = roc.order_id
  WHERE 
    distance IS NOT NULL
  )
SELECT
  SUM(pizza_price) + SUM(extras_price) AS profit_with_extras_charge
FROM pizza_prices
```
| profit_with_extras_charge |
| ------------------------- |
| 142                       |

**3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.**
```sql
DROP TABLE IF EXISTS runner_ratings;
CREATE TABLE runner_ratings 
  ("order_id" INTEGER,
  "rating" INTEGER,
  "rating_date" TIMESTAMP,
  "comments" VARCHAR(500)
  );

INSERT INTO runner_ratings
  ("order_id", "rating", "rating_date", "comments")
VALUES
  ('1', '4', '2021-01-01 19:01:35', ''),
  ('2', '5', '2021-01-01 20:30:13', 'Arrived fast and tasted delicious!'),
  ('3', '4', '2021-01-03 00:45:28', ''),
  ('4', '3', '2021-01-04 14:35:07', 'Food took forever to arrive and wasn''t hot but taste was decent. Maybe the driver was having an off day.'),
  ('5', '5', '2021-01-08 21:30:17', 'Fastest pizza delivery I''ve ever had! So fast it was still hot like it''d just come out of the oven and even burned my mouth. Worth it for this flavor!'),
  ('7', '4', '2021-01-08 22:15:47', 'Driver forgot napkins but otherwise delivery was quick and still warm.'),
  --customer did not leave rating for order_id 8
  ('10', '5', '2021-01-11 19:03:22', '');

SELECT *
FROM runner_ratings
```
order_id | rating | rating_date              | comments
-------- | ------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------
1        | 4      | 2021-01-01T19:01:35.000Z |
2        | 5      | 2021-01-01T20:30:13.000Z | Arrived fast and tasted delicious!
3        | 4      | 2021-01-03T00:45:28.000Z |
4        | 3      | 2021-01-04T14:35:07.000Z | Food took forever to arrive and wasn't hot but taste was decent. Maybe the driver was having an off day.
5        | 5      | 2021-01-08T21:30:17.000Z | Fastest pizza delivery I've ever had! So fast it was still hot like it'd just come out of the oven and even burned my mouth. Worth it for this flavor!
7        | 4      | 2021-01-08T22:15:47.000Z | Driver forgot napkins but otherwise delivery was quick and still warm.
10       | 5      | 2021-01-11T19:03:22.000Z |

**4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?**
* customer_id
* order_id
* runner_id
* rating
* order_time
* pickup_time
* Time between order and pickup
* Delivery duration
* Average speed
* Total number of pizzas
```sql
SELECT
  coc.customer_id,
  roc.order_id,
  roc.runner_id,
  rr.rating,
  coc.clean_order_time,
  roc.pickup_time,
  DATE_TRUNC('minute', AGE(roc.pickup_time, coc.clean_order_time)
              + interval '30 seconds') AS order_to_pickup,
  roc.duration,
  ROUND((roc.distance/roc.duration) * 60) || ' kph' AS speed,
  COUNT(roc.order_id) AS pizzas_ordered
FROM customer_orders_clean coc
       JOIN runner_orders_clean roc
         ON coc.order_id = roc.order_id
  LEFT JOIN runner_ratings rr  --left join to include deliveries where the customer chose not to leave a review
         ON roc.order_id = rr.order_id
WHERE
  roc.duration IS NOT NULL
GROUP BY
  coc.customer_id,
  roc.order_id,
  roc.runner_id,
  rr.rating,
  coc.clean_order_time,
  roc.pickup_time,
  roc.duration,
  speed
ORDER BY
  roc.order_id
```
customer_id | order_id | runner_id | rating | clean_order_time         | pickup_time              | order_to_pickup | duration | speed | pizzas_ordered
----------- | -------- | --------- | ------ | ------------------------ | ------------------------ | --------------- | -------- | ----- | --------------
101         | 1        | 1         | 4      | 2021-01-01T18:05:02.000Z | 2021-01-01T18:15:34.000Z | 11 minutes      | 32       | 38    | 1
101         | 2        | 1         | 5      | 2021-01-01T19:00:52.000Z | 2021-01-01T19:10:54.000Z | 10 minutes      | 27       | 44    | 1
102         | 3        | 1         | 4      | 2021-01-02T23:51:23.000Z | 2021-01-03T00:12:37.000Z | 21 minutes      | 20       | 40    | 2
103         | 4        | 2         | 3      | 2021-01-04T13:23:46.000Z | 2021-01-04T13:53:03.000Z | 29 minutes      | 40       | 35    | 3
104         | 5        | 3         | 5      | 2021-01-08T21:00:29.000Z | 2021-01-08T21:10:57.000Z | 10 minutes      | 15       | 40    | 1
105         | 7        | 2         | 4      | 2021-01-08T21:20:29.000Z | 2021-01-08T21:30:45.000Z | 10 minutes      | 25       | 60    | 1
102         | 8        | 2         |        | 2021-01-09T23:54:33.000Z | 2021-01-10T00:15:02.000Z | 20 minutes      | 15       | 94    | 1
104         | 10       | 1         | 5      | 2021-01-11T18:34:49.000Z | 2021-01-11T18:50:20.000Z | 16 minutes      | 10       | 60    | 2

**5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?**
```sql
SELECT
  SUM(profits.order_total) -
    SUM(expenses.delivery_fee) AS net_profit
FROM
  (SELECT
    coc.order_id,
    SUM(CASE WHEN coc.pizza_id = 1 THEN 12
             WHEN coc.pizza_id = 2 THEN 10
             ELSE NULL
              END) 
        AS order_total
  FROM customer_orders_clean coc
    JOIN runner_orders_clean roc
      ON coc.order_id = roc.order_id
  WHERE 
    distance IS NOT NULL
  GROUP BY
    coc.order_id
  ) profits
JOIN
  (SELECT
    roc.order_id,
    SUM(distance * 0.3) AS delivery_fee
  FROM runner_orders_clean roc
  WHERE 
    distance IS NOT NULL
  GROUP BY
    roc.order_id
  ) expenses
ON profits.order_id = expenses.order_id
```
| net_profit |
| ---------- |
| 94.44      |

### E. Bonus Question
If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?
```sql
INSERT INTO pizza_runner.pizza_names
  ("pizza_id", "pizza_name")
VALUES
  (3, 'Supreme');

INSERT INTO pizza_runner.pizza_recipes
  ("pizza_id", "toppings")
VALUES
  (3, '1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12');

SELECT
  pn.pizza_id,
  pn.pizza_name,
  pr.toppings
FROM pizza_runner.pizza_names pn
  JOIN pizza_runner.pizza_recipes pr
    ON pn.pizza_id = pr.pizza_id
```
pizza_id | pizza_name | toppings
-------- | ---------- | -------------------------------------
1        | Meatlovers | 1, 2, 3, 4, 5, 6, 8, 10
2        | Vegetarian | 4, 6, 7, 9, 11, 12
3        | Supreme    | 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12
