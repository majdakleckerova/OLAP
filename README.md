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
