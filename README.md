# SQL-Project/*
Stores database contains 8 tables
Customers: customer DATABASE
Employees: all employee information
Offices: sales office information
OrderDetails: sales order line for each sales ORDER
Payments: customers' payment record
Products: a list of scale model cars
productLines: a list of product line categories
*/

/*Question 1: Which products should we order more of or less of?
Question 2: How should we tailor marketing and communication strategies to customer behaviors?
Question 3: How much can we spend on acquiring new customers?
*/

SELECT * 
 FROM orderdetails;

 SELECT *
 FROM products
 LIMIT 5;

/*
Select each table name as string
*/

SELECT name AS table_name
FROM sqlite_master
WHERE type = 'table';

/*
Select the numer of columns in each table
*/

SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('customers')
UNION ALL
SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('employees')
UNION ALL
SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('offices')
UNION ALL
SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('orderdetails')
UNION ALL
SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('orders')
UNION ALL
SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('payments')
UNION ALL
SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('productlines')
UNION ALL
SELECT COUNT(*) AS number_of_attributes
FROM pragma_table_info('products');

/*
Select the number of rows in each TABLE
*/

SELECT COUNT(*) AS number_of_rows
FROM customers
UNION ALL
SELECT COUNT(*) AS number_of_rows
FROM products
UNION ALL
SELECT COUNT(*) AS number_of_rows
FROM productlines
UNION ALL
SELECT COUNT(*) AS number_of_rows
FROM orders
UNION ALL
SELECT COUNT(*) AS number_of_rows
FROM orderdetails
UNION ALL
SELECT COUNT(*) AS number_of_rows
FROM payments
UNION ALL
SELECT COUNT(*) AS number_of_rows
FROM employees
UNION ALL
SELECT COUNT(*) AS number_of_rows
FROM offices;

/*
JOIN in 1 TABLE
*/

select 'customers' table_name, count(*) number_of_attributes,(select count(*)from customers ) number_of_rows from pragma_table_info('customers')
UNION ALL
select 'employees' table_name, count(*) number_of_attributes,(select count(*)from employees ) number_of_rows from pragma_table_info('employees')
UNION ALL
select 'offices' table_name, count(*) number_of_attributes,(select count(*)from offices ) number_of_rows from pragma_table_info('offices')
UNION ALL
select 'orderdetails' table_name, count(*) number_of_attributes,(select count(*)from orderdetails ) number_of_rows from pragma_table_info('orderdetails')
UNION ALL
select 'payments' table_name, count(*) number_of_attributes,(select count(*)from payments ) number_of_rows from pragma_table_info('payments')
UNION ALL
select 'products' table_name, count(*) number_of_attributes,(select count(*)from products ) number_of_rows from pragma_table_info('products')
UNION ALL
select 'productlines' table_name, count(*) number_of_attributes,(select count(*)from productlines ) number_of_rows from pragma_table_info('productlines');

/*
 Which Products Should We Order More of or Less of?
 */
 
  
/*
here the low stock for each product is calculated using by a correlated subquery.
It is rounded to hundred thlimeted top ten of products by low stock*/

/*чем больше ratio тем меньше на стоке складских запасов*/

-- 1 option with JOIN

SELECT p.productcode, p.productName, p.productLine, sum(o.quantityOrdered)/p.quantityInStock as lowstock
FROM products as p
JOIN orderdetails as o
ON p.productCode=o.productCode
GROUP BY p.productcode
ORDER BY lowstock DESC;

-- 2 option with subquery

SELECT
    productCode,
    ROUND((SELECT SUM(quantityOrdered) FROM orderdetails WHERE productCode = p.productCode) / p.quantityInStock, 2) AS lowStock
FROM
    products p
	GROUP BY productCode
ORDER BY
    lowStock DESC
LIMIT 10;

/*
Calculate product performance. Quantityordered*priceEach=revenue (выручка). More revenue better for cashflow.
*/
 
 SELECT od.productCode, productName, productLine, sum(quantityordered*priceEach) as product_performance
 FROM orderdetails as od
 JOIN products as p
 ON od.productCode=p.productCode
 GROUP BY od.productCode
 ORDER BY
    product_performance DESC
LIMIT 10;

/*
Previuose queries combined using CTE(common table expression). We are displaying priority products for restocking. 
То есть то, надо вытащить что продается больше всего и имеет меньше запасы.
*/

-- 1 option

WITH CTE_Lowstock as 
(
SELECT p.productCode 
FROM 
(
SELECT p.productcode, p.productName, sum(o.quantityOrdered)/p.quantityInStock as lowstock
FROM products as p
JOIN orderdetails as o
ON p.productCode=o.productCode
GROUP BY p.productcode
ORDER BY lowstock DESC) p), 
CTE_performance as 
(
SELECT p.productCode 
FROM 
(SELECT o.productCode ,sum(quantityordered*priceEach) as product_performance
 FROM orderdetails o
 GROUP BY o.productCode
 ORDER BY
    sum(quantityordered*priceEach) DESC) p)
SELECT p.productCode, p.productName, productLine
FROM products p
WHERE p.productCode IN CTE_Lowstock and p.productCode IN CTE_performance
LIMIT 10;

-- 2nd option

WITH 
tab1 as (
select
	p.productCode
from
	(
	select
		p.productCode ,
		p.productName ,
		sum(o.quantityOrdered)/ p.quantityInStock as low_stock
	from
		products p
	join orderdetails o on
		p.productCode = o.productCode
	group by
		p.productCode
	order by
		low_stock desc) p ),
tab2 as (
select
	p.productCode
from
	(
	select
		p.productCode ,
		p.productName,
		round(sum(o.quantityOrdered * o.priceEach),
		2) as product_performance
	from
		orderdetails o
	join products p on
		p.productCode = o.productCode
	group by
		p.productCode
	order by
		product_performance DESC)p)
SELECT
	p.productCode, p.productName, productLine
from
	products p
WHERE
	p.productCode IN tab1
	AND p.productCode IN tab2
	LIMIT 10;

/*
How Should We Match Marketing and Communication Strategies to Customer Behavior?
VIP customers bring in the most profit for the store.
Less-engaged customers bring in less profit.
*/

-- Joining products, orders and orderdetails

SELECT customerNumber, p.productCode, round(sum(quantityordered*(priceEach-buyPrice)),0) as profit
FROM products as p
JOIN orderdetails as od
ON p.productCode=od.productCode
JOIN orders as o
ON od.orderNumber=o.orderNumber
GROUP BY customerNumber
ORDER BY sum(quantityordered*(priceEach-buyPrice)) DESC;

-- to find the top 5 VIP customers and their details

WITH CTE_top_customers as
(SELECT customerNumber, p.productCode, round(sum(quantityordered*(priceEach-buyPrice)),0) as profit
FROM products as p
JOIN orderdetails as od
ON p.productCode=od.productCode
JOIN orders as o
ON od.orderNumber=o.orderNumber
GROUP BY customerNumber
ORDER BY sum(quantityordered*(priceEach-buyPrice)) DESC
LIMIT 5)
SELECT contactLastName, contactFirstName, city, country, profit
FROM customers as c
JOIN CTE_top_customers as tpc ON c.customerNumber=tpc.customerNumber;

-- to find the top 5 less-engaged customers and their details

WITH CTE_top_customers as
(SELECT customerNumber, p.productCode, round(sum(quantityordered*(priceEach-buyPrice)),0) as profit
FROM products as p
JOIN orderdetails as od
ON p.productCode=od.productCode
JOIN orders as o
ON od.orderNumber=o.orderNumber
GROUP BY customerNumber
ORDER BY sum(quantityordered*(priceEach-buyPrice)) ASC
LIMIT 5)
SELECT contactLastName, contactFirstName, city, country, profit
FROM customers as c
JOIN CTE_top_customers as tpc 
ON c.customerNumber=tpc.customerNumber;

 /*
 How Much Can We Spend on Acquiring New Customers?
*/

--  find the number of new customers arriving each month
-- ищем в процентном отношении сколько новых клиентов
 
 WITH payment_with_year_month_table AS (
  SELECT *,
         CAST(SUBSTRING(paymentDate, 1, 4) AS INTEGER) * 100 + CAST(SUBSTRING(paymentDate, 6, 7) AS INTEGER) AS year_month
  FROM payments
),
customers_by_month_table AS (
  SELECT p1.year_month,
         COUNT(*) AS number_of_customers,
         SUM(p1.amount) AS total
  FROM payment_with_year_month_table p1
  GROUP BY p1.year_month
),
new_customers_by_month_table AS (
  SELECT p1.year_month,
         COUNT(*) AS number_of_new_customers,
         SUM(p1.amount) AS new_customer_total,
         (SELECT number_of_customers
           FROM customers_by_month_table c
          WHERE c.year_month = p1.year_month) AS number_of_customers,
         (SELECT total
           FROM customers_by_month_table c
          WHERE c.year_month = p1.year_month) AS total
  FROM payment_with_year_month_table p1
  WHERE p1.customerNumber NOT IN (SELECT customerNumber
                                  FROM payment_with_year_month_table p2
                                  WHERE p2.year_month < p1.year_month)
  GROUP BY p1.year_month
)
SELECT year_month,
       ROUND(number_of_new_customers * 100 / number_of_customers, 1) AS number_of_new_customers_props,
       ROUND(new_customer_total * 100 / total, 1) AS new_customers_total_props
FROM new_customers_by_month_table;

-- above we see that the number of new clients decreasing. since september 2004 there are no new customers.

/*
to determine how much money we can spend aquiring new customers, we can compute the Customer Lifetime Value (LTV), 
which represents the average amount of money a customer generates. LTV tells us how much profit average customer generates 
during their lifetime. It means if we have 10 new customers next month they may generate USD 390 395.
*/

-- here we compute the average of customer profits using CTE

WITH CTE_profit as
(
SELECT od.productCode, sum((priceEach-buyPrice)*quantityOrdered) as profit, o.customerNumber
FROM orderdetails as od
JOIN products as p
ON p.productCode=od.productCode
JOIN orders as o
ON od.orderNumber=o.orderNumber
GROUP BY o.customerNumber
ORDER BY profit)
SELECT round(avg(profit), 0) as LTV
FROM CTE_profit;