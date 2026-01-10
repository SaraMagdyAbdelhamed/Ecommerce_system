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
- [6- MassData Generation] (#6-Mass Data Generation & Initialization)

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
-----------------------------------------------
## 6. Mass Data Generation & Initialization

To simulate a real-world environment, this project uses Cross-Join Data Generation. This method allows for the insertion of millions of rows in seconds by mathematically multiplying small "pool" tables, bypassing the performance bottlenecks of standard loops.

6.1 Data Pool Setup
Before running procedures, permanent pool tables are created to act as the "DNA" for generating realistic names and titles.
-- 1. Create permanent pool tables
```sql
CREATE TABLE IF NOT EXISTS Pool_Adjectives (word VARCHAR(50));
CREATE TABLE IF NOT EXISTS Pool_Nouns (word VARCHAR(50));
CREATE TABLE IF NOT EXISTS Pool_Words (word VARCHAR(50));
CREATE TABLE IF NOT EXISTS Pool_Cats (word VARCHAR(50));
CREATE TABLE IF NOT EXISTS Pool_First_Names (name VARCHAR(50));
CREATE TABLE IF NOT EXISTS Pool_Last_Names (name VARCHAR(50));

-- Populate with realistic fragments
INSERT INTO Pool_Adjectives VALUES ('Pro'),('Ultra'),('Wireless'),('Eco'),('Luxury'),('Smart'),('Classic'),('Portable'),('Organic'),('Premium');
INSERT INTO Pool_Nouns VALUES ('Gadget'),('Solution'),('Kit'),('Device'),('System'),('Gear'),('Tool'),('Pack'),('Unit'),('Essentials');
INSERT INTO Pool_Words VALUES ('Advanced'),('Modern'),('Professional'),('Digital'),('Compact'),('Heavy-Duty'),('Reliable'),('Generic'),('Special'),('Elite');
INSERT INTO Pool_Cats VALUES ('Electronics'),('Home'),('Books'),('Clothing'),('Health'),('Toys'),('Automotive'),('Beauty'),('Sports'),('Gourmet');
INSERT INTO Pool_First_Names VALUES ('James'),('Mary'),('Robert'),('Patricia'),('John'),('Jennifer'),('Michael'),('Linda'),('David'),('Elizabeth');
INSERT INTO Pool_Last_Names VALUES ('Smith'),('Johnson'),('Williams'),('Brown'),('Jones'),('Garcia'),('Miller'),('Davis'),('Rodriguez'),('Martinez');

-- Generate 30 distinct Authors from name pools
INSERT INTO Author (author_name)
SELECT CONCAT(f.name, ' ', l.name)
FROM Pool_First_Names f CROSS JOIN Pool_Last_Names l LIMIT 30;

```



1. SetupCategories()
   Description: This function initializes the product hierarchy. It combines adjectives (like "Smart") with base category names (like "Electronics") to create a diverse set of 100 unique categories.Logic: It uses a CROSS JOIN between Pool_Adjectives and Pool_Cats ($10 \times 10 = 100$) to generate the names.
```sql
DELIMITER //
CREATE PROCEDURE SetupCategories()
BEGIN
    INSERT IGNORE INTO Category (category_name)
    SELECT CONCAT(word, ' ', name) 
    FROM Pool_Adjectives 
    CROSS JOIN Pool_Cats;
END //
DELIMITER ;
```
2. SetupProducts()
Description: This function populates the digital catalog. It generates 200,000 unique products by mixing multiple word pools. It randomly assigns each product to an existing category and author. Logic: It uses a nested CROSS JOIN and a multiplier subquery to explode the dataset to 200,000 rows in a single operation.
```sql

DELIMITER //
CREATE PROCEDURE SetupProducts()
BEGIN
    SET SESSION autocommit = 0;
    INSERT INTO Product (category_id, author_id, name, description, price, stock_quantity)
    SELECT 
        (SELECT category_id FROM Category ORDER BY RAND() LIMIT 1),
        (SELECT author_id FROM Author ORDER BY RAND() LIMIT 1),
        CONCAT(a.word, ' ', n.word, ' ', w.word, ' ', m.n),
        CONCAT('High-quality ', a.word, ' product.'),
        ROUND(10 + RAND() * 500, 2),
        FLOOR(RAND() * 100)
    FROM Pool_Adjectives a 
    CROSS JOIN Pool_Nouns n 
    CROSS JOIN Pool_Words w 
    CROSS JOIN (SELECT 1 n UNION SELECT 2) m 
    CROSS JOIN (SELECT 1 FROM Pool_Adjectives CROSS JOIN Pool_Adjectives) counters
    LIMIT 200000;
    COMMIT;
END //
DELIMITER ;
```

3. SetupCustomers()
   Description: This function creates 1,000,000 unique customer profiles. It ensures that every customer has a unique email address by appending a row number to the generated names.Logic: It joins first and last name pools and scales them up using three 10-row multipliers ($100 \times 10 \times 10 \times 10 = 1,000,000$).

```sql

DELIMITER //
CREATE PROCEDURE SetupCustomers()
BEGIN
    SET SESSION autocommit = 0;
    INSERT INTO Customer (customer_name, email, password)
    SELECT 
        CONCAT(f.name, ' ', l.name),
        CONCAT(LOWER(f.name), '.', LOWER(l.name), (ROW_NUMBER() OVER()), '@example.com'),
        '$2y$10$hashed_password_string'
    FROM Pool_First_Names f 
    CROSS JOIN Pool_Last_Names l 
    CROSS JOIN (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) m1
    CROSS JOIN (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) m2
    CROSS JOIN (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) m3;
    COMMIT;
END //
DELIMITER ;
```

5. SetupOrdersOnly()
Description: This function generates the "Header" of the orders (2,000,000 rows). It assigns each order to a random customer and generates a random date within the last 2 years. Logic: By setting total_amount to 0 initially, it allows for a high-speed insert that bypasses complex price calculations until the items are added.
```sql

DELIMITER //
CREATE PROCEDURE SetupOrdersOnly()
BEGIN
    SET FOREIGN_KEY_CHECKS = 0;
    SET SESSION autocommit = 0;
    INSERT INTO `Order` (customer_id, order_date, total_amount)
    SELECT 
        FLOOR(1 + RAND() * 1000000),
        DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 730) DAY),
        0
    FROM (SELECT 1 FROM Customer LIMIT 1000) a 
    CROSS JOIN (SELECT 1 FROM Customer LIMIT 2000) b;
    COMMIT;
    SET FOREIGN_KEY_CHECKS = 1;
END //
DELIMITER ;
```
5. SetupOrderItemsBatched()
Description: The most critical function for large datasets. It populates 5,000,000 line items. It uses a Batching Strategy to prevent "504 Gateway Timeouts" and database crashes. Logic: It uses a WHILE loop that commits every 100,000 rows. This clears the database memory periodically and ensures that if a crash occurs, the previous batches are safely saved.
```sql

DELIMITER //
CREATE PROCEDURE SetupOrderItemsBatched()
BEGIN
    DECLARE i INT DEFAULT 0;
    SET FOREIGN_KEY_CHECKS = 0;
    SET UNIQUE_CHECKS = 0;

    WHILE i < 50 DO
        START TRANSACTION;
        INSERT INTO OrderItem (order_id, product_id, quantity, price)
        SELECT 
            FLOOR(1 + RAND() * 2000000),
            FLOOR(1 + RAND() * 200000),
            FLOOR(1 + RAND() * 5),
            ROUND(5 + (RAND() * 495), 2)
        FROM (SELECT 1 FROM Customer LIMIT 100000) AS t;
        COMMIT;
        SET i = i + 1;
    END WHILE;

    SET FOREIGN_KEY_CHECKS = 1;
    SET UNIQUE_CHECKS = 1;
END //
DELIMITER ;
```

7. SetupAuthors()
Description: This function populates the Author table with 30 unique names. It acts as the "Brand" or "Creator" registry for the products. By using a CROSS JOIN between the first and last name pools, it creates realistic human names rather than generic placeholders. Logic: It combines the Pool_First_Names and Pool_Last_Names tables. Since each pool contains 10 names, the cross-join creates 100 possible combinations, which we then LIMIT to the required 30.
```sql

DELIMITER //
CREATE PROCEDURE SetupAuthors()
BEGIN
    -- Disable checks for a clean, fast insert
    SET FOREIGN_KEY_CHECKS = 0;
    
    -- Insert 30 unique names by mixing first and last name pools
    INSERT INTO Author (author_name)
    SELECT DISTINCT CONCAT(f.name, ' ', l.name)
    FROM Pool_First_Names f
    CROSS JOIN Pool_Last_Names l
    LIMIT 30;
    
    SET FOREIGN_KEY_CHECKS = 1;
END //
DELIMITER ;
```

