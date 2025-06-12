# Применение движка VersionedCollapsingMergeTree в Clickhouse

### Удаляем и создаем таблицу orders
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	sign Int8,
	version UInt32
)
ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY (order_id);
```

### Вставляем данные
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 100, 1, 1, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 100, 1, -1, 1),
(1, 'created', 90, 1, 1, 2);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 90, 1, -1, 2),
(1, 'created', 80, 1, 1, 3);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 80, 1, -1, 3);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 70, 1, 1, 4);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'created', 70, 1, -1, 4),
(1, 'packed', 70, 1, 1, 5);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, sign, version)
VALUES
(1, 'packed', 70, 1, -1, 5),
(1, 'packed', 60, 1, 1, 6);
```

### Запускаем принудительное слияние всех частей таблицы
```sql
OPTIMIZE TABLE orders FINAL;
```

### Смотрим содержимое таблицы
```sql
SELECT *, _part FROM orders;
```

### Запрос определения актуальных строк
```sql
SELECT 
	order_id,
	status,
	version, 
	SUM(amount * sign) AS amount,
	SUM(pcs * sign) AS pcs
FROM 
	orders
GROUP BY
	order_id,
	status,
	version
HAVING 
	SUM(sign) > 0;
```
