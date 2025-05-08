# OLAP

```sql
CREATE TABLE dim_date (
    order_date DATE,
    day INT,
    month INT,
    year INT,
    quarter INT,
	date_id INT PRIMARY KEY
);
CREATE TABLE dim_customer (
    customer_id VARCHAR(20),
    customer_name VARCHAR(100),
    segment VARCHAR(50),
    city VARCHAR(100),
    state VARCHAR(100),
    region VARCHAR(50),
    country VARCHAR(50),
    postal_code INT,
	customer_key INT PRIMARY KEY
);
CREATE TABLE dim_product (
    product_id VARCHAR(20),
    product_name TEXT,
    category VARCHAR(100),
    sub_category VARCHAR(100),
	product_key INT PRIMARY KEY
);
CREATE TABLE fact_sales (
    order_id VARCHAR(20),
    date_id INT REFERENCES dim_date(date_id),
    customer_key INT REFERENCES dim_customer(customer_key),
    product_key INT REFERENCES dim_product(product_key),
    sales NUMERIC(10, 2),
    quantity INT,
    discount NUMERIC(4, 2),
    profit NUMERIC(10, 2)
);
```




Last login: Sun May  4 15:18:26 on ttys106
majdikleckerova@Marie--MacBook-Air ~ % psql -U postgres -d OLAP
Password for user postgres: 
psql (14.17 (Homebrew), server 17.0)
WARNING: psql major version 14, server major version 17.
         Some psql features might not work.
Type "help" for help.

OLAP=# \copy  dim_product(product_id, product_name, category, sub_category, product_key)
FROM '/Users/majdikleckerova/Desktop/dim_product_pg_clean.csv'
DELIMITER ','
CSV HEADER
QUOTE '"'
ESCAPE '"';
COPY 1894
OLAP=# 


# Postup
1. výběr datové sady (https://www.kaggle.com/datasets/vivek468/superstore-dataset-final)
2. transformace datové sady na hvězdicové schéma (faktová tabulka + 3 dimenze - date, product, customer)
3. vytvoření databáze v PostgreSQL
4. import dat .csv do databáze OLAP v PostgreSQL
5. vytvoření diagramu hvězdicového schéma
6. instalace Metabase přes docker (localhost:3000)
7. připojení databáze OLAP do metabase
8. 4 dotazy + vizualizace


# Dotazy
## 1. Vývoj ziskovosti podle segmentu zákazníků v čase (měsíc + rok)
Zjistit, které zákaznické segmenty (Consumer, Corporate, Home Office) jsou nejziskovější a jak se jejich profit vyvíjel v čase.
```sql
SELECT 
    d.year,
    d.month,
    c.segment,
    SUM(f.profit) AS total_profit
FROM fact_sales f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_date d ON f.date_id = d.date_id
GROUP BY d.year, d.month, c.segment
ORDER BY d.year, d.month, c.segment;
```


## a. Vývoj zisku v čase podle regionu
```sql
SELECT
  d.year,
  c.region,
  SUM(f.profit) AS total_profit
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_customer c ON f.customer_key = c.customer_key
GROUP BY d.year, c.region
ORDER BY d.year, c.region;
```

## b. Zákaznické segmenty a jejich výkonnost
```sql

```

## c. Slevy a jejich vliv na ziskovost
```sql
SELECT
  CASE
    WHEN f.discount BETWEEN 0.00 AND 0.05 THEN '0–5 %'
    WHEN f.discount BETWEEN 0.05 AND 0.10 THEN '5–10 %'
    WHEN f.discount BETWEEN 0.10 AND 0.20 THEN '10–20 %'
    WHEN f.discount BETWEEN 0.20 AND 0.30 THEN '20–30 %'
    WHEN f.discount > 0.30 THEN '30+ %'
    ELSE 'Unknown'
  END AS discount_range,
  CASE
    WHEN f.discount BETWEEN 0.00 AND 0.05 THEN 1
    WHEN f.discount BETWEEN 0.05 AND 0.10 THEN 2
    WHEN f.discount BETWEEN 0.10 AND 0.20 THEN 3
    WHEN f.discount BETWEEN 0.20 AND 0.30 THEN 4
    WHEN f.discount > 0.30 THEN 5
    ELSE 6
  END AS sort_order,
  COUNT(*) AS number_of_sales,
  SUM(f.sales) AS total_sales,
  SUM(f.profit) AS total_profit,
  ROUND(SUM(f.profit) / NULLIF(SUM(f.sales), 0), 2) AS profit_margin
FROM fact_sales f
GROUP BY discount_range, sort_order
ORDER BY sort_order;
```

## d. Regionální výkon podle státu (s filtrem na rok
```sql
SELECT
  c.state,
  d.year,
  p.category,
  SUM(f.sales) AS total_sales,
  SUM(f.profit) AS total_profit,
  COUNT(DISTINCT f.order_id) AS total_orders,

  -- Zaokrouhlené verze metrik
  FLOOR(SUM(f.profit) / 5000) * 5000 AS rounded_profit,
  FLOOR(SUM(f.sales) / 10000) * 10000 AS rounded_sales,
  FLOOR(COUNT(DISTINCT f.order_id) / 50) * 50 AS rounded_orders

FROM fact_sales f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_product p ON f.product_key = p.product_key
WHERE 1=1
[[AND d.year = {{year}}]]
GROUP BY c.state, d.year, p.category
ORDER BY total_profit DESC;
```

## e. Dotaz 5 - Celkový profit v jednotlivých qvartálech jednotlivých let
```sql
SELECT 
    CONCAT(d.year, '-Q', d.quarter) AS year_quarter,
    c.segment,
    ROUND(SUM(f.profit), 2) AS total_profit
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.year, d.quarter, c.segment
ORDER BY d.year, d.quarter, c.segment;
```

## f. Dotaz 6 – 8 Podkategorií s nějvyšším profitem a jejich průměrná sleva, filtr rok
```sql
SELECT 
    p.sub_category,
    ROUND(SUM(f.profit), 2) AS total_profit,
    ROUND(AVG(f.discount) * 100, 1) AS avg_discount_percent
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_date d ON f.date_id = d.date_id
WHERE 1=1
  [[AND d.year = {{year}}]]
GROUP BY p.sub_category
ORDER BY total_profit DESC
LIMIT 8;
```

## g. Města s nejvyšší tržbou podle státu
```sql
SELECT 
    dim_customer.city,
    ROUND(SUM(fact_sales.sales), 2) AS total_sales
FROM fact_sales
JOIN dim_customer ON fact_sales.customer_key = dim_customer.customer_key
WHERE 1=1
  [[AND {{state}}]]
GROUP BY dim_customer.city
ORDER BY total_sales DESC
LIMIT 10;
```
