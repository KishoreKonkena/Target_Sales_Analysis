# Target_Sales_Analysis

[Click Here to get Dataset](https://drive.google.com/drive/folders/1aJZRWKsmFS6zyPUwT6bticRWacZlZFoC?usp=drive_link)

![Target Logo](images.png)

## Overview

This project focuses on analyzing Target sales data using Google BigQuery. The aim is to practice advanced SQL skills while extracting meaningful business insights about customer behavior, sales trends, delivery efficiency, and payment preferences.

## The project workflow covers:

**Data Exploration**: Checking dataset characteristics, data types, and time ranges.

**Customer Insights**: Distribution of customers across states, cities, and seasonal order behaviors.

**Sales Trends**: Yearly and monthly growth trends, seasonality in order placement, and time-of-day analysis.

**Operational Insights**: Delivery time analysis vs. estimated delivery, top states with highest/lowest freight values, and fast delivery performance.

**Payment Analysis**: Payment methods, installments, and their influence on customer purchase behavior.

``` BigQuery
-- Example: Customers Table Check
SELECT column_name, data_type
FROM `Target_SQL.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'customers';
```
## Project Steps

1. Data Exploration:

Before performing analysis, the dataset was explored to understand its structure and characteristics.

Checked data types of columns in the customers table.

Identified the time range for which orders were placed.

Reviewed dataset size and column details to understand available attributes.

2. Customer & Geographic Insights:

Extracted details of cities and states where customers placed orders during September & December 2018.

Analyzed customer distribution across states.

Identified top states with the highest and lowest average freight values.

3. Sales & Trend Analysis:

Checked if there is a growing trend in the number of orders placed across years.

Identified monthly seasonality patterns in orders.

Calculated month-on-month order volumes.

Measured % increase in cost of orders between 2017 and 2018 (Jan–Aug).

4. Delivery Performance:

Calculated the actual delivery time (days) vs. purchase date.

Found the top 5 states with fastest delivery compared to estimated delivery dates.

5. Payment Insights:

Analyzed orders placed using different payment types month-on-month.

Counted orders based on number of payment installments.

6. Time-Based Analysis:

Categorized orders placed by time of day (Dawn, Morning, Afternoon, Night).

Determined when Brazilian customers mostly place their orders.


## 15 Practice Questions :

1. **Data type of all columns in the 'customers' table.**

```BigQuery
select *
from `Target_SQL.customers`
limit 5;
```
2. **Get the time range between which the order placed.**
```BigQuery
select 
min(order_purchase_timestamp) as start_time,
max(order_purchase_timestamp) as end_time
from `Target_SQL.orders`;
```
3. **Display the details of Cities and States of customers who orderd during the september and December in 2018.**
```BigQuery
select 
c.customer_city,c.customer_state
from `Target_SQL.orders` as o
join `Target_SQL.customers` as c
on o.customer_id = c.customer_id
where EXTRACT(YEAR FROM o.order_purchase_timestamp)= 2018
AND EXTRACT(MONTH from o.order_purchase_timestamp) BETWEEN 9 AND 12;
```
4. **Is there a growing trend in the no.of orders placed over the past years?.**
```BigQuery
WITH monthly_orders AS (
    SELECT
        DATE_TRUNC(order_purchase_timestamp, MONTH) AS order_month,
        COUNT(order_id) AS no_of_orders
    FROM `Target_SQL.orders`
    GROUP BY order_month
),
monthly_with_trend AS (
    SELECT
        order_month,
        no_of_orders,
        LAG(no_of_orders) OVER (ORDER BY order_month) AS prev_month_orders,
        CASE
            WHEN no_of_orders > LAG(no_of_orders) OVER (ORDER BY order_month)
            THEN 'Growing'
            ELSE 'Not Growing'
        END AS trend_status
    FROM monthly_orders
)
SELECT
    FORMAT_DATE('%Y', order_month) AS year,
    FORMAT_DATE('%m', order_month) AS month,
    no_of_orders,
    prev_month_orders,
    trend_status
FROM monthly_with_trend
ORDER BY order_month;
```
5. **Can we see some kind of monthly seasonlity in terms of the no.of orders being placed?.**

```BigQuery
SELECT
    EXTRACT(YEAR FROM order_purchase_timestamp) AS year,
    EXTRACT(MONTH FROM order_purchase_timestamp) AS month,
    COUNT(order_id) AS no_of_orders
FROM `Target_SQL.orders`
GROUP BY year, month
ORDER BY year, month;
```
6. **During what time of the day, do the Brazillian customers mostly place their orders?.**

```BigQuery
SELECT
    EXTRACT(HOUR FROM order_purchase_timestamp) AS order_hour,
    COUNT(order_id) AS no_of_orders
FROM `Target_SQL.orders` o
JOIN `Target_SQL.customers` c
    ON o.customer_id = c.customer_id
GROUP BY order_hour
ORDER BY no_of_orders DESC;
```
7. **Get the month on month no.of orders placed.**

```BigQuery
select 
EXTRACT(MONTH from order_purchase_timestamp) as month,
EXTRACT(YEAR from order_purchase_timestamp) as year,
COUNT(*) as num_of_orders
from `Target_SQL.orders`
group by year, month
order by year, month;
```

8. ** How are the customers distributed across all the states?.**

```BigQuery
SELECT customer_city,customer_state,
COUNT(DISTINCT customer_id) as customer_cnt
from `Target_SQL.customers`
group by customer_state,customer_city
order by customer_cnt DESC;
```
9. **Get the % increse in the cost of orders from 2017 to 2018 (include months between Jan to Aug only).**

    ```BigQuery
    
    WITH yearly_totals AS (
    SELECT
        EXTRACT(YEAR FROM order_purchase_timestamp) AS year,
        SUM(p.payment_value) AS total_payment
    FROM `Target_SQL.payments` AS p
    JOIN `Target_SQL.orders` AS o
        ON p.order_id = o.order_id
    WHERE EXTRACT(MONTH FROM order_purchase_timestamp) BETWEEN 1 AND 8
      AND EXTRACT(YEAR FROM order_purchase_timestamp) IN (2017, 2018)
    GROUP BY year
),
yearly_comparison AS (
    SELECT
        year,
        total_payment,
        LAG(total_payment) OVER(ORDER BY year) AS prev_year_payment
    FROM yearly_totals
)
SELECT
    year,
    ROUND(((total_payment - prev_year_payment) / prev_year_payment) * 100, 2) AS pct_increase
FROM yearly_comparison
WHERE prev_year_payment IS NOT NULL;

```
10. **Mean and Sum of price and frieght value by customer state**

```BigQuery

SELECT
    c.customer_state,
    ROUND(AVG(oi.price), 2) AS avg_price,
    SUM(oi.price) AS sum_price,
    ROUND(AVG(oi.freight_value), 2) AS avg_freight,
    SUM(oi.freight_value) AS sum_freight
FROM `Target_SQL.orders` AS o
JOIN `Target_SQL.order_items` AS oi
    ON o.order_id = oi.order_id
JOIN `Target_SQL.customers` AS c
    ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY sum_price DESC;
```
11. **ind the no.of days taken to deliver each order from the other's purchase date as delivery time.**
   
   **Also calculate the diffrrence (in days) between the estimated and actual delivery  date of an order.**
```BigQuery
SELECT 
    order_id,
    DATE_DIFF(DATE(order_delivered_customer_date), DATE(order_purchase_timestamp), DAY) AS days_to_delivery,
    DATE_DIFF(DATE(order_delivered_customer_date), DATE(order_estimated_delivery_date), DAY) AS diff_estimated_delivery
FROM `Target_SQL.orders`
WHERE order_delivered_customer_date IS NOT NULL
  AND order_purchase_timestamp IS NOT NULL
  AND order_estimated_delivery_date IS NOT NULL;
```
12. **Find out the top 5 states with the highest and lowest average freight value.**
```BigQuery

-- Top 5 states with highest avg freight value
SELECT 
    c.customer_state,
    AVG(oi.freight_value) AS avg_freight_value
FROM `Target_SQL.orders` AS o
JOIN `Target_SQL.order_items` AS oi
    ON o.order_id = oi.order_id
JOIN `Target_SQL.customers` AS c
    ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY avg_freight_value DESC
LIMIT 5;

-- Top 5 states with lowest avg freight value
SELECT 
    c.customer_state,
    AVG(oi.freight_value) AS avg_freight_value
FROM `Target_SQL.orders` AS o
JOIN `Target_SQL.order_items` AS oi
    ON o.order_id = oi.order_id
JOIN `Target_SQL.customers` AS c
    ON o.customer_id = c.customer_id
GROUP BY c.customer_state
ORDER BY avg_freight_value ASC
LIMIT 5;
```
13. **Find out the top 5 states where the order delivery is really fast as compared to the estimated date of delevery.**
```BigQuery
SELECT 
    c.customer_state,
    AVG(DATE_DIFF(DATE(o.order_estimated_delivery_date), DATE(o.order_delivered_customer_date), DAY)) AS avg_days_early
FROM `Target_SQL.orders` AS o
JOIN `Target_SQL.customers` AS c
    ON o.customer_id = c.customer_id
WHERE o.order_delivered_customer_date IS NOT NULL
  AND o.order_estimated_delivery_date IS NOT NULL
GROUP BY c.customer_state
HAVING avg_days_early > 0   -- only keep states with early deliveries
ORDER BY avg_days_early DESC   -- larger = delivered much earlier
LIMIT 5;
```
14. **Find the month on the month no.of orders placed using different payment type.**

```BigQuery
select 
payment_type,
EXTRACT(YEAR from order_purchase_timestamp) as year,
EXTRACT(month from order_purchase_timestamp) as month,
COUNT(DISTINCT o.order_id) as order_count
from `Target_SQL.orders` as o
INNER JOIN `Target_SQL.payments` as p
ON o.order_id = p.order_id
GROUP BY payment_type,year,month
order by payment_type,year,month;
```
15. **Find the no.of orders place on the bisic of the payment installments that have been paid.**

```BigQuery
select 
count(DISTINCT order_id) as No_of_orders,
payment_installments
from `Target_SQL.payments`
group by payment_installments;
```

---


## Technology Stack
- **Database**: Google BigQuery
- **SQL Queries**: DDL, DML, Aggregations, Joins, Subqueries, Window Functions, Date Functions
- **Tools**: BigQuery Console, Google Cloud Platform (GCP)

## How to Run the Project
1. Open Google BigQuery in your GCP account.
2. Create a new dataset and upload the CSV files (customers, orders, payments, etc.).
3.Set up the tables (schema will be automatically inferred from the CSV headers, or you can define manually).
4.Execute the provided SQL queries in BigQuery Editor.
5.Explore query results and optimize queries using functions like partitioning, clustering, and CTEs for large datasets.

---

## Contributing
If you would like to contribute to this project, feel free to fork the repository, submit pull requests, or raise issues.
