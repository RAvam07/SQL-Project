# SQL-Project

#### Stores database contains 8 tables
Customers: customer DATABASE  
Employees: all employee information  
Offices: sales office information  
OrderDetails: sales order line for each sales ORDER  
Payments: customers' payment record  
Products: a list of scale model cars  
ProductLines: a list of product line categories


#### Question 1: Which products should we order more of or less of?
#### Question 2: How should we tailor marketing and communication strategies to customer behaviors?
#### Question 3: How much can we spend on acquiring new customers?

```sql
SELECT * 
 FROM orderdetails;

 SELECT *
 FROM products
 LIMIT 5;
```
*Select each table name as string*
```sql
SELECT name AS table_name
FROM sqlite_master
WHERE type = 'table';
```

*Select the numer of columns in each table*

```sql SELECT COUNT(*) AS number_of_attributes
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
```

*Select the number of rows in each TABLE*

```sql SELECT COUNT(*) AS number_of_rows
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
```

*JOIN in 1 TABLE*

```sql select 'customers' table_name, count(*) number_of_attributes,(select count(*)from customers ) number_of_rows from pragma_table_info('customers')
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
```

### Which Products Should We Order More of or Less of?
 
*here the low stock for each product is calculated using by a correlated subquery.
It is rounded to hundred thlimeted top ten of products by low stock*

*higher ratio less inventory*

*1 option with JOIN*

```sql SELECT p.productcode, p.productName, p.productLine, sum(o.quantityOrdered)/p.quantityInStock as lowstock
FROM products as p
JOIN orderdetails as o
ON p.productCode=o.productCode
GROUP BY p.productcode
ORDER BY lowstock DESC;
```

*2 option with subquery*

```sql SELECT
    productCode,
    ROUND((SELECT SUM(quantityOrdered) FROM orderdetails WHERE productCode = p.productCode) / p.quantityInStock, 2) AS lowStock
FROM
    products p
	GROUP BY productCode
ORDER BY
    lowStock DESC
LIMIT 10;
```

*Calculate product performance. Quantityordered*priceEach=revenue (выручка). More revenue better for cashflow.*
 
 ```sql SELECT od.productCode, productName, productLine, sum(quantityordered*priceEach) as product_performance
 FROM orderdetails as od
 JOIN products as p
 ON od.productCode=p.productCode
 GROUP BY od.productCode
 ORDER BY
    product_performance DESC
LIMIT 10;
```

*Previuose queries combined using CTE(common table expression). We are displaying priority products for restocking. 
То есть то, надо вытащить что продается больше всего и имеет меньше запасы.*

*1 option*

```sql WITH CTE_Lowstock as 
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
```

*2nd option*

```sql WITH 
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
```

*How Should We Match Marketing and Communication Strategies to Customer Behavior?
VIP customers bring in the most profit for the store.
Less-engaged customers bring in less profit.*

*Joining products, orders and orderdetails*

```sql SELECT customerNumber, p.productCode, round(sum(quantityordered*(priceEach-buyPrice)),0) as profit
FROM products as p
JOIN orderdetails as od
ON p.productCode=od.productCode
JOIN orders as o
ON od.orderNumber=o.orderNumber
GROUP BY customerNumber
ORDER BY sum(quantityordered*(priceEach-buyPrice)) DESC;
```

*to find the top 5 VIP customers and their details*

```sql WITH CTE_top_customers as
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
```

*to find the top 5 less-engaged customers and their details*

```sql WITH CTE_top_customers as
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
```

 *How Much Can We Spend on Acquiring New Customers?*

*find the number of new customers arriving each month*

*ищем в процентном отношении сколько новых клиентов*
 
 ```sql WITH payment_with_year_month_table AS (
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
```

*Above we see that the number of new clients decreasing. since september 2004 there are no new customers.*

*To determine how much money we can spend aquiring new customers, we can compute the Customer Lifetime Value (LTV), 
which represents the average amount of money a customer generates. LTV tells us how much profit average customer generates 
during their lifetime. It means if we have 10 new customers next month they may generate USD 390 395.*

*here we compute the average of customer profits using CTE*

```sql WITH CTE_profit as
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
```
# CONCLUSION

*1.	Which products should we order more of or less of? 
Answer is: Classic models and Motorcycles should be ordered more to restock and keep up with sales volume.*

*Classic cars and motorcycles are sold better:*

| No | productCode | productName | productLine | product_performance | 
|---|---|---|---|---|
| 1 | S18_3232 | 1992 Ferrari 360 Spider red | Classic Cars | 276839.98 |
| 2 | S12_1108 | 2001 Ferrari Enzo | Classic Cars | 190755.86 |
| 3 | S10_1949 | 1952 Alpine Renault 1300 | Classic Cars | 190017.96 |
| 4 | S10_4698 | 2003 Harley-Davidson Eagle Drag Bike | Motorcycles | 170696.0 |
| 5 | S12_1099 | 1968 Ford Mustang | Classic Cars | 161531.48 |
| 6 | S12_3891 | 1969 Ford Falcon | Classic Cars | 152543.02 |
| 7 | S18_1662 | 1980s Black Hawk  Helicopter | Planes | 144959.91 |
| 8 | S18_2238 | 1998 Chrysler Plymouth Prowler | Classic Cars | 142530.63 |
| 9 | S18_1749 | 1917 Grand Touring Sedan | Vintage Cars | 140535.6 |
| 10 | S12_2823 | 2002 Suzuki XREO | Motorcycles | 135767.03 |

*Low stock ratio shows products with low inventory. As more ratio then less inventory:*

| No | productCode | productName | productLine | lowstock | 
|---|---|---|---|---|
| 1 | S24_2000 | 1960 BSA Gold Star DBD34 | Motorcycles | 67|
| 2 | S12_1099 | 1968 Ford Mustang | Classic Cars | 13 |
| 3 | S32_4289 | 1968 Ford Phaeton Delux | Vintage Cars | 7 |
| 4 | S32_1374 | 1997 BMW F650 ST | Motorcycles | 5 |
| 5 | S72_3212 | Pont Yacht | Ships | 2 |

*And here is combined dataset where we can see the priority for restocking. *

| No | productCode | productName | productLine |
|---|---|---|---|
| 1 | S10_1949 | 1952 Alpine Renault 1300 | Classic Cars |
| 2 | S10_4698 | 2003 Harley-Davidson Eagle Drag Bike | Motorcycles |
| 3 | S12_1199 | 1968 Ford Mustang | Classic Cars |
| 4 | S12_1108 | 2001 Ferrari Enzo | Classic Cars |
| 5 | S12_2823 | 2002 Suzuki XREO | Motorcycles |
| 6 | S12_3891 | 1969 Ford Falcon | Classic Cars |
| 7 | S18_1662 | 1980s Black Hawk Helicopter | Planes |
| 8 | S18_1749 | 1917 Grand Touring Sedan | Classic Cars |
| 9 | S18_2238 | 1998 Chrysler Plymouth Prowler | Classic Cars |
| 10 | S18_3232 | 1992 Ferrari 360 Spider red | Classic Cars |	

*2.	 How should we tailor marketing and communication strategies to customer behaviors?
Answer: we see how much profit comes from VIP and less engaged Customers.*

*TOP 5 VIP Customers:*

| No | contactLastName | contactFirstName | city | country | profit |
|---|---|---|---|---|---|
| 1 | Freyre | Diego | Madrid | Spain | 326520.0 |
| 2 | Nelson | Susan | San Rafael | USA | 236769.0 |
| 3 | Young | Jeff | NYC | USA | 72370.0
| 4 | Ferguson | Peter | Melbourn | Australia | 70311.0 |
| 5 | Labrune | Janine | Nantes | France | 60875.0 |

*5 Less engaged Customers:*
