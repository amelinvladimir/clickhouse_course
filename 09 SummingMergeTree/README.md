# Применение движка SummingMergeTree

### Создаем таблицу с движком MergeTree, в которой будут храниться сырые данные
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	customer_id UInt32,
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = MergeTree()
ORDER BY (sale_dt, product_id, customer_id, order_id);
```

### Вставляем данные в таблицу с сырыми данными
```sql
INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(1, 1, '2025-06-01', 1, 'successed', 100, 1),
(1, 1, '2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(2, 1, '2025-06-01', 1, 'canceled', 110, 1),
(2, 1, '2025-06-01', 2, 'canceled', 60, 2);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(3, 2, '2025-06-01', 1, 'successed', 100, 1);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(4, 3, '2025-06-01', 2, 'successed', 25, 1);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(5, 1, '2025-06-02', 1, 'successed', 95, 1);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(6, 1, '2025-06-02', 1, 'successed', 285, 3);
```

### Создаем таблицу с движком SummingMergeTree и ключом сортировки по дате, статусу и продукту
```sql
DROP TABLE IF EXISTS orders_summ;
CREATE TABLE orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = SummingMergeTree()
ORDER BY (sale_dt, status, product_id);
```

### Вставляем в таблицу с агрегированными данными данные аналогичные, вставленным в таблицу с сырыми данным 
```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),
('2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'canceled', 110, 1),
('2025-06-01', 2, 'canceled', 60, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 2, 'successed', 25, 1);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-02', 1, 'successed', 95, 1);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-02', 1, 'successed', 285, 3);
```

### Смотрим содержимое таблиц
```sql
SELECT *, _part FROM orders;
SELECT *, _part FROM orders_summ;
```

### Окончательный запрос получения сгруппированных, просуммированных данных
```sql
SELECT 
	sale_dt,
	product_id,
	status,
	SUM(amount) as amount,
	SUM(pcs) as pcs
FROM
	orders_summ
GROUP BY
	sale_dt,
	product_id,
	status;
```

### Пересоздаем таблицу с агрегированными данными с ключом сортировки по дате и продуту (без статуса)
```sql
DROP TABLE IF EXISTS orders_summ;
CREATE TABLE orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = SummingMergeTree()
ORDER BY (sale_dt, product_id);
```

### Вставляем данные
```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),
('2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'canceled', 110, 1),
('2025-06-01', 2, 'canceled', 60, 2);
```

### Делаем принудительное слияние частей и смотрим содержимое
```sql
OPTIMIZE TABLE learn_db.orders_summ FINAL; 
SELECT *, _part FROM orders_summ;
```

### Пересоздаем таблицу с агрегированными данными, указав суммирование только по полю amount
```sql
DROP TABLE IF EXISTS orders_summ;
CREATE TABLE orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = SummingMergeTree(amount)
ORDER BY (sale_dt, status, product_id);
```

### Вставляем данные
```sql
INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(1, 1, '2025-06-01', 1, 'successed', 100, 1),
(1, 1, '2025-06-01', 2, 'successed', 50, 2);

INSERT INTO learn_db.orders
(order_id, customer_id, sale_dt, product_id, status, amount, pcs)
VALUES
(3, 2, '2025-06-01', 1, 'successed', 100, 1);
```

### Делаем принудительное слияние частей и смотрим содержимое
```sql
OPTIMIZE TABLE learn_db.orders_summ FINAL; 
SELECT *, _part FROM orders_summ;
```

### Пересоздаем таблицу с агрегированными данными (у поля pcs тип меняем на Int32) 
```sql
DROP TABLE IF EXISTS orders_summ;
CREATE TABLE orders_summ (
	sale_dt Date,
	product_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs Int32
)
ENGINE = SummingMergeTree()
ORDER BY (sale_dt, status, product_id);
```

### Вставляем данные
```sql
INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', 100, 1),

INSERT INTO learn_db.orders_summ
(sale_dt, product_id, status, amount, pcs)
VALUES
('2025-06-01', 1, 'successed', -100, -1),
```

### Делаем принудительное слияние частей и смотрим содержимое
```sql
OPTIMIZE TABLE learn_db.orders_summ FINAL; 
SELECT *, _part FROM orders_summ;
```
