# E-commerce Database Project

This document outlines the database schema, relationships, and key business-intelligence queries for a sample e-commerce platform. It also includes an exploration of database optimization techniques such as denormalization and an example SQL schema script.

## Table of Contents
- [1. Database Schema](#1-database-schema)
	- [1.1 Entity Descriptions](#11-entity-descriptions)
	- [1.2 SQL Schema Script](#12-sql-schema-script)
- [2. Entity-Relationship (ER) Model](#2-entity-relationship-er-model)
	- [2.1 Relationship Identification](#21-relationship-identification)
	- [2.2 ER Diagram](#22-er-diagram)
- [3. SQL Queries for Business Intelligence](#3-sql-queries-for-business-intelligence)
	- [3.1 Daily Revenue Report](#31-daily-revenue-report)
	- [3.2 Monthly Top-Selling Products](#32-monthly-top-selling-products)
	- [3.3 High-Value Customers (Past Month)](#33-high-value-customers-past-month)
	- [3.4 Full-Text Search for Products](#34-full-text-search-for-products)
	- [3.5 Product Recommendations (Same Category and Author)](#35-product-recommendations-same-category-and-author)
	- [3.6 Transaction Locking Queries](#36-transaction-locking-queries)
- [4. Sales History Trigger](#4-sales-history-trigger)
- [5. Database Optimization: Denormalization](#5-database-optimization-denormalization)

---

## 1. Database Schema

### 1.1 Entity Descriptions
The database consists of the following entities:

- **Author**: Stores information about product authors/brands.
	- `author_id`: Unique identifier for the author.
	- `author_name`: Name of the author/brand.

- **Category**: Stores product categories.
	- `category_id`: Unique identifier for the category.
	- `category_name`: Name of the category (e.g., 'Electronics', 'Books').

- **Product**: Stores information about individual products.
	- `product_id`: Unique identifier for the product.
	- `category_id`: Foreign key linking to the `Category` table.
	- `author_id`: Foreign key linking to the `Author` table (brand/author name).
	- `name`: Name of the product.
	- `description`: Detailed description of the product.
	- `price`: The selling price of the product.
	- `stock_quantity`: The number of units available in inventory.

- **Customer**: Stores customer account information.
	- `customer_id`: Unique identifier for the customer.
	- `customer_name`: Customer's full name.
	- `email`: Customer's email address (should be unique).
	- `password`: Hashed password for customer authentication.

- **Order**: Stores high-level information about a customer's order.
	- `order_id`: Unique identifier for the order.
	- `customer_id`: Foreign key linking to the `Customer` table.
	- `order_date`: The date and time the order was placed.
	- `total_amount`: The total cost of the order.

- **OrderItem**: A junction table that links products to orders, storing details for each item in an order.
	- `item_id`: Unique identifier for the order line item.
	- `order_id`: Foreign key linking to the `Order` table.
	- `product_id`: Foreign key linking to the `Product` table.
	- `quantity`: The number of units of the product purchased in this order.
	- `price`: The price of the product at the time of purchase.

- **Sales_History**: Denormalized table for reporting and analytics.
	- `history_id`: Unique identifier for the history record.
	- `order_ref_id`: Reference to the order.
	- `customer_name`: Customer name (denormalized).
	- `product_name`: Product name (denormalized).
	- `quantity`: Quantity purchased.
	- `price_charged`: Price at time of purchase.
	- `total_line_cost`: Total cost for the line item.
	- `created_at`: Timestamp of record creation.

### 1.2 SQL Schema Script
This SQL script creates all tables with appropriate primary keys, foreign keys, and constraints.

```sql
-- =============================================
-- Schema for E-commerce Database
-- =============================================

-- Table for authors/brands
CREATE TABLE Author (
    author_id INT PRIMARY KEY AUTO_INCREMENT,
    author_name VARCHAR(255) NOT NULL
);

-- Table for product categories
CREATE TABLE Category (
    category_id INT PRIMARY KEY AUTO_INCREMENT,
    category_name VARCHAR(255) NOT NULL UNIQUE
);

-- Table for products
CREATE TABLE Product (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    category_id INT,
    author_id INT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    FOREIGN KEY (category_id) REFERENCES Category(category_id),
    FOREIGN KEY (author_id) REFERENCES Author(author_id)
);

-- Add FULLTEXT index for product search
ALTER TABLE Product ADD FULLTEXT(name, description);

-- Table for customers
CREATE TABLE Customer (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL -- Should be a hashed value
);

-- Table for orders
CREATE TABLE `Order` (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    order_date DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES Customer(customer_id)
);

-- Junction table for order items, linking Orders and Products
CREATE TABLE OrderItem (
    item_id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT,
    product_id INT,
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL, -- Price at the time of sale
    FOREIGN KEY (order_id) REFERENCES `Order`(order_id),
    FOREIGN KEY (product_id) REFERENCES Product(product_id)
);

-- Sales History table (Denormalized for reporting)
CREATE TABLE Sales_History (
    history_id INT PRIMARY KEY AUTO_INCREMENT,
    order_ref_id INT,
    customer_name VARCHAR(100),
    product_name VARCHAR(100),
    quantity INT,
    price_charged DECIMAL(10, 2),
    total_line_cost DECIMAL(10, 2),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Add indexes for frequently queried columns to improve performance
CREATE INDEX idx_product_name ON Product(name);
CREATE INDEX idx_product_category ON Product(category_id);
CREATE INDEX idx_product_author ON Product(author_id);
CREATE INDEX idx_customer_email ON Customer(email);
CREATE INDEX idx_order_date ON `Order`(order_date);
CREATE INDEX idx_order_customer ON `Order`(customer_id);
```

---

## 2. Entity-Relationship (ER) Model

### 2.1 Relationship Identification
The relationships between the entities are crucial for maintaining data integrity and are defined by foreign keys.

- **Author → Product**: One-to-Many
	- One Author can have many Products.
	- Each Product belongs to exactly one Author.

- **Category → Product**: One-to-Many
	- One Category can have many Products.
	- Each Product belongs to exactly one Category.

- **Customer → Order**: One-to-Many
	- One Customer can place many Orders.
	- Each Order is placed by exactly one Customer.

- **Order → OrderItem**: One-to-Many
	- One Order can consist of many OrderItem line items.
	- Each OrderItem line item belongs to exactly one Order.

- **Product → OrderItem**: One-to-Many
	- One Product can appear in many OrderItem line items across different orders.
	- Each OrderItem line item refers to exactly one Product.

- **Product ↔ Order**: Many-to-Many (implemented via `OrderItem` junction table)

### 2.2 ER Diagram

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Author    │       │  Category   │       │  Customer   │
├─────────────┤       ├─────────────┤       ├─────────────┤
│ author_id   │───┐   │ category_id │───┐   │ customer_id │───┐
│ author_name │   │   │category_name│   │   │customer_name│   │
└─────────────┘   │   └─────────────┘   │   │ email       │   │
                  │                     │   │ password    │   │
                  │                     │   └─────────────┘   │
                  │                     │                     │
                  ▼                     ▼                     ▼
            ┌─────────────────────────────────┐         ┌─────────────┐
            │            Product              │         │    Order    │
            ├─────────────────────────────────┤         ├─────────────┤
            │ product_id                      │         │ order_id    │───┐
            │ category_id (FK)                │         │ customer_id │   │
            │ author_id (FK)                  │         │ order_date  │   │
            │ name                            │         │total_amount │   │
            │ description                     │         └─────────────┘   │
            │ price                           │◄──────────────────────────┤
            │ stock_quantity                  │                           │
            └─────────────────────────────────┘                           │
                          │                                               │
                          │              ┌─────────────┐                  │
                          │              │  OrderItem  │                  │
                          │              ├─────────────┤                  │
                          └─────────────►│ item_id     │◄─────────────────┘
                                         │ order_id(FK)│
                                         │product_id(FK)
                                         │ quantity    │
                                         │ price       │
                                         └─────────────┘
                                                │
                                                ▼ (Trigger)
                                         ┌─────────────────┐
                                         │  Sales_History  │
                                         ├─────────────────┤
                                         │ history_id      │
                                         │ order_ref_id    │
                                         │ customer_name   │
                                         │ product_name    │
                                         │ quantity        │
                                         │ price_charged   │
                                         │ total_line_cost │
                                         │ created_at      │
                                         └─────────────────┘
```



## 3. SQL Queries for Business Intelligence

Below are example queries for common BI needs. Adjust names/quoting for your SQL dialect.

### 3.1 Daily Revenue Report
Summarize total revenue for a specific date.

```sql
SELECT
    DATE(o.order_date) AS report_date,
    SUM(o.total_amount) AS total_revenue
FROM
    `Order` AS o
WHERE
    DATE(o.order_date) = '2024-01-15' -- Replace with your specific date
GROUP BY
    DATE(o.order_date);
```

### 3.2 Monthly Top-Selling Products
List top 10 products by quantity sold for a given month.

```sql
SELECT
    oi.product_id,
    p.name AS product_name,
    SUM(oi.quantity) AS total_quantity_sold,
    SUM(oi.quantity * oi.price) AS total_revenue
FROM
    OrderItem AS oi
JOIN
    `Order` AS o ON oi.order_id = o.order_id
JOIN
    Product AS p ON oi.product_id = p.product_id
WHERE
    YEAR(o.order_date) = 2024 AND MONTH(o.order_date) = 1
GROUP BY
    oi.product_id, p.name
ORDER BY
    total_quantity_sold DESC
LIMIT 10;
```

### 3.3 High-Value Customers (Past Month)
Customers with the highest spend in the past 30 days.

```sql
SELECT
    c.customer_id,
    c.customer_name,
    SUM(o.total_amount) AS total_spending_past_month
FROM
    Customer AS c
JOIN
    `Order` AS o ON c.customer_id = o.customer_id
WHERE
    o.order_date >= DATE_SUB(CURDATE(), INTERVAL 1 MONTH)
GROUP BY
    c.customer_id, c.customer_name
HAVING
    SUM(o.total_amount) > 500
ORDER BY
    total_spending_past_month DESC;
```

### 3.4 Full-Text Search for Products
To enable efficient text searching on product names and descriptions, add a FULLTEXT index:

```sql
ALTER TABLE Product 
ADD FULLTEXT(name, description);
```

Search for all products with the word "camera" in either the product name or description:

```sql
SELECT 
    product_id, 
    name, 
    description, 
    price, 
    stock_quantity
FROM 
    Product
WHERE 
    MATCH(name, description) AGAINST('camera' IN NATURAL LANGUAGE MODE);
```

### 3.5 Product Recommendations (Same Category and Author)
This query suggests products in the same category and from the same author as products the customer has previously purchased, while excluding products they have already bought.

```sql
SELECT 
    p.product_id,
    p.name AS product_name,
    p.category_id,
    p.author_id,
    a.author_name
FROM 
    Product p
JOIN
    Author a ON p.author_id = a.author_id
WHERE 
    -- 1. تحديد الفئة (Category Filter)
    -- هل فئة هذا المنتج موجودة ضمن فئات المنتجات التي اشتراها العميل؟
    p.category_id IN (
        SELECT DISTINCT p_sub.category_id
        FROM Product p_sub
        WHERE p_sub.product_id IN (
            -- هنا الاستعلام الفرعي الذي يجلب أرقام المنتجات التي اشتراها العميل
            SELECT oi.product_id
            FROM OrderItem oi
            JOIN `Order` o ON oi.order_id = o.order_id
            WHERE o.customer_id = ?customer_id
        )
    )

    -- 2. استبعاد المشتريات السابقة (Exclude Purchased Products)
    -- استبعاد المنتج إذا كان رقم تعريفه موجوداً في قائمة مشتريات العميل
    AND p.product_id NOT IN (
        SELECT oi.product_id
        FROM OrderItem oi
        JOIN `Order` o ON oi.order_id = o.order_id
        WHERE o.customer_id = ?customer_id
    )

    -- 3. تصفية حسب المؤلف (Author Filter)
    -- هل مؤلف هذا المنتج موجود ضمن المؤلفين الذين اشترى منهم العميل؟
    AND p.author_id IN (
        SELECT DISTINCT p_sub.author_id
        FROM Product p_sub
        WHERE p_sub.product_id IN (
            -- نفس الاستعلام الفرعي لجلب أرقام المنتجات المشتراة
            SELECT oi.product_id
            FROM OrderItem oi
            JOIN `Order` o ON oi.order_id = o.order_id
            WHERE o.customer_id = ?customer_id
        )
    );
```

### 3.6 Transaction Locking Queries

#### 3.6.1 Lock a Specific Field (Column-Level Simulation)
MySQL does not support true column-level locking. However, you can simulate locking a specific field by selecting only that column with `FOR UPDATE`:

```sql
START TRANSACTION;

-- Lock the stock_quantity field for product_id = 211
SELECT stock_quantity 
FROM Product 
WHERE product_id = 211 
FOR UPDATE;

-- Perform your update on the quantity field
UPDATE Product 
SET stock_quantity = stock_quantity - 1 
WHERE product_id = 211;

COMMIT;
```

#### 3.6.2 Lock an Entire Row
To lock the entire row with `product_id = 211` from being updated by other transactions:

```sql
START TRANSACTION;

-- Lock the entire row for product_id = 211
SELECT * 
FROM Product 
WHERE product_id = 211 
FOR UPDATE;

-- Perform any updates needed
UPDATE Product 
SET price = 299.99, 
    stock_quantity = 50 
WHERE product_id = 211;

COMMIT;
```

#### 3.6.3 Lock Row with NOWAIT Option (MySQL 8.0+)

```sql
START TRANSACTION;

SELECT * 
FROM Product 
WHERE product_id = 211 
FOR UPDATE NOWAIT;

UPDATE Product 
SET stock_quantity = 100 
WHERE product_id = 211;

COMMIT;
```

#### 3.6.4 Lock Row with SKIP LOCKED Option (MySQL 8.0+)

```sql
START TRANSACTION;

SELECT * 
FROM Product 
WHERE product_id = 211 
FOR UPDATE SKIP LOCKED;

COMMIT;
```

---

## 4. Sales History Trigger

This trigger automatically generates a sale history record when a new order item is inserted, capturing details such as customer name, product name, quantity, price, and total amount.

```sql
DELIMITER $$

CREATE TRIGGER capture_order_details
AFTER INSERT
ON OrderItem FOR EACH ROW
BEGIN
    DECLARE v_customer_name VARCHAR(100);
    DECLARE v_product_name VARCHAR(100);

    -- 1. Find the Customer Name
    SELECT c.customer_name 
    INTO v_customer_name
    FROM `Order` o
    JOIN Customer c ON o.customer_id = c.customer_id
    WHERE o.order_id = NEW.order_id;

    -- 2. Find the Product Name
    SELECT name 
    INTO v_product_name
    FROM Product 
    WHERE product_id = NEW.product_id;

    -- 3. Save to History
    INSERT INTO Sales_History (
        order_ref_id,
        customer_name,
        product_name,
        quantity,
        price_charged,
        total_line_cost
    )
    VALUES (
        NEW.order_id,
        v_customer_name,
        v_product_name,
        NEW.quantity,
        NEW.price,
        (NEW.quantity * NEW.price)
    );
END$$

DELIMITER ;
```

---

## 5. Database Optimization: Denormalization

### 5.1 Concept Overview
Normalization is the process of organizing columns and tables in a relational database to minimize data redundancy. The schema provided is well-normalized (Third Normal Form, 3NF).

Denormalization involves intentionally adding redundant data to improve query performance by reducing complex joins.

- **Advantage**: Faster read queries (fewer joins).
- **Disadvantage**: Slower write/update operations, increased storage, risk of data inconsistency.

### 5.2 Application Example: Customer and Order Entities

**Scenario:** Displaying recent orders with customer names requires a JOIN:

```sql
-- Query in a NORMALIZED schema
SELECT
    o.order_id,
    o.order_date,
    o.total_amount,
    c.customer_name
FROM
    `Order` o
JOIN
    Customer c ON o.customer_id = c.customer_id
WHERE
    o.order_id IN (101, 102, 103);
```

**Denormalization Strategy:** Add customer_name directly to the Order table:

```sql
ALTER TABLE `Order` ADD COLUMN customer_name VARCHAR(100);
```

**Result:** Faster queries without joins:

```sql
-- Query in a DENORMALIZED schema (faster)
SELECT
    order_id,
    order_date,
    total_amount,
    customer_name
FROM
    `Order`
WHERE
    order_id IN (101, 102, 103);
```

**Trade-off:** On customer name updates, you must update both the Customer table and all associated orders.
