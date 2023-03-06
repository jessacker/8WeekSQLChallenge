## Case Study #4 - Data Bank

 ### A. Customer Nodes Exploration

**1. How many unique nodes are there on the Data Bank system?**
```sql
SELECT
  SUM(unique_nodes) AS total_unique_nodes
FROM
  (SELECT
    region_id,
    COUNT (DISTINCT node_id) as unique_nodes
  FROM data_bank.customer_nodes
  GROUP BY
    region_id
  ) region_nodes
```
| total_unique_nodes |
| ------------------ |
| 25                 |

**2. What is the number of nodes per region?**
```sql
SELECT
  region_id,
  COUNT (DISTINCT node_id) as unique_nodes
FROM data_bank.customer_nodes
GROUP BY
  region_id
```
| region_id | unique_nodes |
| --------- | ------------ |
| 1         | 5            |
| 2         | 5            |
| 3         | 5            |
| 4         | 5            |
| 5         | 5            |

**3. How many customers are allocated to each region?**
```sql
SELECT
  region_id,
  COUNT (DISTINCT customer_id) as customer_allocation
FROM data_bank.customer_nodes
GROUP BY
  region_id
```
| region_id | customer_allocation |
| --------- | ------------------- |
| 1         | 110                 |
| 2         | 105                 |
| 3         | 102                 |
| 4         | 95                  |
| 5         | 88                  |

**4. How many days on average are customers reallocated to a different node?**
```sql
WITH node_duration AS
  (SELECT
    *,
    end_date - start_date AS days_in_node
  FROM data_bank.customer_nodes
  WHERE
    end_date != '9999-12-31'
  )
SELECT
  ROUND(AVG(days_in_node)) AS avg_days_in_node
FROM node_duration
```
| avg_days_in_node |
| ---------------- |
| 15               |

**5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**
```sql
WITH node_duration AS
  (SELECT
    *,
    end_date - start_date AS days_in_node
  FROM data_bank.customer_nodes
  WHERE
    end_date != '9999-12-31'
  )
SELECT
  region_id,
  PERCENTILE_CONT(0.50) 
      WITHIN GROUP(ORDER BY days_in_node) AS median,
  PERCENTILE_CONT(0.80) 
      WITHIN GROUP(ORDER BY days_in_node) AS percentile_80,
  PERCENTILE_CONT(0.95) 
      WITHIN GROUP(ORDER BY days_in_node) AS percentile_95
FROM node_duration
GROUP BY
  region_id
```
| region_id | median | percentile_80 | percentile_95 |
| --------- | ------ | ------------- | ------------- |
| 1         | 15     | 23            | 28            |
| 2         | 15     | 23            | 28            |
| 3         | 15     | 24            | 28            |
| 4         | 15     | 23            | 28            |
| 5         | 15     | 24            | 28            |

### B. Customer Transactions

**1. What is the unique count and total amount for each transaction type?**
```sql
SELECT
  txn_type,
  COUNT(txn_type) AS txn_count,
  SUM(txn_amount) AS txn_total
FROM data_bank.customer_transactions
GROUP BY
  txn_type
```
| txn_type   | txn_count | txn_total |
| ---------- | --------- | --------- |
| purchase   | 1617      | 806537    |
| deposit    | 2671      | 1359168   |
| withdrawal | 1580      | 793003    |

**2. What is the average total historical deposit counts and amounts for all customers?**
```sql
WITH customer_deposits AS
  (SELECT
    customer_id,
    COUNT(txn_type) AS txn_count,
    SUM(txn_amount) AS txn_total
  FROM data_bank.customer_transactions
  WHERE
    txn_type = 'deposit'
  GROUP BY
    customer_id
  )
SELECT
  ROUND(AVG(txn_count)) AS avg_deposit_count,
  ROUND(AVG(txn_total), 2) AS avg_deposit_total
FROM customer_deposits
```
| avg_deposit_count | avg_deposit_total |
| ----------------- | ----------------- |
| 5                 | 2718.34           |

**3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**
```sql
WITH customer_activity AS
  (SELECT
    customer_id,
    txn_date,
    TO_CHAR(DATE_TRUNC('MONTH', txn_date), 'yyyy-mm') AS txn_month,
    txn_type,
    CASE WHEN txn_type = 'deposit' THEN 1 
         ELSE 0 
          END 
       AS deposits,
    CASE WHEN txn_type = 'withdrawal' OR 
              txn_type = 'purchase' THEN 1 
         ELSE 0 
          END 
       AS fund_removal
  FROM data_bank.customer_transactions
  ORDER BY
    txn_date
  )
SELECT
  txn_month,
  COUNT(customer_id) AS customer_count
FROM
  (SELECT
    txn_month,
    customer_id,
    SUM(deposits) AS monthly_deposits,
    SUM(fund_removal) AS monthly_removals
  FROM customer_activity
  GROUP BY
    txn_month,
    customer_id
  ) txn_count
WHERE
  monthly_deposits >= 2 AND
  monthly_removals = 1  -- interpreted literally as EITHER/OR, not atleast/or
GROUP BY
  txn_month
```
| txn_month | customer_count |
| --------- | -------------- |
| 2020-01   | 53             |
| 2020-02   | 36             |
| 2020-03   | 38             |
| 2020-04   | 22             |

**4. What is the closing balance for each customer at the end of the month?**
```sql
WITH full_calendar AS
  (SELECT
    c.customer_id,
    c.month AS txn_month,
    COALESCE(a.txn_type, 'none') AS txn_type,
    COALESCE(a.txn_amount, 0) AS txn_amount
  FROM
    (SELECT distinct
      customer_id,
      TO_CHAR(GENERATE_SERIES('2020-01-01'::DATE, 
                              '2020-04-30'::DATE, 
                              INTERVAL '1 MONTH'), 'YYYY-MM') 
           AS month
    FROM data_bank.customer_transactions
    ) c --to include months where customers did not have any account activity
  LEFT JOIN
    (SELECT
      customer_id,
      txn_date,
      TO_CHAR(DATE_TRUNC('MONTH', txn_date), 'yyyy-mm') AS txn_month,
      txn_type,
      txn_amount
    FROM data_bank.customer_transactions
    ) a
  ON c.customer_id = a.customer_id AND
     c.month =  a.txn_month
  )
SELECT
  customer_id,
  txn_month,
  SUM(SUM(deposits - fund_removal)) OVER(PARTITION BY customer_id ORDER BY txn_month) AS closing_balance
FROM
  (SELECT
    customer_id,
    txn_month,
    CASE WHEN txn_type = 'deposit' THEN txn_amount 
         ELSE 0 
          END 
       AS deposits,
    CASE WHEN txn_type = 'withdrawal' OR 
              txn_type = 'purchase' THEN txn_amount 
         ELSE 0  
          END 
       AS fund_removal
  FROM full_calendar
  ORDER BY
    customer_id,
    txn_month
  ) aa
GROUP BY
  customer_id,
  txn_month
```
*(sample only)*
| customer_id | txn_month | closing_balance |
| ----------- | --------- | --------------- |
| 1           | 2020-01   | 312             |
| 1           | 2020-02   | 312             |
| 1           | 2020-03   | -640            |
| 1           | 2020-04   | -640            |
| 2           | 2020-01   | 549             |
| 2           | 2020-02   | 549             |
| 2           | 2020-03   | 610             |
| 2           | 2020-04   | 610             |
| 3           | 2020-01   | 144             |
| 3           | 2020-02   | -821            |
| 3           | 2020-03   | -1222           |
| 3           | 2020-04   | -729            |
| 4           | 2020-01   | 848             |
| 4           | 2020-02   | 848             |
| 4           | 2020-03   | 655             |
| 4           | 2020-04   | 655             |
| 5           | 2020-01   | 954             |
| 5           | 2020-02   | 954             |
| 5           | 2020-03   | -1923           |
| 5           | 2020-04   | -2413           |   

*(20 of 2000 total rows shown)*

**5. What is the percentage of customers who increase their closing balance by more than 5%?**
```sql
WITH full_calendar AS
  (SELECT
    c.customer_id,
    c.month AS txn_month,
    COALESCE(a.txn_type, 'none') AS txn_type,
    COALESCE(a.txn_amount, 0) AS txn_amount
  FROM
    (SELECT distinct
      customer_id,
      TO_CHAR(GENERATE_SERIES('2020-01-01'::DATE, 
                              '2020-04-30'::DATE, 
                              INTERVAL '1 MONTH'), 'YYYY-MM') 
           AS month
    FROM data_bank.customer_transactions
    ) c --to include months where customers did not have any account activity
  LEFT JOIN
    (SELECT
      customer_id,
      txn_date,
      TO_CHAR(DATE_TRUNC('MONTH', txn_date), 'yyyy-mm') AS txn_month,
      txn_type,
      txn_amount
    FROM data_bank.customer_transactions
    ) a
  ON c.customer_id = a.customer_id AND
     c.month =  a.txn_month
)
SELECT
  COUNT(DISTINCT customer_id),
  (SELECT 
     COUNT(DISTINCT customer_id)
   FROM data_bank.customer_transactions
  ) AS total_customers,
  ROUND(COUNT(DISTINCT customer_id)::DECIMAL /
          (SELECT 
             COUNT(DISTINCT customer_id)::DECIMAL
          FROM data_bank.customer_transactions)
      * 100, 1) || '%'
    AS percent_of_total
FROM
  (SELECT
    customer_id,
    txn_month,
    LAG(closing_balance) 
       OVER(PARTITION BY customer_id) 
      AS prev_cl_balance,
    closing_balance,
    closing_balance - LAG(closing_balance) 
                        OVER(PARTITION BY customer_id) 
      AS monthly_balance_change
  FROM
    (SELECT
      customer_id,
      txn_month,
      SUM(SUM(deposits - fund_removal)) 
            OVER(PARTITION BY customer_id 
                 ORDER BY txn_month) 
        AS closing_balance
    FROM
      (SELECT
        customer_id,
        txn_month,
        CASE WHEN txn_type = 'deposit' THEN txn_amount 
             ELSE 0 
              END 
           AS deposits,
        CASE WHEN txn_type = 'withdrawal' OR 
                  txn_type = 'purchase' THEN txn_amount 
             ELSE 0 
              END 
           AS fund_removal
      FROM full_calendar
      ) aa
    GROUP BY
      customer_id,
      txn_month
    ) cb
) mbc
WHERE
  monthly_balance_change / NULLIF(prev_cl_balance, 0) > 0.05 AND
  monthly_balance_change > 0 OR
  monthly_balance_change > prev_cl_balance AND
  prev_cl_balance = 0
```	
| count | total_customers | percent_of_total |
| ----- | --------------- | ---------------- |
| 187   | 500             | 37.4%            |

187 of 500 total customers have, at some point in their Data Bank history, increased their monthly balance by at least 5%.
