# E-commerce Database Project

This document outlines the database schema, relationships, and key business-intelligence queries for a sample e-commerce platform. It also includes an exploration of database optimization techniques such as denormalization and an example SQL schema script.

## Table of Contents
- Database Schema
	- 1.1 Entity Descriptions
	- 1.2 SQL Schema Script
- Entity-Relationship (ER) Model
	- 2.1 Relationship Identification
	- 2.2 ER Diagram
- SQL Queries for Business Intelligence
	- 3.1 Daily Revenue Report
	- 3.2 Monthly Top-Selling Products
	- 3.3 High-Value Customers (Past Month)
- Database Optimization: Denormalization
	- 4.1 Concept Overview
	- 4.2 Application Example: Customer and Order Entities

---

## 1. Database Schema

### 1.1 Entity Descriptions
The initial CategoryProduct entity has been correctly normalized into two separate entities: `Category` and `Product`, to maintain a proper relational structure.

- **Category**: Stores product categories.
	- `category_id`: Unique identifier for the category.
	- `category_name`: Name of the category (e.g., 'Electronics', 'Books').

- **Product**: Stores information about individual products.
	- `product_id`: Unique identifier for the product.
	- `category_id`: Foreign key linking to the `Category` table.
	- `name`: Name of the product.
	- `description`: Detailed description of the product.
	- `price`: The selling price of the product.
	- `stock_quantity`: The number of units available in inventory.

- **Customer**: Stores customer account information.
	- `customer_id`: Unique identifier for the customer.
	- `first_name`: Customer's first name.
	- `last_name`: Customer's last name.
	- `email`: Customer's email address (should be unique).
	- `password`: Hashed password for customer authentication.

- **Order**: Stores high-level information about a customer's order. Note: `Order` is a reserved keyword in some SQL dialects; use quoting/backticks or choose an alternative name like `Orders`.
	- `order_id`: Unique identifier for the order.
	- `customer_id`: Foreign key linking to the `Customer` table.
	- `order_date`: The date and time the order was placed.
	- `total_amount`: The total cost of the order.

- **Order_Details**: A junction table that links products to orders, storing details for each item in an order.
	- `order_detail_id`: Unique identifier for the order line item.
	- `order_id`: Foreign key linking to the `Order` table.
	- `product_id`: Foreign key linking to the `Product` table.
	- `quantity`: The number of units of the product purchased in this order.
	- `unit_price`: The price of the product at the time of purchase.

### 1.2 SQL Schema Script
This SQL script creates the tables with appropriate primary keys, foreign keys, and constraints. It is written in a standard SQL dialect (compatible with MySQL, PostgreSQL, etc., with minor syntax adjustments if needed).

```sql
-- =============================================
-- Schema for E-commerce Database
-- =============================================

-- Table for product categories
CREATE TABLE Category (
		category_id INT PRIMARY KEY AUTO_INCREMENT,
		category_name VARCHAR(255) NOT NULL UNIQUE
);

-- Table for products
CREATE TABLE Product (
		product_id INT PRIMARY KEY AUTO_INCREMENT,
		category_id INT,
		name VARCHAR(255) NOT NULL,
		description TEXT,
		price DECIMAL(10, 2) NOT NULL,
		stock_quantity INT NOT NULL DEFAULT 0,
		FOREIGN KEY (category_id) REFERENCES Category(category_id)
);

-- Table for customers
CREATE TABLE Customer (
		customer_id INT PRIMARY KEY AUTO_INCREMENT,
		first_name VARCHAR(100) NOT NULL,
		last_name VARCHAR(100) NOT NULL,
		email VARCHAR(255) NOT NULL UNIQUE,
		password VARCHAR(255) NOT NULL -- Should be a hashed value
);

-- Table for orders
-- Note: if your SQL dialect treats "Order" as a reserved word, use `Orders` or quote the identifier.
CREATE TABLE `Order` (
		order_id INT PRIMARY KEY AUTO_INCREMENT,
		customer_id INT,
		order_date DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
		total_amount DECIMAL(10, 2) NOT NULL,
		FOREIGN KEY (customer_id) REFERENCES Customer(customer_id)
);

-- Junction table for order details, linking Orders and Products
CREATE TABLE Order_Details (
		order_detail_id INT PRIMARY KEY AUTO_INCREMENT,
		order_id INT,
		product_id INT,
		quantity INT NOT NULL,
		unit_price DECIMAL(10, 2) NOT NULL, -- Price at the time of sale
		FOREIGN KEY (order_id) REFERENCES `Order`(order_id),
		FOREIGN KEY (product_id) REFERENCES Product(product_id)
);

-- Add indexes for frequently queried columns to improve performance
CREATE INDEX idx_product_name ON Product(name);
CREATE INDEX idx_customer_email ON Customer(email);
CREATE INDEX idx_order_date ON `Order`(order_date);
```

## 2. Entity-Relationship (ER) Model

### 2.1 Relationship Identification
The relationships between the entities are crucial for maintaining data integrity and are defined by foreign keys.

- **Category → Product**: One-to-Many
	- One Category can have many Products.
	- Each Product belongs to exactly one Category.

- **Customer → Order**: One-to-Many
	- One Customer can place many Orders.
	- Each Order is placed by exactly one Customer.

- **Order → Order_Details**: One-to-Many
	- One Order can consist of many Order_Details line items.
	- Each Order_Details line item belongs to exactly one Order.

- **Product → Order_Details**: One-to-Many
	- One Product can appear in many Order_Details line items across different orders.
	- Each Order_Details line item refers to exactly one Product.

- **Product ↔ Order**: Many-to-Many (implemented via `Order_Details` junction table)

### 2.2 ER Diagram
For a visual ER diagram, you can use tools like draw.io, Lucidchart, or an automated DB diagram generator. Entities: `Category`, `Product`, `Customer`, `Order`, `Order_Details`. Connect using the foreign keys described above. Consider exporting an SVG/PNG and placing it in the repo as `docs/er-diagram.png`.

## 3. SQL Queries for Business Intelligence
Below are example queries for common BI needs. Adjust names/quoting for your SQL dialect.

### 3.1 Daily Revenue Report
Summarize total revenue per day.

```sql
SELECT
	DATE(order_date) AS order_day,
	SUM(total_amount) AS daily_revenue,
	COUNT(order_id) AS total_orders
FROM `Order`
GROUP BY DATE(order_date)
ORDER BY order_day DESC;
```

### 3.2 Monthly Top-Selling Products
List top N products by quantity sold for a given month.

```sql
-- Top 10 products for March 2025
SELECT
	p.product_id,
	p.name,
	SUM(od.quantity) AS total_quantity_sold,
	SUM(od.quantity * od.unit_price) AS total_sales
FROM Order_Details od
JOIN Product p ON od.product_id = p.product_id
JOIN `Order` o ON od.order_id = o.order_id
WHERE o.order_date >= '2025-03-01'::date
	AND o.order_date < '2025-04-01'::date
GROUP BY p.product_id, p.name
ORDER BY total_quantity_sold DESC
LIMIT 10;
```

Note: the `::date` cast is PostgreSQL-style — for MySQL use `DATE(o.order_date) = '2025-03'` style predicates or appropriate functions.

### 3.3 High-Value Customers (Past Month)
Customers with the highest spend in the past 30 days.

```sql
SELECT
	c.customer_id,
	c.first_name,
	c.last_name,
	c.email,
	SUM(o.total_amount) AS total_spent,
	COUNT(o.order_id) AS orders_count
FROM Customer c
JOIN `Order` o ON c.customer_id = o.customer_id
WHERE o.order_date >= (CURRENT_DATE - INTERVAL '30 days')
GROUP BY c.customer_id, c.first_name, c.last_name, c.email
ORDER BY total_spent DESC
LIMIT 20;
```

## 4. Database Optimization: Denormalization

### 4.1 Concept Overview
Denormalization is the deliberate introduction of redundancy to reduce the number of joins and improve read performance for common queries. It's useful for read-heavy workloads, BI queries, and analytic reporting, but it increases storage and complicates writes/updates.

When to consider denormalization:
- Frequent complex joins that slow down critical read paths.
- High-read / low-write scenarios, such as reporting or dashboards.

Trade-offs:
- Faster reads for targeted queries.
- More complex updates and potential for data inconsistency unless application logic or triggers keep denormalized fields synchronized.

### 4.2 Application Example: Customer and Order Entities
Example: If you frequently query customer orders and display the customer's full name and email with each order, you can denormalize by storing `customer_name` and `customer_email` on the `Order` record at creation time. This avoids a join when listing orders but requires careful handling if customer details change — either accept historical snapshots, update existing orders via background job, or disallow certain changes.

Example denormalized columns on `Order`:
- `customer_full_name` VARCHAR(255)
- `customer_email` VARCHAR(255)

Populate these fields at order creation with the current customer info to create an immutable snapshot of what the customer looked like at purchase time.

---

## Next Steps
- If you want, I can:
	- Add an SQL file `schema.sql` with the DDL above.
	- Create sample seed data and a small script to load it.
	- Generate an ER diagram image and add it to `docs/er-diagram.png`.

Open an issue or reply with which of the above you'd like next.
