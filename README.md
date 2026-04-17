

#  Swiggy Sales Analysis

## Project Overview

This project focuses on analyzing Swiggy food delivery data using **PostgreSQL**. The goal is to clean raw data, design an optimized data model, and generate meaningful business insights through SQL queries.

The analysis helps understand customer behavior, restaurant performance, and overall sales trends.

---

## Business Objectives

* Ensure data quality through cleaning and validation
* Design a scalable **Star Schema** for analytic
* Perform KPI analysis on sales and customer behavior
* Generate actionable insights for business decision-making

---

##  Tools & Technologies

* **Database:** PostgreSQL
* **Language:** SQL
* **Concepts Used:**

  * Data Cleaning
  * Data Validation
  * Window Functions
  * Joins & Aggregations
  * Dimensional Modeling (Star Schema)

---

##  Dataset Description

The dataset contains food delivery records with the following attributes:

* State
* City
* Order Date
* Restaurant Name
* Location
* Category (Cuisine)
* Dish Name
* Price (INR)
* Rating
* Rating Count

---

##  Data Cleaning & Validation

Performed the following steps to ensure data quality:

*  Checked for **NULL values** in key columns
  ```SQL
SELECT
SUM CASE WHEN "State" IS NULL THEN 1 ELSE END) AS null_state,
SUM (CASE WHEN "City" IS NULL THEN 1 ELSE 0 END) AS null_city,
SUM (CASE WHEN "Order Date" IS NULL THEN 1 ELSE END) AS null_order_date,
SUM (CASE WHEN "Restaurant Name" IS NULL THEN 1 ELSE END) AS null restaurant,
SUM (CASE WHEN "Location" IS NULL THEN 1 ELSE 0 END) AS null location,
SUM (CASE WHEN "Category" IS NULL THEN 1 ELSE 0 END) AS null_category,
SUM CASE WHEN "Dish Name" IS NULL THEN 1 ELSE 0 END) AS null_dish,
SUM (CASE WHEN "Price (INR)" IS NULL THEN 1 ELSE 0 END) AS null_price,
SUM (CASE WHEN "Rating" IS NULL THEN 1 ELSE 0 END) AS null_rating,
SUM (CASE WHEN "Rating Count" IS NULL THEN 1 ELSE 0 END) AS null_rating_count
FROM swiggy_data;
```
*  Identified and handled **blank/empty strings**
```SQL

SELECT *
FROM swiggy_data
WHERE
"State" = '' 
OR "City" = '' 
OR "Restaurant_Name" = '' 
OR "Location" = '' 
OR "Category" = '' 
OR "Dish_Name" = '';
```
*  Detected **duplicate records**
```SQL

SELECT 
    state, 
    city, 
    order_date, 
    restaurant_name, 
    location, 
    category,
    dish_name, 
    price_inr, 
    rating, 
    rating_count, 
    COUNT(*) AS cnt
FROM swiggy_data
GROUP BY 
    state, 
    city, 
    order_date, 
    restaurant_name, 
    location, 
    category,
    dish_name, 
    price_inr, 
    rating, 
    rating_count
HAVING COUNT(*) > 1;
``` 
*  Removed duplicates using `ROW_NUMBER()`
```SQL

WITH duplicates AS (
    SELECT 
        ctid,
        ROW_NUMBER() OVER (
            PARTITION BY 
                state, city, order_date, restaurant_name, location, category,
                dish_name, price_inr, rating, rating_count
            ORDER BY ctid
        ) AS rn
    FROM swiggy_data
)

DELETE FROM swiggy_data
WHERE ctid IN (
    SELECT ctid
    FROM duplicates
    WHERE rn > 1
);

```
---

##  Data Modeling (Star Schema)

Designed a **Star Schema** to optimize query performance and simplify analysis.

###  Dimension Tables

* `dim_date` → Year, Month, Quarter, Week
* `dim_location` → State, City, Location
* `dim_restaurant` → Restaurant Name
* `dim_category` → Cuisine
* `dim_dish` → Dish Name

###  Fact Table

* `fact_swiggy_orders`

  * Price
  * Rating
  * Rating Count
  * Foreign Keys to all dimensions

---

## Key Performance Indicators (KPIs)

###  Basic KPIs

* Total Orders
```SQL

SELECT COUNT(*) AS total_orders
FROM fact_swiggy_orders;
```
 
* Total Revenue (INR)
```SQL
SELECT 
    ROUND(SUM(price_inr) / 1000000, 2) || ' INR Million' AS total_revenue
FROM fact_swiggy_orders;
```
* Average Dish Price
```SQL

SELECT
    ROUND(AVG(price_inr), 2) || ' INR' AS avg_dish_price
FROM fact_swiggy_orders;
```
* Average Rating
```SQL

SELECT
    AVG(rating) AS avg_rating
FROM fact_swiggy_orders;
```

---

##  Business Insights

###  Date-Based Analysis

* Monthly order trends
```SQL

SELECT
    d.year,
    d.month,
    d.month_name,
    COUNT(*) AS total_orders
FROM fact_swiggy_orders f
JOIN dim_date d 
    ON f.date_id = d.date_id
GROUP BY
    d.year,
    d.month,
    d.month_name;
```
* Quarterly performance
```SQL


SELECT
    d.year,
    d.quarter,
    COUNT(*) AS total_orders
FROM fact_swiggy_orders f
JOIN dim_date d 
    ON f.date_id = d.date_id
GROUP BY
    d.year,
    d.quarter
ORDER BY
    COUNT(*) DESC;
```
* Year-over-year growth
```SQL
-- Yearly Order Trends

SELECT
    d.year,
    COUNT(*) AS total_orders
FROM fact_swiggy_orders f
JOIN dim_date d 
    ON f.date_id = d.date_id
GROUP BY
    d.year;
```
* Day-of-week patterns
```SQL
-- Orders by Day of Week (Mon–Sun)

SELECT
    TRIM(TO_CHAR(d.full_date, 'Day')) AS day,
    COUNT(*) AS total_orders
FROM fact_swiggy_orders f
JOIN dim_date d 
    ON f.date_id = d.date_id
GROUP BY
    TRIM(TO_CHAR(d.full_date, 'Day')),
    EXTRACT(DOW FROM d.full_date)
ORDER BY
    EXTRACT(DOW FROM d.full_date);
```

###  Location-Based Analysis

* Top 10 cities by orders
```SQL
-- Top 10 Cities by Order Volume

SELECT
    l.city,
    COUNT(*) AS total_orders
FROM fact_swiggy_orders f
JOIN dim_location l
    ON l.location_id = f.location_id
GROUP BY
    l.city
ORDER BY
    COUNT(*) DESC
LIMIT 10;
```
* Revenue contribution by state
```SQL
-- Revenue Contribution by States

SELECT
    l.state,
    SUM(f.price_inr) AS total_revenue
FROM fact_swiggy_orders f
JOIN dim_location l
    ON l.location_id = f.location_id
GROUP BY
    l.state
ORDER BY
    SUM(f.price_inr) DESC;
```

###  Food & Restaurant Performance

* Top 10 restaurants
```SQL
-- Top 10 Restaurants by Orders

SELECT
    r.restaurant_name,
    SUM(f.price_inr) AS total_revenue
FROM fact_swiggy_orders f
JOIN dim_restaurant r
    ON r.restaurant_id = f.restaurant_id
GROUP BY
    r.restaurant_name
ORDER BY
    SUM(f.price_inr) DESC
LIMIT 10;
```
* Most popular categories (Indian, Chinese, etc.)
```SQL
-- Top Categories by Order Volume

SELECT
    c.category,
    COUNT(*) AS total_orders
FROM fact_swiggy_orders f
JOIN dim_category c
    ON c.category_id = f.category_id
GROUP BY
    c.category
ORDER BY
    total_orders DESC;
```
* Most ordered dishes
```SQL
-- Most Ordered Dishes

SELECT
    d.dish_name,
    COUNT(*) AS order_count
FROM fact_swiggy_orders f
JOIN dim_dish d
    ON d.dish_id = f.dish_id
GROUP BY
    d.dish_name
ORDER BY
    order_count DESC;
```
* Cuisine performance (Orders + Ratings)
```SQL
-- Cuisine Performance (Orders + Avg Rating)

SELECT
    c.category,
    COUNT(*) AS total_orders,
    ROUND(AVG(f.rating), 2) AS avg_rating
FROM fact_swiggy_orders f
JOIN dim_category c
    ON c.category_id = f.category_id
GROUP BY
    c.category
ORDER BY
    total_orders DESC;
```

###  Customer Spending Analysis

Customer spending grouped into buckets:

* Under ₹100
* ₹100–199
* ₹200–299
* ₹300–499
* ₹500+
```SQL
-- Total Orders By Price Range

WITH price AS (
    SELECT
        CASE
            WHEN price_inr < 100 THEN 'Under 100'
            WHEN price_inr BETWEEN 100 AND 199 THEN '100 - 199'
            WHEN price_inr BETWEEN 200 AND 299 THEN '200 - 299'
            WHEN price_inr BETWEEN 300 AND 499 THEN '300 - 499'
            ELSE '500+'
        END AS price_range
    FROM fact_swiggy_orders
)

SELECT
    price_range,
    COUNT(*) AS total_orders
FROM price
GROUP BY
    price_range
ORDER BY
    total_orders;
```

###  Ratings Analysis

* Distribution of ratings (1–5)
```SQL
-- Rating Count Distribution (1–5)

SELECT
    rating,
    COUNT(*) AS rating_count
FROM fact_swiggy_orders
GROUP BY
    rating
ORDER BY
    rating;
```

---

##  Key Learnings

* Built a complete **end-to-end SQL analytics project**
* Gained hands-on experience with **data modeling (Star Schema)**
* Improved skills in **writing optimized SQL queries**
* Learned how to convert raw data into **business insights**

---
