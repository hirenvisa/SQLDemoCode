## Create database

```
CREATE DATABASE coffee_chain;
CREATE DATABASE brewery;
```
## Connect to database
```
\c coffee_chain
```

## Create schemas
```
CREATE SCHEMA products;
CREATE SCHEMA customers;
CREATE SCHEMA sales;
```

```
CREATE TABLE products.catalog (  
    id SERIAL PRIMARY KEY, 
    name VARCHAR (100) NOT NULL, 
    description TEXT NOT NULL,
    category TEXT CHECK (category IN ('coffee', 'mug', 't-shirt')),
    price NUMERIC(10, 2),
    stock_quantity INT CHECK (stock_quantity >= 0)
);            

CREATE TABLE products.review (  
    id BIGSERIAL PRIMARY KEY,  
    product_id INT,
    customer_id INT,
    review TEXT,
    rank SMALLINT 
);

CREATE TABLE customers.account (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    passwd_hash TEXT NOT NULL
);

```

## Insert rows
```
INSERT INTO products.catalog (name, description, category, price, stock_quantity)
VALUES
    ('Sunrise Blend', 'A smooth and balanced blend with notes of caramel and citrus.', 'coffee', 14.99, 50),
    ('Midnight Roast', 'A dark roast with rich flavors of chocolate and toasted nuts.', 'coffee', 16.99, 40),
    ('Morning Glory', 'A light roast with bright acidity and floral notes.', 'coffee', 13.99, 30),
    ('Sunrise Brew Co. Mug', 'A ceramic mug with the Sunrise Brew Co. logo.', 'mug', 9.99, 100),
    ('Sunrise Brew Co. T-Shirt', 'A soft cotton t-shirt with the Sunrise Brew Co. logo.', 't-shirt', 19.99, 25);
```

## Constraint
```
ALTER TABLE products.catalog 
    ADD CONSTRAINT catalog_price_check CHECK (price > 0);

select id, name from products.catalog;

\d products.review

ALTER TABLE products.review 
    ALTER COLUMN review SET NOT NULL,
    ADD CONSTRAINT review_rank_check CHECK (rank BETWEEN 1 AND 5);

ALTER TABLE products.review
    ADD CONSTRAINT products_review_product_id_fk
    FOREIGN KEY (product_id) REFERENCES products.catalog(id);

```

```

ALTER TABLE products.review 
    ALTER COLUMN review SET NOT NULL,
    ADD CONSTRAINT review_rank_check CHECK (rank BETWEEN 1 AND 5);

\d products.review


ALTER TABLE products.review 
    ADD CONSTRAINT products_review_customer_id_fk
    FOREIGN KEY (customer_id) REFERENCES customers.account(id);
```

# Insert rows
```



INSERT INTO customers.account (name, email, passwd_hash)
VALUES
    ('Alice Johnson', 'alice.johnson@example.com', '5f4dcc3b5aa765d61d8327deb882cf99'),
    ('Bob Smith', 'bob.smith@example.com', 'd8578edf8458ce06fbc5bb76a58c5ca4'), 
    ('Charlie Brown', 'charlie.brown@example.com', '5f4dcc3b5aa765d61d8327deb882cf99');


INSERT INTO products.review (product_id, customer_id, review, rank)
VALUES (4, 1, 'This mug is perfect — sturdy, stylish, and keeps my coffee warm for a good while.', 5);
```


```
SELECT id, name FROM products.catalog WHERE name = 'Sunrise Brew Co. Mug';


INSERT INTO products.review (product_id, customer_id, review, rank)
VALUES (1004, 1, 'This mug is perfect — sturdy, stylish, and keeps my coffee warm for a good while.', 5);

INSERT INTO products.review (product_id, customer_id, review, rank)
VALUES (4, 1, 'This mug is perfect — sturdy, stylish, and keeps my coffee warm for a good while.', 5);

ALTER TABLE customers.account 
    ADD COLUMN deleted boolean DEFAULT false;

 UPDATE customers.account SET deleted = true WHERE id = 1;
```
## Transactions
```
UPDATE products.catalog SET stock_quantity = stock_quantity + 100 WHERE id = 1;
------------------
UPDATE products.catalog SET stock_quantity = stock_quantity + 50 WHERE id = 1 or id = 3;
```

## Sales order and order items
```
CREATE TABLE sales.order (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  
    customer_id int REFERENCES customers.account(id),
    order_date timestamp DEFAULT CURRENT_TIMESTAMP,
    total_amount decimal(10, 2)
);
                        
CREATE TABLE sales.order_item (
    order_id UUID REFERENCES sales.order(id),
    product_id int REFERENCES products.catalog(id),
    quantity int CHECK (quantity > 0),
    price decimal(10, 2), 
    PRIMARY KEY (order_id, product_id)  
);
```


```
// Explicit transaction
BEGIN; 
    
INSERT INTO sales.order (id, customer_id, total_amount)  
VALUES ('19a0cffc-8757-453c-a4d2-b554fdc08954', 1, 26.53);
    
INSERT INTO sales.order_item (order_id, product_id, quantity, price)  
VALUES ('19a0cffc-8757-453c-a4d2-b554fdc08954', 1, 1, 16.54),
 ('19a0cffc-8757-453c-a4d2-b554fdc08954', 4, 1, 9.99);
    
UPDATE products.catalog  
SET stock_quantity = stock_quantity - 1
WHERE id IN (1, 4);
    
COMMIT; 
```

## JOINS
```
SELECT c.name, c.id, count(*) as total_orders
FROM customers.account c
JOIN sales.order s ON c.id = s.customer_id
GROUP BY c.id
ORDER BY total_orders DESC
LIMIT 3;
```
```
SELECT c.name 
FROM customers.account c 
LEFT JOIN sales.order s ON c.id = s.customer_id 
WHERE s.customer_id IS NULL; 
```
```
SELECT c.name, c.category, c.price, SUM(oi.quantity) AS total_sold
FROM products.catalog c
LEFT JOIN sales.order_item oi ON c.id = oi.product_id
GROUP BY c.id
ORDER BY total_sold DESC NULLS LAST, price DESC;
```
## Functions
```
CREATE OR REPLACE FUNCTION get_product_price(product_id INT) 
RETURNS NUMERIC(10, 2) AS $$ 
    SELECT price    
    FROM products.catalog
    WHERE id = product_id;
$$ LANGUAGE sql;
```
```
ALTER TABLE sales.order 
ADD COLUMN status TEXT DEFAULT 'pending' CHECK (status in ('pending','ordered'));
            
UPDATE sales.order SET status = 'ordered';  
            
ALTER TABLE sales.order  
ADD CONSTRAINT one_pending_order_per_customer
EXCLUDE USING btree (customer_id WITH =)
WHERE (status = 'pending');
```
```
CREATE OR REPLACE FUNCTION order_add_item(customer_id_param INT, product_id_param INT, quantity_param INT)  
RETURNS TABLE (order_id UUID, prod_id INT, quantity INT, prod_price DECIMAL) AS $$
DECLARE
    pending_order_id UUID;  
BEGIN
    SELECT id INTO pending_order_id  
    FROM sales.order
    WHERE customer_id = customer_id_param
      AND status = 'pending'
    LIMIT 1;
        
    IF pending_order_id IS NULL THEN  
        INSERT INTO sales.order (customer_id, status)
        VALUES (customer_id_param, 'pending')
        RETURNING id INTO pending_order_id;
    END IF;
        
    MERGE INTO sales.order_item AS oi  
    USING (SELECT id, price FROM products.catalog WHERE id = product_id_param) AS prod
    ON oi.product_id = prod.id AND oi.order_id = pending_order_id
    WHEN MATCHED THEN
        UPDATE SET quantity = quantity_param
    WHEN NOT MATCHED THEN
        INSERT (order_id, product_id, quantity, price)
        VALUES (pending_order_id, prod.id, quantity_param, prod.price);
        
    RETURN QUERY  
    SELECT oi.order_id, oi.product_id, oi.quantity, oi.price as prod_price
    FROM sales.order_item as oi
    WHERE oi.order_id = pending_order_id;
END;
$$ LANGUAGE plpgsql;
```

```
SELECT * FROM order_add_item(
    customer_id_param => 3,
    product_id_param => 3,
    quantity_param => 2
    );
```

```
CREATE OR REPLACE FUNCTION order_checkout(customer_id_param INT) 
RETURNS TABLE (order_id UUID, customer_id INT, order_date timestamp, total_amount DECIMAL) AS $$
DECLARE
    pending_order_id UUID; 
    final_total_amount DECIMAL := 0;
BEGIN
    SELECT id INTO pending_order_id 
    FROM sales.order as o
    WHERE o.customer_id = customer_id_param
      AND status = 'pending'
    LIMIT 1;
        
    IF pending_order_id IS NULL THEN 
        RAISE EXCEPTION 'No pending order found for customer %', customer_id_param;
    END IF;
            
    SELECT SUM(oi.quantity * oi.price) INTO final_total_amount 
    FROM sales.order_item oi
    WHERE oi.order_id = pending_order_id;
        
    UPDATE sales.order  
    SET status = 'ordered',
        total_amount = final_total_amount,
        order_date = CURRENT_TIMESTAMP
    WHERE id = pending_order_id;
        
    UPDATE products.catalog  
    SET stock_quantity = stock_quantity - oi.quantity
    FROM sales.order_item oi
    WHERE products.catalog.id = oi.product_id
      AND oi.order_id = pending_order_id;
            
    RETURN QUERY  
    SELECT o.id, o.customer_id, o.order_date, o.total_amount
    FROM sales.order as o
    WHERE o.id = pending_order_id;
END;
$$ LANGUAGE plpgsql;
```

```
SELECT id, name FROM customers.account WHERE name = 'Bob Smith';
SELECT * FROM order_add_item(
    customer_id_param => 2, 
    product_id_param => 1, 
    quantity_param => 1);

```
```
CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE sales.order
    SET total_amount = (  
        SELECT COALESCE(SUM(oi.quantity * oi.price), 0)  
        FROM sales.order_item oi
        WHERE oi.order_id = COALESCE(NEW.order_id, OLD.order_id)  
    )
    WHERE id = COALESCE(NEW.order_id, OLD.order_id) AND status = 'pending';        
    RETURN NEW;  
END;
$$ LANGUAGE plpgsql;
```

## Trigger
```
CREATE TRIGGER trigger_update_order_total
AFTER INSERT OR UPDATE OR DELETE ON sales.order_item
FOR EACH ROW
EXECUTE FUNCTION update_order_total();
```


## Views
```
CREATE VIEW product_sales_summary AS  
SELECT  
    c.name AS product_name,
    c.category,
    SUM(oi.quantity) AS total_quantity_sold,
    SUM(oi.quantity * oi.price) AS total_revenue
FROM products.catalog c
LEFT JOIN sales.order_item oi ON c.id = oi.product_id
GROUP BY c.id
ORDER BY total_quantity_sold DESC, total_revenue DESC;
```
```
CREATE MATERIALIZED VIEW monthly_sales_summary AS
SELECT 
    date_trunc('month', o.order_date) AS sales_month,
    SUM(oi.quantity * oi.price) AS total_revenue,
    COUNT(DISTINCT(o.id)) AS total_orders
FROM sales.order o
JOIN sales.order_item oi ON o.id = oi.order_id
GROUP BY sales_month
ORDER BY sales_month;


SELECT * FROM monthly_sales_summary;
```

## users & roles
```
CREATE ROLE coffee_chain_admin WITH LOGIN PASSWORD 'password'; 
GRANT CONNECT ON DATABASE coffee_chain TO coffee_chain_admin; 
REVOKE CONNECT ON DATABASE coffee_chain FROM PUBLIC; 


REVOKE CONNECT ON DATABASE brewery FROM PUBLIC;
REVOKE CONNECT ON DATABASE postgres FROM PUBLIC;
```

