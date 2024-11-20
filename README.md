# Advanced-Data-Analysis-with-SQL
# Advanced-Data-Analysis-with-SQL

## 1. Window Functions and Analytics
-- Customer purchase rankings by category
WITH CustomerPurchases AS (
    SELECT 
        c.customer_id,
        c.customer_name,
        p.category,
        SUM(o.quantity * o.unit_price) as total_spent,
        COUNT(o.order_id) as number_of_orders,
        DENSE_RANK() OVER (PARTITION BY p.category ORDER BY SUM(o.quantity * o.unit_price) DESC) as category_rank
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN products p ON o.product_id = p.product_id
    GROUP BY c.customer_id, c.customer_name, p.category
)
SELECT *
FROM CustomerPurchases
WHERE category_rank <= 3;

-- Moving averages for sales trends
SELECT 
    date_trunc('month', order_date) as month,
    SUM(quantity * unit_price) as monthly_sales,
    AVG(SUM(quantity * unit_price)) OVER (
        ORDER BY date_trunc('month', order_date)
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as three_month_moving_avg
FROM orders
GROUP BY date_trunc('month', order_date)
ORDER BY month;

## 2. Complex Cohort Analysis
-- Customer retention analysis by cohort
WITH FirstPurchase AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) as cohort_month
    FROM orders
    GROUP BY customer_id
),
Purchases AS (
    SELECT 
        f.customer_id,
        f.cohort_month,
        DATE_TRUNC('month', o.order_date) as order_month,
        DATE_DIFF('month', DATE_TRUNC('month', f.cohort_month), 
                          DATE_TRUNC('month', o.order_date)) as month_number
    FROM FirstPurchase f
    JOIN orders o ON f.customer_id = o.customer_id
)
SELECT 
    cohort_month,
    COUNT(DISTINCT customer_id) as cohort_size,
    COUNT(DISTINCT CASE WHEN month_number = 1 THEN customer_id END) as month_1,
    COUNT(DISTINCT CASE WHEN month_number = 2 THEN customer_id END) as month_2,
    COUNT(DISTINCT CASE WHEN month_number = 3 THEN customer_id END) as month_3
FROM Purchases
GROUP BY cohort_month
ORDER BY cohort_month;

## 3. Market Basket Analysis
-- Product affinity analysis
WITH ProductPairs AS (
    SELECT 
        a.order_id,
        a.product_id as product_1,
        b.product_id as product_2,
        p1.product_name as product_1_name,
        p2.product_name as product_2_name
    FROM orders a
    JOIN orders b ON a.order_id = b.order_id
    JOIN products p1 ON a.product_id = p1.product_id
    JOIN products p2 ON b.product_id = p2.product_id
    WHERE a.product_id < b.product_id
)
SELECT 
    product_1_name,
    product_2_name,
    COUNT(*) as combination_count,
    COUNT(*) * 100.0 / (SELECT COUNT(DISTINCT order_id) FROM orders) as percent_of_orders
FROM ProductPairs
GROUP BY product_1_name, product_2_name
HAVING COUNT(*) > 10
ORDER BY combination_count DESC;

## 4. Customer Segmentation
-- RFM Analysis
WITH CustomerMetrics AS (
    SELECT 
        customer_id,
        MAX(order_date) as last_purchase_date,
        COUNT(DISTINCT order_id) as frequency,
        SUM(quantity * unit_price) as monetary
    FROM orders
    GROUP BY customer_id
),
RFM_Scores AS (
    SELECT 
        customer_id,
        NTILE(5) OVER (ORDER BY last_purchase_date) as recency_score,
        NTILE(5) OVER (ORDER BY frequency) as frequency_score,
        NTILE(5) OVER (ORDER BY monetary) as monetary_score
    FROM CustomerMetrics
)
SELECT 
    customer_id,
    recency_score,
    frequency_score,
    monetary_score,
    CASE 
        WHEN (recency_score >= 4 AND frequency_score >= 4 AND monetary_score >= 4) THEN 'Best Customers'
        WHEN (recency_score >= 3 AND frequency_score >= 3 AND monetary_score >= 3) THEN 'Loyal Customers'
        WHEN (recency_score <= 2 AND frequency_score <= 2 AND monetary_score <= 2) THEN 'Lost Customers'
        ELSE 'Average Customers'
    END as customer_segment
FROM RFM_Scores;

## 5. Time Series Analysis
-- Year over Year Growth Analysis
WITH MonthlySales AS (
    SELECT 
        DATE_TRUNC('month', order_date) as sale_month,
        EXTRACT(YEAR FROM order_date) as sale_year,
        SUM(quantity * unit_price) as total_sales
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date), EXTRACT(YEAR FROM order_date)
)
SELECT 
    m1.sale_month,
    m1.total_sales as current_year_sales,
    m2.total_sales as previous_year_sales,
    ((m1.total_sales - m2.total_sales) / m2.total_sales * 100) as yoy_growth
FROM MonthlySales m1
LEFT JOIN MonthlySales m2 
    ON m1.sale_month = (m2.sale_month + INTERVAL '1 year')
ORDER BY m1.sale_month;

## 6. Advanced Pattern Matching
-- Sequential Pattern Analysis
WITH CustomerSequence AS (
    SELECT 
        customer_id,
        order_date,
        product_category,
        LAG(product_category, 1) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date
        ) as previous_category,
        LAG(product_category, 2) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date
        ) as previous_2_category
    FROM orders o
    JOIN products p ON o.product_id = p.product_id
)
SELECT 
    previous_2_category as first_purchase,
    previous_category as second_purchase,
    product_category as third_purchase,
    COUNT(*) as sequence_count
FROM CustomerSequence
WHERE previous_2_category IS NOT NULL
GROUP BY previous_2_category, previous_category, product_category
HAVING COUNT(*) > 5
ORDER BY sequence_count DESC;

## 7. Forecasting and Prediction
-- Simple Sales Forecasting
WITH DailySales AS (
    SELECT 
        DATE_TRUNC('day', order_date) as sale_date,
        SUM(quantity * unit_price) as daily_sales
    FROM orders
    GROUP BY DATE_TRUNC('day', order_date)
),
SalesStats AS (
    SELECT 
        sale_date,
        daily_sales,
        AVG(daily_sales) OVER (
            ORDER BY sale_date
            ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
        ) as avg_7_day,
        STDDEV(daily_sales) OVER (
            ORDER BY sale_date
            ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
        ) as stddev_7_day
    FROM DailySales
)
SELECT 
    sale_date,
    daily_sales,
    avg_7_day as predicted_sales,
    (daily_sales - avg_7_day) as prediction_error,
    (daily_sales - avg_7_day) / stddev_7_day as z_score
FROM SalesStats
WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY sale_date;

## 8. Advanced Aggregation Techniques
-- Pivot Table Implementation
SELECT *
FROM CROSSTAB(
    'SELECT 
        DATE_TRUNC(''month'', order_date)::date as month,
        product_category,
        SUM(quantity * unit_price)
     FROM orders o
     JOIN products p ON o.product_id = p.product_id
     GROUP BY 1, 2
     ORDER BY 1, 2',
    'SELECT DISTINCT product_category 
     FROM products 
     ORDER BY 1'
) AS ct (
    month date,
    category1 numeric,
    category2 numeric,
    category3 numeric
);

## 9. Geographic Analysis
-- Sales Distribution by Region with Hierarchical Rollup
SELECT 
    COALESCE(region, 'TOTAL') as region,
    COALESCE(country, 'TOTAL') as country,
    COALESCE(city, 'TOTAL') as city,
    SUM(quantity * unit_price) as total_sales,
    COUNT(DISTINCT customer_id) as unique_customers,
    SUM(quantity * unit_price) / COUNT(DISTINCT customer_id) as sales_per_customer
FROM orders o
JOIN customer_locations cl ON o.customer_id = cl.customer_id
GROUP BY ROLLUP(region, country, city)
ORDER BY region, country, city;
