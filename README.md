# Churn-Rates-Calculation-with-SQL
# Project: Churn Rates Calculation
The churn rate calculation is a common revenue model for SaaS (Software as a service) companies to charge a monthly subscription fee for access to their product.
**Churn Rate** is the precent of subscribers that have canceled within a certain period, usually a month. 

## 1. Get familar with the data
There are 2000 subscribers who make the subscriptions between '2016-12-01' and '2017-03-30'. 
In addition, the subscriptions are made between two groups, **segment = 87**, and **segment = 30**
Thus, the churn rate for the two groups will be calculated for **'2017-01', '2017-02', '2017-03'** respectively.

```sql
/*
SELECT * FROM subscriptions LIMIT 5;
*/
```
id|	subscription_start|	subscription_end|	segment|
--|-------------------|-----------------|--------|
|1|	2016-12-01|	2017-02-01|	87|
|2|	2016-12-01|	2017-01-24|	87|
|3|	2016-12-01|	2017-03-07|	87|
|4|	2016-12-01|	2017-02-12|	87|
|5|	2016-12-01|	2017-03-09|	87|

## 2. Steps of Calculating the Churn Rates
#### Two Parts:
- is_active : the number the subscribers at the beginning of the month
- is_canceled: the number of the subscribers cancel between the month (first_day<= cancel_date <= last_day)
- churn rate = COUNT(is_active)/COUNT(is_canceled)
### Step 1 : Determine the months for calculating the Churn Rates
```sql
SELECT '2017-01-01' AS first_day, '2017-01-31' AS last_day
UNION 
SELECT '2017-02-01' AS first_day, '2017-02-28' AS last_day
UNION 
SELECT '2017-03-01' AS first_day, '2017-03-31' AS last_day;
```
### Output:

first_day|	last_day|
---------|----------|
|2017-01-01|	2017-01-31|
|2017-02-01|	2017-02-28|
|2017-03-01|	2017-03-31|

### Step 2 : Combine the months and the subscriptions tables
### Function used: 
- CROSS JOIN

```sql
WITH months AS (
SELECT '2017-01-01' AS first_day, '2017-01-31' AS last_day
UNION 
SELECT '2017-02-01' AS first_day, '2017-02-28' AS last_day
UNION 
SELECT '2017-03-01' AS first_day, '2017-03-31' AS last_day)

SELECT *
FROM subscriptions 
CROSS JOIN months;
```
### Output:

id|	subscription_start|	subscription_end|	segment|	first_day|	last_day|
--|-------------------|-----------------|--------|------------|---------|
|1|	2016-12-01|	2017-02-01|	87|	2017-01-01|	2017-01-31|
|1|	2016-12-01|	2017-02-01|	87|	2017-02-01|	2017-02-28|
|1|	2016-12-01|	2017-02-01|	87|	2017-03-01|	2017-03-31|
|2|	2016-12-01|	2017-01-24|	87|	2017-01-01|	2017-01-31|
|2|	2016-12-01|	2017-01-24|	87|	2017-02-01|	2017-02-28|

### Step 3 : Determine the active, canceled subscribers,  for two groups (seg = 87 vs seg = 30)

```sql
...
cross_join AS 
(
SELECT *
FROM subscriptions 
CROSS JOIN months )

SELECT  id, first_day AS month, subscription_start, subscription_end, segment,
CASE WHEN 
 (segment = 87 AND
 (subscription_start < first_day) 
 AND (subscription_end >= first_day OR subscription_end IS NULL)) 
 THEN 1 ELSE 0
 END AS is_active_87,
 
 CASE WHEN 
 (segment = 30 AND
 (subscription_start < first_day) 
 AND (subscription_end >= first_day OR subscription_end IS NULL)) 
 THEN 1 ELSE 0
 END AS is_active_30,
 
CASE WHEN 
 (segment = 87 AND
 (subscription_start < first_day) 
 AND (subscription_end BETWEEN first_day AND last_day)) 
 THEN 1 ELSE 0
 END AS is_canceled_87,
 
 CASE WHEN 
 (segment = 30 AND
 (subscription_start < first_day) 
 AND (subscription_end BETWEEN first_day AND last_day)) 
 THEN 1 ELSE 0
 END AS is_canceled_30
 
 FROM cross_join
LIMIT 5;
```
### Output (The columns are kept for cross checking whether the SQL query is correct or not):

id|	month|	subscription_start|	subscription_end|	segment|	is_active_87|	is_active_30|	is_canceled_87|	is_canceled_30|
-|-------|--------------------|-----------------|--------|--------------|-------------|---------------|---|
|1	|2017-01-01	|2016-12-01	|2017-02-01	|87	|1	|0	|0	|0|
|1	|2017-02-01	|2016-12-01	|2017-02-01	|87	|1	|0	|1	|0|
|1	|2017-03-01	|2016-12-01	|2017-02-01	|87	|0	|0	|0	|0|
|2	|2017-01-01	|2016-12-01	|2017-01-24	|87	|1	|0	|1	|0|
|2	|2017-02-01	|2016-12-01	|2017-01-24	|87	|0	|0	|0	|0|

### Step 4 : Calculate the counts of the above results for each month separatively
```sql
...
 
 SELECT month, 
 SUM(is_active_87) AS counts_active_87,
 SUM(is_active_30) AS counts_active_30,
 SUM(is_canceled_87) AS counts_canceled_87,
 SUM(is_canceled_30) AS counts_canceled_30
 FROM status
 GROUP BY month;
```
### Output

|month	|counts_active_87	|counts_active_30	|counts_canceled_87	|counts_canceled_30|
--------|-----------------|-----------------|------------------|-------------------|
|2017-01-01	|279	|291	|70	|22|
|2017-02-01	|467	|518	|148|38|
|2017-03-01	|541	|718	|258|84|

### Step 5 : Calculate the churn rates for each groups for the 3 months respectively

```sql
...
 SELECT SUBSTR(month, 1, 7) AS month, 
ROUND(1.0 *  counts_canceled_87/counts_active_87, 2) AS churn_rate_87,
ROUND(1.0 *  counts_canceled_30/counts_active_30, 2) AS churn_rate_30
 FROM status_aggregate;
```
### Output: The churn rates for the two groups among the three months

|month	|churn_rate_87	|churn_rate_30|
--------|--------------|--------------|
|2017-01	|0.25	|0.08|
|2017-02	|0.32	|0.07|
|2017-03	|0.48	|0.12|

### Conclusion 
From the process above, vertically, the churn rates from increasing from January to March, but the degree are different for the two segments. 
In addition, horizontally, the churn rates for segment 30 are significantly smaller than those of the segment 87. 
Thus, the **segment 30** is **better** for this product with significantly **lower churn rate** (higher retention rate).


### Appendix: the whole query
```sql
WITH months AS (
SELECT '2017-01-01' AS first_day, '2017-01-31' AS last_day
UNION 
SELECT '2017-02-01' AS first_day, '2017-02-28' AS last_day
UNION 
SELECT '2017-03-01' AS first_day, '2017-03-31' AS last_day),

cross_join AS 
(
SELECT *
FROM subscriptions 
CROSS JOIN months ),

status AS (
SELECT  id, first_day AS month, subscription_start, subscription_end, segment,
CASE WHEN 
 (segment = 87 AND
 (subscription_start < first_day) 
 AND (subscription_end >= first_day OR subscription_end IS NULL)) 
 THEN 1 ELSE 0
 END AS is_active_87,
 
 CASE WHEN 
 (segment = 30 AND
 (subscription_start < first_day) 
 AND (subscription_end >= first_day OR subscription_end IS NULL)) 
 THEN 1 ELSE 0
 END AS is_active_30,
 
CASE WHEN 
 (segment = 87 AND
 (subscription_start < first_day) 
 AND (subscription_end BETWEEN first_day AND last_day)) 
 THEN 1 ELSE 0
 END AS is_canceled_87,
 
 CASE WHEN 
 (segment = 30 AND
 (subscription_start < first_day) 
 AND (subscription_end BETWEEN first_day AND last_day)) 
 THEN 1 ELSE 0
 END AS is_canceled_30
 
 FROM cross_join),
 
 status_aggregate AS (
 SELECT month, 
 SUM(is_active_87) AS counts_active_87,
 SUM(is_active_30) AS counts_active_30,
 SUM(is_canceled_87) AS counts_canceled_87,
 SUM(is_canceled_30) AS counts_canceled_30
 FROM status
 GROUP BY month)
 
 SELECT SUBSTR(month, 1, 7) AS month, 
ROUND(1.0 *  counts_canceled_87/counts_active_87, 2) AS churn_rate_87,
ROUND(1.0 *  counts_canceled_30/counts_active_30, 2) AS churn_rate_30
 FROM status_aggregate;
```
