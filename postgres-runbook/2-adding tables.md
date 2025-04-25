```

CREATE TABLE trade(
    id bigint,
    buyer_id integer,
    symbol text,
    order_quantity integer,
    bid_price numeric(5,2),
    order_time timestamp
);
```

```bash
\d
```

```
INSERT INTO trade (id, buyer_id, symbol, order_quantity, bid_price, order_time)
    SELECT
        id,
        random(1,10) as buyer_id,
        (array['AAPL','F','DASH'])[random(1,3)] as symbol,
        random(1,20) as order_quantity,
        round(random(10.00,20.00), 2) as bid_price,
        now() as order_time
    FROM generate_series(1,1000) AS id;

SELECT count(*) FROM trade WHERE symbol = 'AAPL';
```

```bash
EXPLAIN ANALYZE SELECT count(*) FROM trade WHERE symbol = 'AAPL';
----------------------------------------------------------------------------------------------------------
 Aggregate  (cost=22.26..22.27 rows=1 width=8) (actual time=1.920..1.922 rows=1 loops=1)
   ->  Seq Scan on trade  (cost=0.00..21.50 rows=305 width=0) (actual time=0.190..1.862 rows=305 loops=1)
         Filter: (symbol = 'AAPL'::text)
         Rows Removed by Filter: 695
 Planning Time: 0.298 ms
 Execution Time: 2.033 ms
```

Some queries
```
SELECT symbol, count(order_quantity) AS total_volume
FROM trade
GROUP BY symbol
ORDER BY total_volume DESC;
------------------------------------------------------
SELECT buyer_id, sum(bid_price * order_quantity) AS total_value
FROM trade
GROUP BY buyer_id
ORDER BY total_value DESC
LIMIT 3;
------------------------------------------------------

```


```
CREATE SCHEMA products;
CREATE SCHEMA customers;
CREATE SCHEMA sales;
\dn



CREATE TABLE products.catalog (  
    id SERIAL PRIMARY KEY, 
    name VARCHAR (100) NOT NULL, 
    description TEXT NOT NULL,
    category TEXT CHECK (category IN ('coffee', 'mug', 't-shirt')),
    price NUMERIC(10, 2),
    stock_quantity INT CHECK (stock_quantity >= 0)
);            

INSERT INTO products.catalog (name, description, category, price, stock_quantity)
VALUES
    ('Sunrise Blend', 'A smooth and balanced blend with notes of caramel and citrus.', 'coffee', 14.99, 50),
    ('Midnight Roast', 'A dark roast with rich flavors of chocolate and toasted nuts.', 'coffee', 16.99, 40),
    ('Morning Glory', 'A light roast with bright acidity and floral notes.', 'coffee', 13.99, 30),
    ('Sunrise Brew Co. Mug', 'A ceramic mug with the Sunrise Brew Co. logo.', 'mug', 9.99, 100),
    ('Sunrise Brew Co. T-Shirt', 'A soft cotton t-shirt with the Sunrise Brew Co. logo.', 't-shirt', 19.99, 25);

ALTER TABLE products.catalog 
    ADD CONSTRAINT catalog_price_check CHECK (price > 0);

select id, name from products.catalog;
```

```
CREATE TABLE products.review (  
    id BIGSERIAL PRIMARY KEY,  
    product_id INT,
    customer_id INT,
    review TEXT,
    rank SMALLINT 
);

ALTER TABLE products.review 
    ALTER COLUMN review SET NOT NULL,
    ADD CONSTRAINT review_rank_check CHECK (rank BETWEEN 1 AND 5);

\d products.review
```


```
CREATE TABLE customers.account (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    passwd_hash TEXT NOT NULL
);


ALTER TABLE products.review 
    ADD CONSTRAINT products_review_customer_id_fk
    FOREIGN KEY (customer_id) REFERENCES customers.account(id);


INSERT INTO customers.account (name, email, passwd_hash)
VALUES
    ('Alice Johnson', 'alice.johnson@example.com', '5f4dcc3b5aa765d61d8327deb882cf99'),
    ('Bob Smith', 'bob.smith@example.com', 'd8578edf8458ce06fbc5bb76a58c5ca4'), 
    ('Charlie Brown', 'charlie.brown@example.com', '5f4dcc3b5aa765d61d8327deb882cf99');

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



