## Case Study #3 - Foodie-Fi

### A. Customer Journey

**1. Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.**
```sql
WITH onboarding AS
  (SELECT
    s.customer_id,
    RANK ()
      OVER(PARTITION BY s.customer_id
           ORDER BY s.plan_id) AS sub_order,
    p.plan_name
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  WHERE
    s.customer_id IN (1, 2, 11, 13, 15, 16, 18, 19)
      AND (CASE WHEN s.plan_id != 0 THEN s.start_date
                ELSE NULL
                 END) 
      IS NOT NULL
  ORDER BY
    s.customer_id
  )
SELECT
  plan_name AS post_trial_plan,
  COUNT(customer_id) AS customer_count
FROM onboarding
WHERE
  sub_order = 1
GROUP BY
  post_trial_plan
ORDER BY
  customer_count DESC,
  post_trial_plan DESC
```
post_trial_plan | customer_count
--------------- | --------------
pro monthly     | 3
basic monthly   | 3
pro annual      | 1
churn           | 1

For the chosen sample of Foodie-Fi customers, 3 customers continued into the default pro-monthly plan after their trial ended, while 3 others kept a subscription to Foodie-Fi but downgraded it to the basic plan. Additionally, 1 customer canceled at the end of their trial and 1 customer upgraded to the pro-annual plan.

### B. Data Analysis

**1. How many customers has Foodie-Fi ever had?**
```sql
SELECT
  COUNT(DISTINCT customer_id)
FROM foodie_fi.subscriptions
```
| count |
| ----- |
| 1000  |

**2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value**
```sql
SELECT
  TO_CHAR(DATE_TRUNC('MONTH', start_date)::DATE, 'yyyy-mm') AS start_month,
  COUNT(DATE_TRUNC('MONTH', start_date)) AS monthly_distribution
FROM foodie_fi.subscriptions
WHERE
  plan_id = 0
GROUP BY
  start_month
```
start_month | monthly_distribution
----------- | --------------------
2020-01     | 88
2020-02     | 68
2020-03     | 94
2020-04     | 81
2020-05     | 88
2020-06     | 79
2020-07     | 89
2020-08     | 88
2020-09     | 87
2020-10     | 79
2020-11     | 75
2020-12     | 84

**3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name.**
```sql
SELECT
  p.plan_name,
  COUNT(p.plan_id) AS event_count
FROM foodie_fi.subscriptions s
  JOIN foodie_fi.plans p
    ON s.plan_id = p.plan_id
WHERE
  s.start_date > '2020-12-31'
GROUP BY
  p.plan_name,
  p.plan_id
ORDER BY
  p.plan_id
```
plan_name     | event_count
------------- | -----------
basic monthly | 8
pro monthly   | 60
pro annual    | 63
churn         | 71

**4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**
```sql
SELECT
  SUM(CASE WHEN s.plan_id = 4 THEN 1
           ELSE 0
            END) AS churn_total,
  (SUM(CASE WHEN s.plan_id = 4 THEN 1
            ELSE 0 END) / 
      COUNT(DISTINCT s.customer_id)::FLOAT) * 100 || '%' 
    AS churn_percent
FROM foodie_fi.subscriptions s
```
churn_total | churn_percent
----------- | -------------
307         | 30.7%

**5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?**
```sql
WITH sub_order AS
  (SELECT
    customer_id,
    RANK ()
      OVER(PARTITION BY customer_id
           ORDER BY plan_id) AS plan_order,
    plan_id
  FROM foodie_fi.subscriptions
  )
SELECT
  SUM(CASE WHEN plan_order = 2 AND plan_id = 4 
           THEN 1 ELSE 0 
            END) trial_churn_total,
  ROUND((SUM(CASE WHEN plan_order = 2 AND plan_id = 4 
                  THEN 1 ELSE 0 END) / 
          SUM(CASE WHEN plan_id = 0 
                   THEN 1 ELSE 0 END)::FLOAT) * 100) || '%' 
      AS trial_churn_percentage
FROM sub_order
```
trial_churn_total | trial_churn_percentage
----------------- | ----------------------
92                | 9%

**6. What is the number and percentage of customer plans after their initial free trial?**
```sql
WITH sub_order AS
  (SELECT
    s.customer_id,
    s.plan_id,
    p.plan_name,
    RANK ()
      OVER(PARTITION BY s.customer_id
           ORDER BY s.plan_id) AS plan_order
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  )
SELECT
  plan_name,
  COUNT(plan_id) AS trial_conversion_plan_count,
  ROUND(COUNT(plan_id) /
           (SELECT 
              COUNT(distinct customer_id)
            FROM foodie_fi.subscriptions
            )::DECIMAL * 100, 1) || '%' 
      AS conversion_percentage
FROM sub_order
WHERE
  plan_order = 2
GROUP BY
  plan_name,
  plan_id
ORDER BY
  plan_id
```
plan_name     | trial_conversion_plan_count | conversion_percentage
------------- | --------------------------- | ---------------------
basic monthly | 546                         | 54.6%
pro monthly   | 325                         | 32.5%
pro annual    | 37                          | 3.7%
churn         | 92                          | 9.2%

**7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?**
```sql
WITH current_plan AS
  (SELECT
    s.customer_id,
    RANK ()
      OVER(PARTITION BY s.customer_id
           ORDER BY s.plan_id DESC) AS plan_order,
    s.plan_id,
    p.plan_name,
    s.start_date
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  WHERE 
    s.start_date <= '2020-12-31'
  )
SELECT
  plan_name,
  COUNT(plan_name) AS end_year_totals,
  ROUND(COUNT(plan_name) /
          (SELECT
              COUNT(DISTINCT customer_id)
          FROM foodie_fi.subscriptions
          )::DECIMAL * 100, 1) || '%' 
      AS plan_percentages_2020
FROM current_plan
WHERE 
  plan_order = 1
GROUP BY
  plan_name,
  plan_id
ORDER BY
  plan_id
```
plan_name     | end_year_totals | plan_percentages_2020
------------- | --------------- | ---------------------
trial         | 19              | 1.9%
basic monthly | 224             | 22.4%
pro monthly   | 326             | 32.6%
pro annual    | 195             | 19.5%
churn         | 236             | 23.6%

**8. How many customers have upgraded to an annual plan in 2020?**
```sql
WITH current_plan AS
  (SELECT
    s.customer_id,
    RANK ()
      OVER(PARTITION BY s.customer_id
           ORDER BY s.plan_id DESC) AS plan_order,
    s.plan_id,
    p.plan_name,
    s.start_date
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  WHERE
    s.start_date <= '2020-12-31'
  )
SELECT
  COUNT(plan_name) AS annual_plans_2020
FROM current_plan
WHERE 
  plan_order = 1 AND
  plan_id = 3
```
| annual_plans_2020 |
| ----------------- |
| 195               |

**9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?**
```sql
SELECT
  ROUND(AVG(sa.start_date - ts.start_date)) AS avg_days_to_annual
FROM
  (SELECT
    s.customer_id,
    s.start_date
  FROM foodie_fi.subscriptions s
  WHERE
    s.plan_id = 0
  ) ts -- trial start dates
JOIN
  (SELECT
    s.customer_id,
    s.start_date
  FROM foodie_fi.subscriptions s
  WHERE
    s.plan_id = 3
  ) sa -- annual start dates
ON ts.customer_id = sa.customer_id
```
| avg_days_to_annual |
| ------------------ |
| 105                |

**10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)**
```sql
WITH annual_upgrade_avgs AS
  (SELECT
    (CASE WHEN sa.start_date - ts.start_date <= 30 THEN '1'
          WHEN sa.start_date - ts.start_date BETWEEN 31 AND 60 THEN '2'
          WHEN sa.start_date - ts.start_date BETWEEN 61 AND 90 THEN '3'
          WHEN sa.start_date - ts.start_date BETWEEN 91 AND 120 THEN '4'
          WHEN sa.start_date - ts.start_date BETWEEN 121 AND 150 THEN '5'
          WHEN sa.start_date - ts.start_date BETWEEN 151 AND 180 THEN '6'
          WHEN sa.start_date - ts.start_date BETWEEN 181 AND 210 THEN '7'
          WHEN sa.start_date - ts.start_date BETWEEN 211 AND 240 THEN '8'
          WHEN sa.start_date - ts.start_date BETWEEN 241 AND 270 THEN '9'
          WHEN sa.start_date - ts.start_date BETWEEN 271 AND 300 THEN '10'
          WHEN sa.start_date - ts.start_date BETWEEN 301 AND 330 THEN '11'
          WHEN sa.start_date - ts.start_date BETWEEN 331 AND 360 THEN '12'
          WHEN sa.start_date - ts.start_date > 360 THEN '13'
           END::numeric) 
        AS upgrade_time_order,
      CASE WHEN sa.start_date - ts.start_date <= 30 THEN '0-30 days'
           WHEN sa.start_date - ts.start_date BETWEEN 31 AND 60 THEN '31-60 days'
           WHEN sa.start_date - ts.start_date BETWEEN 61 AND 90 THEN '61-90 days'
           WHEN sa.start_date - ts.start_date BETWEEN 91 AND 120 THEN '91-120 days'
           WHEN sa.start_date - ts.start_date BETWEEN 121 AND 150 THEN '121-150 days'
           WHEN sa.start_date - ts.start_date BETWEEN 151 AND 180 THEN '151-180 days'  
           WHEN sa.start_date - ts.start_date BETWEEN 181 AND 210 THEN '181-210 days'
           WHEN sa.start_date - ts.start_date BETWEEN 211 AND 240 THEN '211-240 days'
           WHEN sa.start_date - ts.start_date BETWEEN 241 AND 270 THEN '241-270 days'
           WHEN sa.start_date - ts.start_date BETWEEN 271 AND 300 THEN '271-300 days'
           WHEN sa.start_date - ts.start_date BETWEEN 301 AND 330 THEN '301-330 days'
           WHEN sa.start_date - ts.start_date BETWEEN 331 AND 360 THEN '331-360 days'
           WHEN sa.start_date - ts.start_date > 360 THEN '360+ days'
            END 
        AS upgrade_time_chunks,
      COUNT(sa.customer_id) AS annual_subscribers,
      ROUND(AVG(sa.start_date - ts.start_date)) AS avg_days_to_annual
  FROM
    (SELECT
      s.customer_id,
      s.start_date
    FROM foodie_fi.subscriptions s
    WHERE
      s.plan_id = 0
    ) ts -- trial start dates
  JOIN
    (SELECT
      s.customer_id,
      s.start_date
    FROM foodie_fi.subscriptions s
    WHERE
      s.plan_id = 3
    ) sa -- annual start dates
  ON ts.customer_id = sa.customer_id
  GROUP BY
    upgrade_time_chunks,
    upgrade_time_order
  ORDER BY
    upgrade_time_order
  )
SELECT
  upgrade_time_chunks,
  annual_subscribers,
  avg_days_to_annual
FROM annual_upgrade_avgs
```
upgrade_time_chunks | annual_subscribers | avg_days_to_annual
------------------- | ------------------ | ------------------
0-30 days           | 49                 | 10
31-60 days          | 24                 | 42
61-90 days          | 34                 | 71
91-120 days         | 35                 | 101
121-150 days        | 42                 | 133
151-180 days        | 36                 | 162
181-210 days        | 26                 | 191
211-240 days        | 4                  | 224
241-270 days        | 5                  | 257
271-300 days        | 1                  | 285
301-330 days        | 1                  | 327
331-360 days        | 1                  | 346

**11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**
```sql
WITH pro_downgrade AS
  (SELECT
    s.customer_id,
    s.plan_id,
    p.plan_name,
    RANK()
      OVER(PARTITION BY customer_id
           ORDER BY start_date),
    s.start_date
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  WHERE 
    s.start_date <= '2020-12-31' AND 
    s.plan_id IN (1, 2)
  ORDER BY
    s.customer_id
  )
SELECT
  COUNT(plan1.customer_id) AS number_of_downgrades
FROM
  (SELECT
    customer_id,
    CASE WHEN rank = 1 THEN plan_name
          END AS first_plan
  FROM pro_downgrade
  WHERE
    CASE WHEN rank = 1 THEN plan_name
          END IS NOT NULL
  ) plan1
JOIN
  (SELECT
    customer_id,
    CASE WHEN rank = 2 THEN plan_name
          END AS second_plan
  FROM pro_downgrade
  WHERE
    CASE WHEN rank = 2 THEN plan_name
          END IS NOT NULL
  ) plan2
ON plan1.customer_id = plan2.customer_id
WHERE
  second_plan = 'basic monthly'
```
| number_of_downgrades |
| -------------------- |
| 0                    |

### C. Challenge Payment Question
The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:
* monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
* upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
* upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period
* once a customer churns they will no longer make payments
```sql
CREATE TABLE IF NOT EXISTS payments_2020 AS
WITH payment_info AS
  (SELECT
    s.customer_id,
    s.plan_id,
    p.plan_name,
    (GENERATE_SERIES(s.start_date,  
                     CASE WHEN s.plan_id = 3 THEN s.start_date   
                          WHEN LEAD(s.start_date)  
                                OVER (PARTITION BY s.customer_id  
                                      ORDER BY s.start_date)  
                              IS NOT NULL THEN LEAD(s.start_date)  
                                               OVER (PARTITION BY s.customer_id  
                                                     ORDER BY s.start_date)  
                          ELSE '2020-12-31'  
                           END,  
                     INTERVAL '1 MONTH' + '1 SECOND')::DATE) AS payment_date,
    p.price AS payment_amount
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  WHERE
    s.plan_id != 0 AND
    s.plan_id != 4 AND
    s.start_date < '2021-01-01'
  ORDER BY
    s.customer_id
  )
SELECT
  customer_id,
  plan_name,
  TO_CHAR(payment_date::DATE, 'YYYY-MM-DD') AS payment_date,
  CASE WHEN plan_id - LAG(plan_id)
                       OVER (PARTITION BY customer_id
                             ORDER BY plan_id)  
            != 0 
        AND payment_date - LAG(payment_date)
                            OVER (PARTITION BY customer_id
                                  ORDER BY payment_date)
            <30
       THEN payment_amount - LAG(payment_amount)
                              OVER(PARTITION BY customer_id
                                   ORDER BY payment_amount)
       ELSE payment_amount
       END 
    AS payment_amount
FROM payment_info
```
*(sample only):*
customer_id | plan_name     | payment_date | payment_amount
----------- | ------------- | ------------ | --------------
1           | basic monthly | 2020-08-08   | 9.90
1           | basic monthly | 2020-09-08   | 9.90
1           | basic monthly | 2020-10-08   | 9.90
1           | basic monthly | 2020-11-08   | 9.90
1           | basic monthly | 2020-12-08   | 9.90
7           | basic monthly | 2020-02-12   | 9.90
7           | basic monthly | 2020-03-12   | 9.90
7           | basic monthly | 2020-04-12   | 9.90
7           | basic monthly | 2020-05-12   | 9.90
7           | pro monthly   | 2020-05-22   | 10.00
7           | pro monthly   | 2020-06-22   | 19.90
7           | pro monthly   | 2020-07-22   | 19.90
7           | pro monthly   | 2020-08-22   | 19.90
7           | pro monthly   | 2020-09-22   | 19.90
7           | pro monthly   | 2020-10-22   | 19.90
7           | pro monthly   | 2020-11-22   | 19.90
7           | pro monthly   | 2020-12-22   | 19.90
8           | basic monthly | 2020-06-18   | 9.90
8           | basic monthly | 2020-07-18   | 9.90
8           | pro monthly   | 2020-08-03   | 10.00
8           | pro monthly   | 2020-09-03   | 19.90
8           | pro monthly   | 2020-10-03   | 19.90
8           | pro monthly   | 2020-11-03   | 19.90
8           | pro monthly   | 2020-12-03   | 19.90
13          | basic monthly | 2020-12-22   | 9.90
16          | basic monthly | 2020-06-07   | 9.90
16          | basic monthly | 2020-07-07   | 9.90
16          | basic monthly | 2020-08-07   | 9.90
16          | basic monthly | 2020-09-07   | 9.90
16          | basic monthly | 2020-10-07   | 9.90
16          | pro annual    | 2020-10-21   | 189.10
17          | basic monthly | 2020-08-03   | 9.90
17          | basic monthly | 2020-09-03   | 9.90
17          | basic monthly | 2020-10-03   | 9.90
17          | basic monthly | 2020-11-03   | 9.90
17          | basic monthly | 2020-12-03   | 9.90
17          | pro annual    | 2020-12-11   | 189.10
19          | pro monthly   | 2020-06-29   | 19.90
19          | pro monthly   | 2020-07-29   | 19.90
19          | pro annual    | 2020-08-29   | 199.00
21          | basic monthly | 2020-02-11   | 9.90
21          | basic monthly | 2020-03-11   | 9.90
21          | basic monthly | 2020-04-11   | 9.90
21          | basic monthly | 2020-05-11   | 9.90
21          | pro monthly   | 2020-06-03   | 10.00
21          | pro monthly   | 2020-07-03   | 19.90
21          | pro monthly   | 2020-08-03   | 19.90
21          | pro monthly   | 2020-09-03   | 19.90
21          | pro monthly   | 2020-10-03   | 19.90
21          | pro monthly   | 2020-11-03   | 19.90
21          | pro monthly   | 2020-12-03   | 19.90

### D. Outside The Box Questions
**1. How would you calculate the rate of growth for Foodie-Fi?**  
If Foodie-Fi wanted to determine how many new customers they convert each month after the customer's trial period, the following query can be run to determine growth month-over-month for paying customer conversions:
```sql
WITH start_month AS
  (SELECT
    customer_id,
    TO_CHAR(DATE_TRUNC('MONTH', start_date), 'YYYY-MM') AS month,
    RANK()  
      OVER(PARTITION BY customer_id  
           ORDER BY start_date) AS plan_order
  FROM foodie_fi.subscriptions
  WHERE
    plan_id != 0 AND
    plan_id != 4
  ORDER BY
    month
  )
SELECT
  month,
  new_customer_count,
  ROUND(((new_customer_count - prev_month_customers) / 
            prev_month_customers::decimal)*100, 1) || '%' 
    AS growth
FROM
  (SELECT
    month,
    COUNT(customer_id) AS new_customer_count,
    LAG(COUNT(customer_id))
      OVER (ORDER BY month) AS prev_month_customers
  FROM start_month
  WHERE
    plan_order = 1
  GROUP BY
    month
  ORDER BY
    month
  ) customer_counts
```
| month   | new_customer_count | growth |
| ------- | ------------------ | ------ |
| 2020-01 | 61                 |        |
| 2020-02 | 69                 | 13.1%  |
| 2020-03 | 83                 | 20.3%  |
| 2020-04 | 70                 | -15.7% |
| 2020-05 | 80                 | 14.3%  |
| 2020-06 | 77                 | -3.8%  |
| 2020-07 | 71                 | -7.8%  |
| 2020-08 | 86                 | 21.1%  |
| 2020-09 | 77                 | -10.5% |
| 2020-10 | 78                 | 1.3%   |
| 2020-11 | 63                 | -19.2% |
| 2020-12 | 76                 | 20.6%  |
| 2021-01 | 17                 | -77.6% |

**2. What key metrics would you recommend Foodie-Fi management to track over time to assess performance of their overall business?**  
Some key metrics that Foodie-Fi can monitor to assess performance are the streaming minutes per account and downloaded videos per account. 

**3. What are some key customer journeys or experiences that you would analyse further to improve customer retention?**  
  
**A. Trial Conversion:** One experience to consider is the model of trial defaulting to pro-monthly. We can look at the amount of customers who went from a trial to a pro-monthly plan and then churned rather than downgraded, and the age of their accounts before they churned compared to the total amount of paying customers who churned and the age of their accounts before churn to see if the trial-conversion churn makes up a significant portion of paying churn customers. This can help Foodie-Fi determine if their model is best-suited for retention or if switching the model to a trial-into-basic would increase retention.

As you can see in the query below, 33% of paying churn is attributed to customers who went directly from the trial to a pro-monthly plan and did not change plans prior to canceling their subscription. Additionally, these customers churn, on average, around 88 days vs 109 days of all paying customers who churn.
```sql
WITH paying_customers AS
  (SELECT
    *
  FROM
    (SELECT
      s.customer_id,
      s.plan_id,
      p.plan_name,
      RANK()    
        OVER(PARTITION BY customer_id    
             ORDER BY start_date) AS plan_order,
      s.start_date
    FROM foodie_fi.subscriptions s
      JOIN foodie_fi.plans p
        ON s.plan_id = p.plan_id
    WHERE 
      s.plan_id != 0
    ) plan_order
  WHERE
    CASE WHEN plan_order = 1 AND plan_id = 4
         THEN 1 
          END
     IS NULL
  )
SELECT
  COUNT(plan1.customer_id) AS pro_churn,
  (SELECT
     SUM(CASE WHEN plan_id = 4 THEN 1
              ELSE 0
               END)
   FROM paying_customers
  ) AS total_pay_churn,
  ROUND(COUNT(plan1.customer_id) /
                (SELECT
                   SUM(CASE WHEN plan_id = 4 THEN 1
                            ELSE 0
                             END)
                 FROM paying_customers
                )::DECIMAL * 100, 1) || '%'
      AS percent_churn,
  ROUND(AVG(plan2.start_date - plan1.start_date)) || ' days' AS avg_prochurn_account_age,
  (SELECT
     ROUND(AVG(paychurn.churn_date - pay1.pay_start_date)) || ' days'
   FROM
    (SELECT
       customer_id,
       CASE WHEN plan_order = 1 THEN start_date 
             END AS pay_start_date
     FROM paying_customers
     WHERE
       CASE WHEN plan_order = 1 THEN start_date 
             END
          IS NOT NULL
     ) pay1
   JOIN
    (SELECT
       customer_id,
       CASE WHEN plan_id = 4 THEN start_date 
             END AS churn_date
     FROM paying_customers
     WHERE
       CASE WHEN plan_id = 4 THEN start_date 
             END
          IS NOT NULL
     ) paychurn
   ON pay1.customer_id = paychurn.customer_id
) AS avg_total_paychurn_account_age
FROM
  (SELECT
     customer_id,
     CASE WHEN plan_order = 1 THEN plan_name
           END AS first_plan,
     start_date
   FROM paying_customers
   WHERE
     CASE WHEN plan_order = 1 THEN plan_name
           END IS NOT NULL
  ) plan1
JOIN
  (SELECT
     customer_id,
     CASE WHEN plan_order = 2 THEN plan_name
           END AS second_plan,
     start_date
  FROM paying_customers
  WHERE
     CASE WHEN plan_order = 2 THEN plan_name
           END IS NOT NULL
  ) plan2
ON plan1.customer_id = plan2.customer_id
WHERE
  first_plan = 'pro monthly' AND
  second_plan = 'churn'
```
pro_churn | total_pay_churn | percent_churn | avg_prochurn_account_age | avg_total_paychurn_account_age
--------- | --------------- | ------------- | ------------------------ | ------------------------------
71        | 215             | 33.0%         | 88 days                  | 109 days

**B. Pay Churn Length:** For paying customers that churn, how long do they have a subscription before canceling? Are there any trends?
```sql
WITH churn_journey AS
  (SELECT
    s.customer_id,
    s.plan_id,
    p.plan_name,
    RANK()  
      OVER(PARTITION BY customer_id  
           ORDER BY start_date) AS plan_order,
    s.start_date
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  WHERE
    s.plan_id != 0
  ORDER BY
    s.customer_id
  )
SELECT
  CASE WHEN cd.churn_date - sd.start_date <= 30 THEN '<1 month'
       WHEN cd.churn_date - sd.start_date BETWEEN 31 AND 90 THEN '1-3 months'
       WHEN cd.churn_date - sd.start_date BETWEEN 91 AND 180 THEN '3-6 months'
       WHEN cd.churn_date - sd.start_date BETWEEN 181 AND 270 THEN '6-9 months'
       WHEN cd.churn_date - sd.start_date BETWEEN 271 AND 360 THEN '9-12 months'
       WHEN cd.churn_date - sd.start_date > 360 THEN '12+ months'
        END 
    AS churn_timeframes,
  COUNT(sd.customer_id) AS churn_count
FROM
  (SELECT
     customer_id,
     plan_name AS initial_plan,
     start_date
  FROM churn_journey
  WHERE
     plan_order = 1 AND
     plan_id != 4
  ) sd -- first payment plan start date
JOIN
  (SELECT
     customer_id,
     start_date AS churn_date
  FROM churn_journey
  WHERE
     plan_order != 1 AND
     plan_id = 4
  ) cd -- churn date
ON sd.customer_id = cd.customer_id
GROUP BY
  churn_timeframes
ORDER BY
  churn_count DESC
```
churn_timeframes | churn_count
---------------- | -----------
3-6 months       | 96
1-3 months       | 64
<1 month         | 31
6-9 months       | 17
12+ months       | 6
9-12 months      | 1

**C. Returning Customers:** Are there any customers who churn and return to the service?
```sql
WITH returning_customers AS
  (SELECT
     s.customer_id,
     s.plan_id,
     LAG(s.plan_id)  
      OVER(PARTITION BY s.customer_id  
           ORDER BY s.start_date) AS prev_plan,
     s.start_date
  FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans p
      ON s.plan_id = p.plan_id
  )
SELECT
  COUNT(*) AS returning_customers
FROM returning_customers
WHERE 
  plan_id < prev_plan
```
| returning_customers |
| ------------------- |
| 0                   |

**D. Time Limits:** Are basic_monthly customers hitting their watch time limits (what are these) and, if so, how often? Do basic customers who do hit their limit upgrade, or cancel?  
  
**4. If the Foodie-Fi team were to create an exit survey shown to customers who wish to cancel their subscription, what questions would you include in the survey?**  
1. Why are you canceling your Foodie-Fi subscription?
    * Another service works better for me
    * Poor quality of videos
    * Limited selection of videos
    * Service is too expensive
    * Other *(please comment)*  
2. How can we improve Foodie-Fi? *(optional)*  

**5. What business levers could the Foodie-Fi team use to reduce the customer churn rate? How would you validate the effectiveness of your ideas?**  
Looking at our previous query from 3B, we see that the majority of customers who churn do so between 3-6 months, with another significant portion churning in the first 3 months. Foodie-Fi could offer some loyalty rewards or discounts to increase customer retention. Having a reward/discount structure that is transparent and lets customers know what they can earn by staying subscribed would give incentives to continue subscribing. One example would be partnering with other cooking services to give discounts or samples of their products (coupons for cookware, samples of olive oil or spices, etc). 

Additionally, we know from our query in 3A that paying customers who churn but do not make any changes to their plans churn earlier than those that do make changes. Foodie-Fi could pilot a model that places customers in a basic monthly plan after the free trial or allows customers a choice between either monthly option when they sign up for the trial. The company can then compare churn and overall revenue between their legacy model and the new model to determine which better suits their needs.

Finally, an additional option could be tested allowing customers to pause their service for a certain amount of time. Currently, we know from the query in 3C that once a customer churns they do not return to the service at all. If customers could pause instead of outright cancel, would they return at a later date (perhaps due to limited finances or taking an extended vacation where they won't need Foodie-Fi and don't want to continue paying for it)? By adding an option in the cancellation process to pause service, Foodie-Fi could monitor whether their customers would utilize that option and then later return as a paying customer.
