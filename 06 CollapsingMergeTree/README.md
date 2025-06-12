# Применение движка CollapsingMergeTree в Clickhouse

### Создаем и удаляем таблицу orders
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	sign Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (order_id);
```

### Вставляем заказ с номером 1
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 100, 1, 1);
```

### Правим сумму заказа со 100 до 90
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 100, 1, -1),
(1, 'created', 90, 1, 1);
```

### Правим сумму заказа со 90 до 80
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 90, 1, -1),
(1, 'created', 80, 1, 1);
```

### Получаем актуальную строку заказа
```sql
SELECT 
	order_id,
	status,
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status
HAVING 
	SUM(sign) > 0;
```

### Меняем сумму заказа с 80 до 70 с помощью 2ух отдельных запросов
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 80, 1, -1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 70, 1, 1);
```

### Смотрим строки таблицы orders
```sql
SELECT *, _part FROM orders;
```

### Меняем статус заказа
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 70, 1, -1),
(1, 'packed', 70, 1, 1);
```

### Удаляем заказа с номером 1
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'packed', 70, 1, -1);
```



### Удаляем и создаем таблицу orders заново. Тип поля pcs изменен на Int32
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs Int32,
	sign Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (order_id);
```

### Добавляем строку заказа и 2 раза меняем сумму заказа
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', 100, 1, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', -100, -1, -1),
(1, 'created', 90, 1, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign)
VALUES
(1, 'created', -90, -1, -1),
(1, 'created', 80, 1, 1);
```

### Получаем актуальную строку заказа
```sql
SELECT 
	order_id,
	status,
	SUM(amount) AS amount,
	SUM(pcs) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status
HAVING 
	SUM(sign) > 0;
```

### Смотрим на строки таблицы orders
```sql
SELECT *, _part  FROM orders;
```

### Получаем актуальные строки таблицы orders с применением FINAL
```sql
SELECT * FROM orders FINAL;
```

### Считаем количество актуальных строк в таблице orders
```sql
SELECT 
	SUM(sign)
FROM 
	orders;
```

### Принудительно оставляем для каждого заказа только одну строку (нежелательная операция)
```sql
OPTIMIZE TABLE orders FINAL;
```
