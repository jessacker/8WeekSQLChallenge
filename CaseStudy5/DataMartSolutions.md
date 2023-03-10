## Case Study Questions
The following case study questions require some data cleaning steps before we start to unpack Danny’s key business questions in more depth.

### A. Data Cleansing Steps
In a single query, perform the following operations and generate a new table in the data_mart schema named clean_weekly_sales:
* convert the `week_date` to a DATE format
* add a `week_number` as the second column for each `week_date` value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc
* add a `month_number` with the calendar month for each `week_date` value as the 3rd column
* add a `calendar_year` column as the 4th column containing either 2018, 2019 or 2020 values
* add a new column called `age_band` after the original segment column using the following mapping on the number inside the segment value
	* 1	Young Adults
	* 2	Middle Aged
	* 3 or 4 Retirees
* add a new `demographic` column using the following mapping for the first letter in the segment values:
	* C	Couples
	* F	Families
* ensure all null string values with an "unknown" string value in the original segment column as well as the `new age_band` and `demographic` columns
* generate a new `avg_transaction` column as the `sales` value divided by `transactions` rounded to 2 decimal places for each record
```sql
DROP TABLE IF EXISTS clean_weekly_sales;
CREATE TABLE clean_weekly_sales AS
  (SELECT 
     TO_DATE(week_date, 'DD-MM-YY') AS date,
     DATE_PART('week', (TO_DATE(week_date, 'DD-MM-YY'))) AS week_number,
     DATE_PART('month', (TO_DATE(week_date, 'DD-MM-YY'))) AS month,
     DATE_PART('year', (TO_DATE(week_date, 'DD-MM-YY'))) AS year,
     region,
     platform,
     customer_type,
     segment,
     CASE WHEN RIGHT(segment, 1) = '1' THEN 'Young Adults' 
          WHEN RIGHT(segment, 1) = '2' THEN 'Middle Aged'  
          WHEN RIGHT(segment, 1) = '3' OR  RIGHT(segment, 1) = '4' THEN 'Retirees'  
          ELSE 'Unknown'  
           END  
        AS age_band,
     CASE WHEN LEFT(segment, 1) = 'C' THEN 'Couples' 
          WHEN LEFT(segment, 1) = 'F' THEN 'Families' 
          ELSE 'Unknown' 
           END 
        AS demographics,
     transactions,
     sales,
     ROUND(sales / transactions::DECIMAL, 2) AS avg_transactions
  FROM data_mart.weekly_sales
  )
```
*(sample)*
| date                     | week_number | month | year | region | platform | customer_type | segment | age_band     | demographics | transactions | sales    | avg_transactions |
| ------------------------ | ----------- | ----- | ---- | ------ | -------- | ------------- | ------- | ------------ | ------------ | ------------ | -------- | ---------------- |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | ASIA   | Retail   | New           | C3      | Retirees     | Couples      | 120631       | 3656163  | 30               |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | ASIA   | Retail   | New           | F1      | Young Adults | Families     | 31574        | 996575   | 31               |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | USA    | Retail   | Guest         | null    | Unknown      | Unknown      | 529151       | 16509610 | 31               |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | EUROPE | Retail   | New           | C1      | Young Adults | Couples      | 4517         | 141942   | 31               |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | AFRICA | Retail   | New           | C2      | Middle Aged  | Couples      | 58046        | 1758388  | 30               |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | CANADA | Shopify  | Existing      | F2      | Middle Aged  | Families     | 1336         | 243878   | 182              |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | AFRICA | Shopify  | Existing      | F3      | Retirees     | Families     | 2514         | 519502   | 206              |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | ASIA   | Shopify  | Existing      | F1      | Young Adults | Families     | 2158         | 371417   | 172              |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | AFRICA | Shopify  | New           | F2      | Middle Aged  | Families     | 318          | 49557    | 155              |
| 2020-08-31T00:00:00.000Z | 36          | 8     | 2020 | AFRICA | Retail   | New           | C3      | Retirees     | Couples      | 111032       | 3888162  | 35               |

### B. Data Exploration
**1. What day of the week is used for each week_date value?**
```sql
SELECT DISTINCT
  TO_CHAR(date, 'Day') AS weekday
FROM clean_weekly_sales
```
| weekday   |
| --------- |
| Monday    |

**2. What range of week numbers are missing from the dataset?**
```sql
WITH wn AS
  (SELECT
     GENERATE_SERIES(1, 53, 1) AS week_number
  )
SELECT
  wn.week_number AS missing_weeks
FROM wn
LEFT OUTER JOIN clean_weekly_sales cws
ON 
  wn.week_number = cws.week_number
WHERE 
  cws.week_number IS NULL
```
| missing_weeks |
| ------------- |
| 1             |
| 2             |
| 3             |
| 4             |
| 5             |
| 6             |
| 7             |
| 8             |
| 9             |
| 10            |
| 11            |
| 12            |
| 37            |
| 38            |
| 39            |
| 40            |
| 41            |
| 42            |
| 43            |
| 44            |
| 45            |
| 46            |
| 47            |
| 48            |
| 49            |
| 50            |
| 51            |
| 52            |
| 53            |

**3. How many total transactions were there for each year in the dataset?**
```sql
SELECT
  year,
  SUM(transactions) AS total_transactions
FROM clean_weekly_sales
GROUP BY
  year
ORDER BY
  year
```
| year | total_transactions |
| ---- | ------------------ |
| 2018 | 346406460          |
| 2019 | 365639285          |
| 2020 | 375813651          |

**4. What is the total sales for each region for each month?**
```sql
SELECT
  region,
  month,
  year,
  SUM(sales) AS total_sales
FROM clean_weekly_sales
GROUP BY
  region,
  month,
  year
ORDER BY
  year,
  month
```
*(sample only)*	
| region        | month | year | total_sales |
| ------------- | ----- | ---- | ----------- |
| ASIA          | 3     | 2018 | 119180883   |
| AFRICA        | 3     | 2018 | 130542213   |
| SOUTH AMERICA | 3     | 2018 | 16302144    |
| USA           | 3     | 2018 | 52734998    |
| OCEANIA       | 3     | 2018 | 175777460   |
| CANADA        | 3     | 2018 | 33815571    |
| EUROPE        | 3     | 2018 | 8402183     |  
*(7 of 140 total rows shown)*

**5. What is the total count of transactions for each platform?**
```sql
SELECT
  platform,
  SUM(transactions) AS total_transactions
FROM clean_weekly_sales
GROUP BY
  platform
ORDER BY
  total_transactions DESC
```
| platform | total_transactions |
| -------- | ------------------ |
| Retail   | 1081934227         |
| Shopify  | 5925169            |	

**6. What is the percentage of sales for Retail vs Shopify for each month?**
```sql
SELECT
  rs.month,
  rs.year,
  ROUND(SUM(rs.sales)::DECIMAL /
        (SUM(rs.sales)::DECIMAL +
         SUM(ss.sales)::DECIMAL)
       * 100, 1) || '%' 
      AS retail_percent,
  ROUND(SUM(ss.sales)::DECIMAL /
       (SUM(rs.sales)::DECIMAL +
        SUM(ss.sales)::DECIMAL)
       * 100, 1) || '%' 
      AS shopify_percent
FROM
  (SELECT
     month,
     year,
     sales
   FROM clean_weekly_sales
   WHERE
     platform = 'Retail'
  ) rs
JOIN
  (SELECT
     month,
     year,
     sales
   FROM clean_weekly_sales
   WHERE
     platform = 'Shopify'
  ) ss
ON 	rs.month = ss.month AND
    rs.year = ss.year
GROUP BY
  rs.year,
  rs.month
ORDER BY
  rs.year,
  rs.month
```
| month | year | retail_percent | shopify_percent |
| ----- | ---- | -------------- | --------------- |
| 3     | 2018 | 97.9%          | 2.1%            |
| 4     | 2018 | 97.9%          | 2.1%            |
| 5     | 2018 | 97.7%          | 2.3%            |
| 6     | 2018 | 97.8%          | 2.2%            |
| 7     | 2018 | 97.7%          | 2.3%            |
| 8     | 2018 | 97.7%          | 2.3%            |
| 9     | 2018 | 97.7%          | 2.3%            |
| 3     | 2019 | 97.7%          | 2.3%            |
| 4     | 2019 | 97.8%          | 2.2%            |
| 5     | 2019 | 97.5%          | 2.5%            |
| 6     | 2019 | 97.4%          | 2.6%            |
| 7     | 2019 | 97.4%          | 2.6%            |
| 8     | 2019 | 97.2%          | 2.8%            |
| 9     | 2019 | 97.1%          | 2.9%            |
| 3     | 2020 | 97.3%          | 2.7%            |
| 4     | 2020 | 97.0%          | 3.0%            |
| 5     | 2020 | 96.7%          | 3.3%            |
| 6     | 2020 | 96.8%          | 3.2%            |
| 7     | 2020 | 96.7%          | 3.3%            |
| 8     | 2020 | 96.5%          | 3.5%            |

**7. What is the percentage of sales by demographic for each year in the dataset?**
```sql
SELECT
  c.year,
  ROUND(c.couple_sales /
        (c.couple_sales + f.family_sales + u.unknown_sales)::DECIMAL
      * 100, 1) || '%' 
      AS couples_sales,
  ROUND(f.family_sales /
        (c.couple_sales + f.family_sales + u.unknown_sales)::DECIMAL
      * 100, 1) || '%' 
      AS families_sales,
  ROUND(u.unknown_sales /
        (c.couple_sales + f.family_sales + u.unknown_sales)::DECIMAL
      * 100, 1) || '%' 
      AS unknown_sales
FROM
  (SELECT
     year,
     SUM(sales) AS couple_sales
   FROM clean_weekly_sales
   WHERE
     demographics = 'Couples'
   GROUP BY
     year
  ) c
JOIN
  (SELECT
     year,
     SUM(sales) AS family_sales
   FROM clean_weekly_sales
   WHERE
     demographics = 'Families'
   GROUP BY
     year
  ) f
ON 	c.year = f.year
JOIN
  (SELECT
     year,
     SUM(sales) AS unknown_sales
   FROM clean_weekly_sales
   WHERE
     demographics = 'Unknown'
   GROUP BY
     year
  ) u
ON 	f.year = u.year
ORDER BY
  c.year
```
| year | couples_sales | families_sales | unknown_sales |
| ---- | ------------- | -------------- | ------------- |
| 2018 | 26.4%         | 32.0%          | 41.6%         |
| 2019 | 27.3%         | 32.5%          | 40.3%         |
| 2020 | 28.7%         | 32.7%          | 38.6%         |

**8. Which age_band and demographic values contribute the most to Retail sales?**
```sql
SELECT
  CASE WHEN segment = 'C1' THEN 'Young Adult Couples' 
       WHEN segment = 'C2' THEN 'Middled Aged Couples' 
       WHEN segment = 'C3' OR 
            segment = 'C4' THEN 'Retiree Couples' 
       WHEN segment = 'F1' THEN 'Young Adult Families' 
       WHEN segment = 'F2' THEN 'Middled Aged Families' 
       WHEN segment = 'F3' OR 
            segment = 'F4' THEN 'Retiree Families' 
       ELSE 'Unknown' 
        END 
     AS segment_details,
  SUM(sales) AS segment_sales,
  ROUND(SUM(sales)::DECIMAL /
        (SELECT
           SUM(sales)
        FROM clean_weekly_sales
        WHERE 
           platform = 'Retail'
        )::DECIMAL 
      * 100, 1) || '%'
    AS percent_of_sales
FROM clean_weekly_sales
WHERE
  platform = 'Retail'
GROUP BY
  segment_details
ORDER BY
  segment_sales DESC
```
| segment_details       | segment_sales | percent_of_sales |
| --------------------- | ------------- | ---------------- |
| Unknown               | 16067285533   | 40.5%            |
| Retiree Families      | 6634686916    | 16.7%            |
| Retiree Couples       | 6370580014    | 16.1%            |
| Middled Aged Families | 4354091554    | 11.0%            |
| Young Adult Couples   | 2602922797    | 6.6%             |
| Middled Aged Couples  | 1854160330    | 4.7%             |
| Young Adult Families  | 1770889293    | 4.5%             |

**9. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?**  
Outside of specific use cases, it is always best practice to calculate the average directly, rather than taking the average of an average. 

Let's compare a newly calculated average:
```sql
SELECT
  r.year,
  r.avg_sale_amount AS retail_avg_sale,
  s.avg_sale_amount AS shopify_avg_sale
FROM
  (SELECT
     year,
     ROUND(SUM(sales) / SUM(transactions)::DECIMAL, 2) AS avg_sale_amount
   FROM clean_weekly_sales
   WHERE
     platform = 'Retail'
   GROUP BY
     year
  ) r
JOIN
  (SELECT
     year,
     ROUND(SUM(sales) / SUM(transactions)::DECIMAL, 2) AS avg_sale_amount
   FROM clean_weekly_sales
   WHERE
     platform = 'Shopify'
   GROUP BY
     year
  ) s
ON r.year = s.year
```
| year | retail_avg_sale | shopify_avg_sale |
| ---- | --------------- | ---------------- |
| 2018 | 36.56           | 192.48           |
| 2019 | 36.83           | 183.36           |
| 2020 | 36.56           | 179.03           |

With the average of the average:
```sql
SELECT
  r.year,
  r.avg_transactions AS retail_avg_avg,
  s.avg_transactions AS shopify_avg_avg
FROM
  (SELECT
     year,
     ROUND(AVG(avg_transactions), 2) AS avg_transactions
   FROM clean_weekly_sales
   WHERE
     platform = 'Retail'
   GROUP BY
     year
  ) r
JOIN
  (SELECT
     year,
     ROUND(AVG(avg_transactions), 2) AS avg_transactions
   FROM clean_weekly_sales
   WHERE
     platform = 'Shopify'
   GROUP BY
     year
  ) s
ON r.year = s.year
```
| year | retail_avg_avg | shopify_avg_avg |
| ---- | -------------- | --------------- |
| 2018 | 42.91          | 188.28          |
| 2019 | 41.97          | 177.56          |
| 2020 | 40.64          | 174.87          |

### C. Before & After Analysis
This technique is usually used when we inspect an important event and want to inspect the impact before and after a certain point in time.

Taking the week_date value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect.

We would include all week_date values for 2020-06-15 as the start of the period after the change and the previous week_date values would be before

Using this analysis approach - answer the following questions:

**1. What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?**
```sql
WITH pr4 AS
  (SELECT
     SUM(sales) AS prechange_sales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '4 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
  ),
po4 AS
  (SELECT
     SUM(sales) AS postchange_sales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE AND
                  '2020-06-15'::DATE + INTERVAL '3 weeks'
  )
SELECT
  prechange_sales,
  postchange_sales,
  postchange_sales - prechange_sales AS sales_change,
  ROUND((postchange_sales - prechange_sales)::DECIMAL / 
          prechange_sales * 100, 2) || '%' 
     AS percentage_change
FROM
  pr4,
  po4
```
| prechange_sales | postchange_sales | sales_change | percentage_change |
| --------------- | ---------------- | ------------ | ----------------- |
| 2345878357      | 2318994169       | -26884188    | -1.15%            |

**2. What about the entire 12 weeks before and after?**
```sql
WITH pr12 AS
  (SELECT
     SUM(sales) AS prechange_sales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '12 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
  ),
po12 AS
  (SELECT
     SUM(sales) AS postchange_sales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE AND
                  '2020-06-15'::DATE + INTERVAL '11 weeks'
  )
SELECT
  prechange_sales,
  postchange_sales,
  postchange_sales - prechange_sales AS sales_change,
  ROUND((postchange_sales - prechange_sales)::DECIMAL /
          prechange_sales * 100, 2) || '%' 
     AS percentage_change
FROM
  pr12,
  po12
```
| prechange_sales | postchange_sales | sales_change | percentage_change |
| --------------- | ---------------- | ------------ | ----------------- |
| 7126273147      | 6973947753       | -152325394   | -2.14%            |

**3. How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?**  
For this question, we'll first want to determine what week_number corresponds to the date of the changes made by Data Mart.
```sql
SELECT DISTINCT
  date,
  week_number
FROM clean_weekly_sales
WHERE
  date = '2020-06-15'
```
| date                     | week_number |
| ------------------------ | ----------- |
| 2020-06-15T00:00:00.000Z | 25          |

With this information, we can modify the above queries to include weeks in the desired ranges for each year.
We'll start with the 4 weeks before and after the change:
```sql
SELECT
  yopr4.year,
  yopr4.presales AS sales_weeks_21to24,
  yopo4.postsales AS sales_weeks_25to28,
  yopo4.postsales - yopr4.presales AS sales_change,
  ROUND((yopo4.postsales - yopr4.presales)::DECIMAL / yopr4.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     year,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     week_number BETWEEN (25 - 4) AND
                         (25 - 1)
   GROUP BY
     year
  ) yopr4
JOIN
  (SELECT
     year,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     week_number BETWEEN 25 AND
                        (25 + 3)
   GROUP BY
     year
  ) yopo4
ON yopr4.year = yopo4.year
```
| year | sales_weeks_21to24 | sales_weeks_25to28 | sales_change | percentage_change |
| ---- | ------------------ | ------------------ | ------------ | ----------------- |
| 2018 | 2125140809         | 2129242914         | 4102105      | 0.19%             |
| 2019 | 2249989796         | 2252326390         | 2336594      | 0.10%             |
| 2020 | 2345878357         | 2318994169         | -26884188    | -1.15%            |

Now, the full 12 weeks before and after the change.
```sql
SELECT
  yopr12.year,
  yopr12.presales AS sales_weeks_13to24,
  yopo12.postsales AS sales_weeks_25to36,
  yopo12.postsales - yopr12.presales AS sales_change,
  ROUND((yopo12.postsales - yopr12.presales)::DECIMAL / yopr12.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     year,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     week_number BETWEEN (25 - 12) AND
                         (25 - 1)
   GROUP BY
     year
  ) yopr12
JOIN
  (SELECT
     year,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     week_number BETWEEN 25 AND
                        (25 + 11)
   GROUP BY
     year
  ) yopo12
ON yopr12.year = yopo12.year
```
| year | sales_weeks_13to24 | sales_weeks_25to36 | sales_change | percentage_change |
| ---- | ------------------ | ------------------ | ------------ | ----------------- |
| 2018 | 6396562317         | 6500818510         | 104256193    | 1.63%             |
| 2019 | 6883386397         | 6862646103         | -20740294    | -0.30%            |
| 2020 | 7126273147         | 6973947753         | -152325394   | -2.14%            |

### D. Bonus Questions
**1. Which areas of the business have the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period?**
* `region`
* `platform`
* `age_band`
* `demographic`
* `customer_type`
- - - - 
`region`
```sql
SELECT
  rpr.region,
  rpr.presales AS prechange_sales,
  rpo.postsales AS postchange_sales,
  rpo.postsales - rpr.presales AS sales_change,
  ROUND((rpo.postsales - rpr.presales)::DECIMAL / rpr.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     region,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '12 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
   GROUP BY
     region
  ) rpr
JOIN
  (SELECT
     region,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15' AND
                  '2020-06-15'::DATE + INTERVAL '11 weeks'
   GROUP BY
     region
  ) rpo
ON rpr.region = rpo.region
ORDER BY
  sales_change
```
| region        | prechange_sales | postchange_sales | sales_change | percentage_change |
| ------------- | --------------- | ---------------- | ------------ | ----------------- |
| OCEANIA       | 2354116790      | 2282795690       | -71321100    | -3.03%            |
| ASIA          | 1637244466      | 1583807621       | -53436845    | -3.26%            |
| USA           | 677013558       | 666198715        | -10814843    | -1.60%            |
| AFRICA        | 1709537105      | 1700390294       | -9146811     | -0.54%            |
| CANADA        | 426438454       | 418264441        | -8174013     | -1.92%            |
| SOUTH AMERICA | 213036207       | 208452033        | -4584174     | -2.15%            |
| EUROPE        | 108886567       | 114038959        | 5152392      | 4.73%             |
- - - - 
`platform`
```sql
SELECT
  ppr.platform,
  ppr.presales AS prechange_sales,
  ppo.postsales AS postchange_sales,
  ppo.postsales - ppr.presales AS sales_change,
  ROUND((ppo.postsales - ppr.presales)::DECIMAL / ppr.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     platform,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '12 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
   GROUP BY
     platform
  ) ppr
JOIN
  (SELECT
     platform,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15' AND
                  '2020-06-15'::DATE + INTERVAL '11 weeks'
   GROUP BY
     platform
  ) ppo
ON ppr.platform = ppo.platform
ORDER BY
  sales_change
```
| platform | prechange_sales | postchange_sales | sales_change | percentage_change |
| -------- | --------------- | ---------------- | ------------ | ----------------- |
| Retail   | 6906861113      | 6738777279       | -168083834   | -2.43%            |
| Shopify  | 219412034       | 235170474        | 15758440     | 7.18%             |
- - - - 
`age_band`
```sql
SELECT
  apr.age_band,
  apr.presales AS prechange_sales,
  apo.postsales AS postchange_sales,
  apo.postsales - apr.presales AS sales_change,
  ROUND((apo.postsales - apr.presales)::DECIMAL / apr.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     age_band,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '12 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
   GROUP BY
     age_band
  ) apr
JOIN
  (SELECT
     age_band,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15' AND
                  '2020-06-15'::DATE + INTERVAL '11 weeks'
   GROUP BY
     age_band
  ) apo
ON apr.age_band = apo.age_band
ORDER BY
  sales_change
```
| age_band     | prechange_sales | postchange_sales | sales_change | percentage_change |
| ------------ | --------------- | ---------------- | ------------ | ----------------- |
| Unknown      | 2764354464      | 2671961443       | -92393021    | -3.34%            |
| Retirees     | 2395264515      | 2365714994       | -29549521    | -1.23%            |
| Middle Aged  | 1164847640      | 1141853348       | -22994292    | -1.97%            |
| Young Adults | 801806528       | 794417968        | -7388560     | -0.92%            |
- - - - 
`demographics`
```sql
SELECT
  dpr.demographics,
  dpr.presales AS prechange_sales,
  dpo.postsales AS postchange_sales,
  dpo.postsales - dpr.presales AS sales_change,
  ROUND((dpo.postsales - dpr.presales)::DECIMAL / dpr.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     demographics,  
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '12 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
   GROUP BY
  demographics
  ) dpr
JOIN
  (SELECT
     demographics,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15' AND
                  '2020-06-15'::DATE + INTERVAL '11 weeks'
   GROUP BY
     demographics
  ) dpo
ON dpr.demographics = dpo.demographics
ORDER BY
  sales_change
```
| demographics | prechange_sales | postchange_sales | sales_change | percentage_change |
| ------------ | --------------- | ---------------- | ------------ | ----------------- |
| Unknown      | 2764354464      | 2671961443       | -92393021    | -3.34%            |
| Families     | 2328329040      | 2286009025       | -42320015    | -1.82%            |
| Couples      | 2033589643      | 2015977285       | -17612358    | -0.87%            |
- - - - 
`customer_type`
```sql
SELECT
  cpr.customer_type,
  cpr.presales AS prechange_sales,
  cpo.postsales AS postchange_sales,
  cpo.postsales - cpr.presales AS sales_change,
  ROUND((cpo.postsales - cpr.presales)::DECIMAL / cpr.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     customer_type,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '12 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
   GROUP BY
     customer_type
  ) cpr
JOIN
  (SELECT
     customer_type,
     SUM(sales) AS postsales
  FROM clean_weekly_sales
  WHERE
     date BETWEEN '2020-06-15' AND
                  '2020-06-15'::DATE + INTERVAL '11 weeks'
  GROUP BY
     customer_type
  ) cpo
ON cpr.customer_type = cpo.customer_type
ORDER BY
  sales_change
```
| customer_type | prechange_sales | postchange_sales | sales_change | percentage_change |
| ------------- | --------------- | ---------------- | ------------ | ----------------- |
| Existing      | 3690116427      | 3606243454       | -83872973    | -2.27%            |
| Guest         | 2573436301      | 2496233635       | -77202666    | -3.00%            |
| New           | 862720419       | 871470664        | 8750245      | 1.01%             |
- - - - 
**2. Do you have any further recommendations for Danny’s team at Data Mart or any interesting insights based off this analysis?**  
The `customer_type` query indicates that there was growth among new customers in the 12 weeks following the sustainability changes, while sales reduction occured for both existing and guest customer types. If we were to combine `customer_type` with `platform` the following results shed more light on the situation:
```sql
SELECT
  cpr.platform,
  cpr.customer_type,
  cpr.presales AS prechange_sales,
  cpo.postsales AS postchange_sales,
  cpo.postsales - cpr.presales AS sales_change,
  ROUND((cpo.postsales - cpr.presales)::DECIMAL / cpr.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     platform,
     customer_type,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15'::DATE - INTERVAL '12 weeks' AND
                  '2020-06-15'::DATE - INTERVAL '1 week'
  GROUP BY
     platform,
     customer_type
  ) cpr
JOIN
  (SELECT
     platform,
     customer_type,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     date BETWEEN '2020-06-15' AND
                  '2020-06-15'::DATE + INTERVAL '11 weeks'
   GROUP BY
     platform,
     customer_type
  ) cpo
ON cpr.customer_type = cpo.customer_type AND
   cpr.platform = cpo.platform
ORDER BY
  customer_type
```
| platform | customer_type | prechange_sales | postchange_sales | sales_change | percentage_change |
| -------- | ------------- | --------------- | ---------------- | ------------ | ----------------- |
| Retail   | Existing      | 3537711266      | 3444979832       | -92731434    | -2.62%            |
| Shopify  | Existing      | 152405161       | 161263622        | 8858461      | 5.81%             |
| Retail   | Guest         | 2520570547      | 2437674013       | -82896534    | -3.29%            |
| Shopify  | Guest         | 52865754        | 58559622         | 5693868      | 10.77%            |
| Retail   | New           | 848579300       | 856123434        | 7544134      | 0.89%             |
| Shopify  | New           | 14141119        | 15347230         | 1206111      | 8.53%             |

As we see, retail sales suffered more heavily than shopify, especially in relation to existing customers and guests. This may indicate that packaging changes were not well advertised ahead of the change, isolating familiar customers from the brand and leading to sales loss due to unfamiliar packaging. If Data Mart's Shopify works like other online shops, the "buy again" feature can help customers overcome packaging changes that they may not otherwise recognise in a brick-and-mortar store without closer inspection.

For future changes, advertising ahead of time or including a label on the package for a certain period of time after the change (same product, different label type deal) will help alert customers to branding changes and prepare them for finding the product on the shelves when shopping.

Additionally, we know from one of the earlier queries that, of the known demographics and age_bands, Retirees contribute the most to Retail sales. However, with additional inspection looking at all `age_band` groups and only the existing `customer_type` we actually find that, again of the known customers, both loss of sales in Retail shops and growth through Shopify can be contributed most to the Middle Aged `age_band`.  

| platform | customer_type | age_band     | prechange_sales | postchange_sales | sales_change | percentage_change |
| -------- | ------------- | ------------ | --------------- | ---------------- | ------------ | ----------------- |
| Retail   | Existing      | Middle Aged  | 893886904       | 866119309        | -27767595    | -3.11%            |
| Shopify  | Existing      | Middle Aged  | 61482439        | 65415581         | 3933142      | 6.40%             |
| Retail   | Existing      | Retirees     | 1974671630      | 1932910269       | -41761361    | -2.11%            |
| Shopify  | Existing      | Retirees     | 51886523        | 54354788         | 2468265      | 4.76%             |
| Retail   | Existing      | Unknown      | 75637074        | 67362688         | -8274386     | -10.94%           |
| Shopify  | Existing      | Unknown      | 3109534         | 3363905          | 254371       | 8.18%             |
| Retail   | Existing      | Young Adults | 593515658       | 578587566        | -14928092    | -2.52%            |
| Shopify  | Existing      | Young Adults | 35926665        | 38129348         | 2202683      | 6.13%             |

 This is especially helpful when determining what key demographics should be targeted for marketing campaigns, especially ahead of branding changes. If an existing customer consents to receive promotional communications, send reminders or even small samples in the new packaging ahead of release to help ensure loyal customers are aware of the coming changes. 
 
Finally, while year-over-year Retail has shown a downward sales trend, even before sustainability changes, Shopify has shown consistent growth and may be worth exploring further among key demographics as a major growth potential. 
```sql
SELECT
  yopr12.year,
  yopr12.platform,
  yopr12.presales AS sales_weeks_13to24,
  yopo12.postsales AS sales_weeks_25to36,
  yopo12.postsales - yopr12.presales AS sales_change,
  ROUND((yopo12.postsales - yopr12.presales)::DECIMAL / yopr12.presales * 100, 2) || '%' AS percentage_change
FROM
  (SELECT
     year,
     platform,
     SUM(sales) AS presales
   FROM clean_weekly_sales
   WHERE
     week_number BETWEEN (25 - 12) AND
                         (25 - 1)
   GROUP BY
     year,
     platform
  ) yopr12
JOIN
  (SELECT
     year,
     platform,
     SUM(sales) AS postsales
   FROM clean_weekly_sales
   WHERE
     week_number BETWEEN 25 AND
                        (25 + 11)
   GROUP BY
     year,
     platform
  ) yopo12
ON yopr12.year = yopo12.year AND
   yopr12.platform = yopo12.platform
```   
| year | platform | sales_weeks_13to24 | sales_weeks_25to36 | sales_change | percentage_change |
| ---- | -------- | ------------------ | ------------------ | ------------ | ----------------- |
| 2018 | Retail   | 6257743808         | 6353427510         | 95683702     | 1.53%             |
| 2018 | Shopify  | 138818509          | 147391000          | 8572491      | 6.18%             |
| 2019 | Retail   | 6721435351         | 6676371376         | -45063975    | -0.67%            |
| 2019 | Shopify  | 161951046          | 186274727          | 24323681     | 15.02%            |
| 2020 | Retail   | 6906861113         | 6738777279         | -168083834   | -2.43%            |
| 2020 | Shopify  | 219412034          | 235170474          | 15758440     | 7.18%             |

