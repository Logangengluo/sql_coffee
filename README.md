# Monday Coffee Expansion SQL Project

![Company Logo](https://github.com/najirh/Monday-Coffee-Expansion-Project-P8/blob/main/1.png)

## Objective
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

## Key Questions
1. **Coffee Consumers Count**  
   How many people in each city are estimated to consume coffee, given that 25% of the population does?
```sql
SELECT city_name, 
		ROUND(
		population*0.25/1000000,
		2) as consumption_in_millions,
		city_rank
FROM city
ORDER BY 2 DESC;
```
2. **Total Revenue from Coffee Sales**  
   What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?
```sql
SELECT SUM(total) AS sales_Q4
FROM sales
WHERE 
	EXTRACT(YEAR FROM sale_date)  = 2023
	AND
	EXTRACT(quarter FROM sale_date) = 4;
```
3. **Sales Count for Each Product**  
   How many units of each coffee product have been sold?
```sql
SELECT 
		product_name, 
		COUNT(total)
FROM products as p
RIGHT JOIN 
sales as s
ON s.product_id = p.product_id
GROUP BY  1
ORDER BY 2 DESC;
```
4. **Average Sales Amount per City**  
   What is the average sales amount per customer in each city?
```sql
SELECT city_name, ROUND((sum(total)/count(distinct cst.customer_id))::numeric,2) as sales_per_customers
FROM sales as s
LEFT JOIN 
customers as cst
ON  cst.customer_id  = s.customer_id
JOIN
city as ct
ON ct.city_id = cst.city_id
GROUP BY 1
ORDER BY 2 DESC;
```
5. **City Population and Coffee Consumers**  
   Provide a list of cities along with their populations and estimated coffee consumers.
```sql
WITH city_table as 
(
	SELECT 
		city_name,
		ROUND((population * 0.25)/1000000, 2) as coffee_consumers
	FROM city
),
customers_table
AS
(
	SELECT 
		ci.city_name,
		COUNT(DISTINCT c.customer_id) as unique_cx
	FROM sales as s
	JOIN customers as c
	ON c.customer_id = s.customer_id
	JOIN city as ci
	ON ci.city_id = c.city_id
	GROUP BY 1
)
SELECT 
	customers_table.city_name,
	city_table.coffee_consumers as coffee_consumer_in_millions,
	customers_table.unique_cx
FROM city_table
JOIN 
customers_table
ON city_table.city_name = customers_table.city_name
ORDER BY 2 DESC;
```
6. **Top Selling Products by City**  
   What are the top 3 selling products in each city based on sales volume?
```sql
SELECT *
FROM 
(
SELECT 
	city.city_name, 
	p.product_name,
	count(s.sale_id),
	DENSE_RANK() OVER (PARTITION BY city.city_name ORDER BY count(s.sale_id) DESC) as consumption_rank
FROM sales as s
LEFT JOIN products as p
ON p.product_id = s.product_id
LEFT JOIN customers as c
ON c.customer_id = s.customer_id
LEFT JOIN city 
ON city.city_id = c.city_id
GROUP BY city.city_name, p.product_name
ORDER BY 1,4
)
WHERE consumption_rank <= 3;
```
7. **Customer Segmentation by City**  
   How many unique customers are there in each city who have purchased coffee products?
```sql
SELECT 
	city_name, 
	count(DISTINCT c.customer_id)
FROM sales as s
LEFT JOIN customers as c
ON c.customer_id = s.customer_id
LEFT JOIN city as ct
ON ct.city_id = c.city_id
WHERE 
	s.product_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14)
GROUP BY 1
ORDER BY 2 DESC;
```
8. **Average Sale vs Rent**  
   Find each city and their average sale per customer and avg rent per customer
```sql
SELECT 
	ct.city_name, 
	SUM(total) as total_sale,
	COUNT(DISTINCT c.customer_id),
	ROUND((
	SUM(total)/COUNT(DISTINCT c.customer_id)
	)::numeric,2) as avg_sale_per_customer,
	ROUND((
	AVG(estimated_rent) / COUNT(DISTINCT c.customer_id)
	)::numeric,2) as avg_rent_per_customer
FROM sales as s
LEFT JOIN customers as c
ON c.customer_id = s.customer_id
LEFT JOIN city as ct
ON ct.city_id = c.city_id
GROUP BY 1
ORDER BY 4 DESC;
```
9. **Monthly Sales Growth**  
   Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly).
```sql
WITH tb1 AS(
SELECT 
	ct.city_name,
 	EXTRACT(year FROM sale_date) as year,
	EXTRACT(month FROM sale_date) as month,
	SUM(total) as current_sales,
	LAG(SUM(total),1) OVER (PARTITION BY ct.city_name ORDER BY EXTRACT(year FROM sale_date), EXTRACT(month FROM sale_date)) as previous_sales
FROM sales as s
LEFT JOIN customers as c
ON c.customer_id = s.customer_id
LEFT JOIN city as ct
ON ct.city_id = c.city_id
GROUP BY 1,2,3
ORDER BY 2,3
)
SELECT 
	city_name,
	year,
	month,
	current_sales,
	previous_sales,
	ROUND(((current_sales - previous_sales)*100/previous_sales)::numeric,2) as growth_rate
FROM tb1
WHERE previous_sales IS NOT NULL
ORDER BY 1,2,3;
```
10. **Market Potential Analysis**  
    Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated  coffee consumer
```sql
SELECT 
	ct.city_name,
	ROUND(SUM(total/1000000)::numeric,2) as total_sales_millions,
	ROUND(SUM(estimated_rent/1000000)::numeric,2) as total_rent_millions,
	COUNT(distinct s.customer_id) as total_customers,
	ROUND(AVG(ROUND((population * 0.25)/1000000, 2)),2)::numeric as coffee_consum
FROM sales as s
LEFT JOIN customers as c
ON c.customer_id = s.customer_id
LEFT JOIN city as ct
ON ct.city_id = c.city_id
GROUP BY ct.city_name
ORDER BY 2 DESC
LIMIT 3;
```    

## Recommendations
After analyzing the data, the recommended top three cities for new store openings are:

**City 1: Pune**  
1. Average rent per customer is very low.  
2. Highest total revenue.  
3. Average sales per customer is also high.

**City 2: Delhi**  
1. Highest estimated coffee consumers at 7.7 million.  
2. Highest total number of customers, which is 68.  
3. Average rent per customer is 330 (still under 500).

**City 3: Jaipur**  
1. Highest number of customers, which is 69.  
2. Average rent per customer is very low at 156.  
3. Average sales per customer is better at 11.6k.

---
