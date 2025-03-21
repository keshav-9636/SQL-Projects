# 🛒 Grocery Sales Database  

## 📌 Overview  
The **Grocery Sales Database** is a structured relational dataset designed for analyzing **sales transactions, customer demographics, product details, employee records, and geographical information** across multiple cities and countries.  

---

## 🔗 Dataset Information  
📥 Please find all the details regarding the dataset through this [link](https://www.kaggle.com/datasets/andrexibiza/grocery-sales-dataset).  

---

## 📜 Entity Relationship Diagram

![](https://github.com/user-attachments/assets/18b86912-f269-4dd4-9955-5e74f7ae6b22)

### 🌐 Data Relationships  

- **🛍️ Sales:** Each sale is linked to a **Product, Customer, and Employee** through their respective IDs. Additionally, each sale is associated with a **location** via the customer.  
- **👥 Customers:** Connected to a **City** and a **Country** to provide geographic context.  
- **🧑‍💼 Employees:** Responsible for managing sales, uniquely identified by **EmployeeID**.  
- **📦 Products:** Organized into **Categories** to structure the inventory effectively.  
- **🌎 Geography:** Cities belong to **Countries**, enabling higher-level geographic segmentation.  

---

## Questions and Answers

### 1. Sales Performance & Revenue Analysis

1.1 What is the total revenue generated in each month?

```sql
SELECT 
    MONTHNAME(sales_date) AS sales_month,
    MONTH(s.sales_date) AS month_number,
    SUM(s.total_price) AS revenue
FROM sales s
GROUP BY sales_month, month_number
ORDER BY month_number;
```

1.2 Which products contribute the most to revenue?

```sql
SELECT 
	DISTINCT p.product_name,
	p.product_id,
	SUM(s.total_price) AS revenue
FROM sales s
JOIN products p
USING (product_id)
GROUP BY p.product_name, p.product_id
ORDER BY revenue DESC
LIMIT 5;
```

1.3 How do discount levels impact total sales?

```sql
SELECT
	DISTINCT s.discount,
    COUNT(*) AS total_orders,
    SUM(s.quantity) AS total_quantity,
    ROUND(SUM(s.total_price),2) AS revenue
FROM sales s
GROUP BY s.discount
ORDER BY s.discount, total_orders, total_quantity, revenue;
```

1.4 What are the highest revenue product categories?

```sql
SELECT
	c.category_id,
    c.category_name,
    SUM(s.total_price) AS revenue
FROM sales s
JOIN products p
USING (product_id)
JOIN categories c
USING (category_id)
GROUP BY c.category_id, c.category_name
ORDER BY revenue DESC
LIMIT 5;
```

1.5 What is the average order value per transaction?

```sql
SELECT ROUND(AVG(total_price),2)
FROM sales;
```


1.6 Which cities/countries generate the highest revenue?

```sql
SELECT 
	co.country_name,
    co.country_code,
    ci.city_name,
    SUM(s.total_price) AS revenue
FROM cities ci
JOIN sales s
USING (city_id)
JOIN countries co
ON ci.country_id = co.country_id
GROUP BY co.country_name, co.country_code, ci.city_name
ORDER BY revenue DESC;
```

### 2. Customer Insights & Segmentation

2.1 Who are the top customers based on total spending?

```sql
SELECT
	c.customer_id,
    IF(middle_initials IS NULL,
		CONCAT(c.first_name," ",c.last_name),
        CONCAT(c.first_name," ",c.middle_initials," ",c.last_name)) AS customer_name,
    SUM(s.total_price) AS revenue
FROM customers c
JOIN sales s
USING (customer_id)
GROUP BY c.customer_id, customer_name
ORDER BY revenue;
```

2.2 What is the average number of purchases per customer?

```sql
SELECT
	ROUND(COUNT(DISTINCT transaction_number)/COUNT(DISTINCT customer_id),1) as average_number_of_purchases
FROM sales;
```

2.3 Which cities have the highest concentration of above average loyal customers?

```sql
WITH customer_orders AS (
    SELECT
        s.customer_id,
        c.city_name,
        COUNT(s.transaction_number) AS purchase_count
    FROM sales s
    JOIN cities c USING (city_id)
    GROUP BY s.customer_id, c.city_name
),
average_purchases AS (
    SELECT ROUND(COUNT(DISTINCT transaction_number) * 1.0 / COUNT(DISTINCT customer_id), 1) AS avg_purchases
    FROM sales
)
SELECT
    co.city_name,
    COUNT(DISTINCT co.customer_id) AS number_of_customers,
    SUM(CASE WHEN co.purchase_count >= ap.avg_purchases THEN 1 ELSE 0 END) AS loyal_customers,
    ROUND(SUM(CASE WHEN co.purchase_count >= ap.avg_purchases THEN 1 ELSE 0 END) * 1.0 / COUNT(DISTINCT co.customer_id), 2) AS loyalty_pct
FROM customer_orders co
JOIN average_purchases ap ON 1 = 1  -- Avoids CROSS JOIN overhead
GROUP BY co.city_name
ORDER BY loyalty_pct DESC
LIMIT 10;
```

2.4 What is the customer retention rate based on repeat purchases monthly?

```sql
WITH monthly_sales AS (
    SELECT 
        customer_id, 
        DATE_FORMAT(sales_date, '%Y-%m') AS sales_month
    FROM sales
    GROUP BY customer_id, sales_month
),
CustomerRetention AS (
    SELECT 
        a.sales_month AS current_month, 
        COUNT(DISTINCT a.customer_id) AS total_customers,
        COUNT(DISTINCT CASE WHEN b.customer_id IS NOT NULL THEN a.customer_id END) AS repeat_customers,
        ROUND((COUNT(DISTINCT CASE WHEN b.customer_id IS NOT NULL THEN a.customer_id END) * 100.0) / 
              COUNT(DISTINCT a.customer_id), 2) AS retention_rate
    FROM monthly_sales a
    LEFT JOIN monthly_sales b 
        ON a.customer_id = b.customer_id 
        AND DATE_FORMAT(DATE_SUB(STR_TO_DATE(a.sales_month, '%Y-%m'), INTERVAL 1 MONTH), '%Y-%m') = b.sales_month
    GROUP BY a.sales_month
    ORDER BY a.sales_month
)
SELECT * FROM CustomerRetention;
```

2.5 Which city has the highest number of transactions?

```sql
SELECT
	s.city_id,
    c.city_name,
    COUNT(DISTINCT transaction_number) AS number_of_transactions
FROM sales s
JOIN cities c
USING (city_id)
GROUP BY s.city_id, c.city_name
ORDER BY number_of_transactions DESC
LIMIT 5;
```

### 3. Sales Representative & Employee Performance

3.1 Who are the top-performing sales employees based on total sales?

```sql
SELECT 
	e.employee_id,
    IF(middle_initial IS NULL,
		CONCAT(e.first_name," ",e.last_name),
        CONCAT(e.first_name," ",e.middle_initial," ",e.last_name)) AS employee_name,
    ROUND(SUM(s.total_price),2) AS revenue
FROM employees e
JOIN sales s
ON e.employee_id = s.salesperson_id
GROUP BY e.employee_id, employee_name
ORDER BY revenue;
```

3.2 What is the average revenue generated per employee?

```sql
SELECT
    ROUND((SUM(total_price)/COUNT(DISTINCT salesperson_id)),2) AS average_revenue
FROM sales;
```

3.3 Which sales employee has the highest number of customers?

```sql
SELECT
	e.employee_id,
    IF(middle_initial IS NULL,
		CONCAT(e.first_name," ",e.last_name),
        CONCAT(e.first_name," ",e.middle_initial," ",e.last_name)) AS employee_name,
	COUNT(DISTINCT s.customer_id) AS customers
FROM sales s
JOIN employees e
ON e.employee_id = s.salesperson_id
GROUP BY e.employee_id, employee_name;
```

3.4 How does an employee's tenure (hire date vs. today) correlate with sales performance?

```sql
CREATE TEMPORARY TABLE tenure_date AS 
	SELECT
		e.employee_id,
        TIMESTAMPDIFF(YEAR, e.hire_date, CURDATE()) AS tenure_years,
        COALESCE(SUM(s.total_price),0) AS revenue
	FROM sales s
    RIGHT JOIN employees e			-- we use right join because there could be salesmen who have not sold anything which can tell us about their performance.
    ON s.salesperson_id = e.employee_id
    GROUP BY e.employee_id;
CREATE TEMPORARY TABLE stats_for_corr AS
	SELECT
		COUNT(*) AS N,
        SUM(tenure_years) AS sum_x,
        SUM(revenue) AS sum_y,
        SUM(tenure_years * revenue) AS sum_xy,
        SUM(tenure_years * tenure_years) AS sum_x2,
        SUM(revenue * revenue) AS sum_y2
	FROM tenure_date;
SELECT 
	((N * sum_xy) - (sum_x * sum_y))/(SQRT((N * sum_x2 - sum_x * sum_x) * ((N * sum_y2) - (sum_y * sum_y)))) AS correlation_coefficient
FROM stats_for_corr;
```

### 4. Product & Inventory Management

4.1 What are the top-selling and least-selling products?

```sql
SELECT 
	DISTINCT p.product_name,
	p.product_id,
	SUM(s.quantity) AS sold_quantity
FROM sales s
JOIN products p
USING (product_id)
GROUP BY p.product_name, p.product_id
ORDER BY sold_quantity DESC
LIMIT 5;

SELECT 
	DISTINCT p.product_name,
	p.product_id,
	SUM(s.quantity) AS sold_quantity
FROM sales s
JOIN products p
USING (product_id)
GROUP BY p.product_name, p.product_id
ORDER BY sold_quantity ASC
LIMIT 5;
```

4.2 How does product pricing impact sales volume?

```sql
CREATE TEMPORARY TABLE stats AS
	SELECT
		COUNT(*) AS N,
        SUM(p.price) AS sum_x,
        SUM(s.total_price) AS sum_y,
        SUM(p.price * s.total_price) AS sum_xy,
        SUM(p.price * p.price) AS sum_x2,
        SUM(s.total_price * s.total_price) AS sum_y2
	FROM sales s
    JOIN products p
    USING (product_id);
SELECT 
	((N * sum_xy) - (sum_x * sum_y))/(SQRT((N * sum_x2 - sum_x * sum_x) * ((N * sum_y2) - (sum_y * sum_y)))) AS correlation_coefficient
FROM stats;
```

4.3 Which product categories have the highest and lowest sales volume?

```sql
SELECT 
	DISTINCT c.category_id,
	c.category_name,
	CAST(SUM(s.total_price) AS FLOAT) AS revenue
FROM sales s
JOIN products p
USING (product_id)
JOIN categories c
ON c.category_id = p.product_id
GROUP BY c.category_id, c.category_name
ORDER BY revenue DESC
LIMIT 5;
```

### 5. Geographic & Market Expansion Strategy

5.1 Which cities have the highest and lowest sales performance?

```sql
(
    SELECT 
        c.city_id,
        c.city_name,
        ROUND(SUM(s.total_price), 2) AS revenue,
        'Highest' AS sales_category
    FROM sales s
    JOIN cities c USING (city_id)
    GROUP BY c.city_id, c.city_name
    ORDER BY revenue DESC
    LIMIT 5
)
UNION
(
    SELECT 
        c.city_id,
        c.city_name,
        ROUND(SUM(s.total_price), 2) AS revenue,
        'Lowest' AS sales_category
    FROM sales s
    JOIN cities c USING (city_id)
    GROUP BY c.city_id, c.city_name
    ORDER BY revenue ASC
    LIMIT 5
)
ORDER BY revenue DESC;
```

5.2 What are the spending patterns of customers in different cities?

```sql
WITH customer_spending AS (
    SELECT 
        s.customer_id,
        ct.city_name,
        SUM(s.total_price) AS total_spent
    FROM sales s
    JOIN customers c 
    USING (customer_id)
    JOIN cities ct 
    ON s.city_id = ct.city_id
    GROUP BY s.customer_id, ct.city_name
),
spending_segments AS (
    SELECT 
        cs.customer_id,
        cs.city_name,
        cs.total_spent,
        CASE 
            WHEN cs.total_spent >= (SELECT AVG(total_spent) FROM customer_spending) 
            THEN 'High Spender'
            ELSE 'Low Spender'
        END AS spending_category
    FROM customer_spending cs
)
SELECT 
    city_name, 
    spending_category, 
    COUNT(customer_id) AS customer_count, 
    SUM(total_spent) AS total_spent_per_segment, 
    AVG(total_spent) AS avg_spent_per_customer
FROM spending_segments
GROUP BY city_name, spending_category
ORDER BY city_name, spending_category DESC;
```

5.3 Are there specific locations where discounts drive higher sales?

```sql
WITH sales_data AS (
    SELECT 
        c.city_id,
        c.city_name,
        COUNT(s.transaction_number) AS total_orders,
        SUM(s.total_price) AS total_revenue,
        COUNT(CASE WHEN s.discount > 0 THEN s.transaction_number END) AS discounted_orders,
        SUM(CASE WHEN s.discount > 0 THEN s.total_price END) AS discount_revenue,
        AVG(CASE WHEN s.discount > 0 THEN s.total_price END) AS avg_discount_order,
        AVG(s.total_price) AS avg_order_value
    FROM sales s
    JOIN cities c 
    USING (city_id)
    GROUP BY c.city_id, c.city_name
)
SELECT 
    city_id,
    city_name,
    total_orders,
    total_revenue,
    discounted_orders,
    discount_revenue,
    ROUND((discounted_orders * 100.0) / NULLIF(total_orders, 0), 2) AS discount_order_pct,
    ROUND((discount_revenue * 100.0) / NULLIF(total_revenue, 0), 2) AS discount_revenue_pct,
    avg_discount_order,
    avg_order_value
FROM sales_data
ORDER BY discount_revenue_pct DESC;
```

5.4 Which cities have the fastest-growing sales?

```sql
WITH monthly_sales AS (
	SELECT
		c.city_id,
        c.city_name,
        DATE_FORMAT(s.sales_date, '%Y-%m') AS month,
        SUM(s.total_price) AS total_revenue
	FROM sales s
    JOIN cities c 
    USING (city_id)
    GROUP BY c.city_id, c.city_name, month
),
sales_growth AS (
	SELECT
		ms.city_id,
        ms.city_name,
        ms.month,
        ms.total_revenue,
        LAG(ms.total_revenue) OVER (PARTITION BY ms.city_id ORDER BY ms.month) AS prev_month_revenue,
        CASE 
            WHEN LAG(ms.total_revenue) OVER (PARTITION BY ms.city_id ORDER BY ms.month) IS NOT NULL
            THEN ROUND(((ms.total_revenue - LAG(ms.total_revenue) OVER (PARTITION BY ms.city_id ORDER BY ms.month)) 
                        / LAG(ms.total_revenue) OVER (PARTITION BY ms.city_id ORDER BY ms.month)) * 100, 2)
			ELSE NULL
		END AS growth_rate
	FROM monthly_sales ms
)
SELECT 
    city_id,
    city_name,
    month,
    total_revenue,
    prev_month_revenue,
    growth_rate
FROM sales_growth
WHERE growth_rate IS NOT NULL
ORDER BY growth_rate DESC; 
```

### 6. Transaction & Discount Impact

6.1 What percentage of transactions involve a discount?

```sql
SELECT 
    COUNT(CASE WHEN discount > 0 THEN transaction_number END) AS discounted_transactions,
    COUNT(transaction_number) AS total_transactions,
    ROUND((COUNT(CASE WHEN discount > 0 THEN sale_id END) * 100.0) / COUNT(sale_id), 2) AS discount_transaction_pct
FROM sales;
```
