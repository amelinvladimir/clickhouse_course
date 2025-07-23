# Инкрементальное материализованное представление

## Сценарий 1

### Создает таблицу с заказами
```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE orders (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
ORDER BY (order_id, user_id);
```

### Создаем агрегированную таблицу с заказами по id продукта и дате заказа 
```sql
DROP TABLE IF EXISTS learn_db.orders_sum;
CREATE TABLE learn_db.orders_sum (
	product_id UInt32,
	order_date Date,
	amount Decimal(18, 2)
)
ENGINE = SummingMergeTree()
ORDER BY (product_id, order_date);
```

### Создаем материализованное представление, отслеживащее новые строки в orders и добавляющее строки в orders_sum_mv
```sql
DROP TABLE IF EXISTS learn_db.orders_sum_mv;
CREATE MATERIALIZED VIEW learn_db.orders_sum_mv TO learn_db.orders_sum AS
SELECT product_id,
       order_date,
       SUM(amount) as amount
FROM learn_db.orders
GROUP BY 
	product_id,
    order_date;
```

### Вставляет новые строки в orders
```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(1, 1, 1, 10, '2025-01-01'),
(2, 2, 1, 10, '2025-01-01'),
(3, 1, 2, 5, '2025-01-01'),
(4, 2, 2, 5, '2025-01-01');
```

### Проверяем, появились ли новые строки в orders_sum
```sql
SELECT 
	*
FROM 
	learn_db.orders_sum;
```

### Вставляем еще одну строку в orders
```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(5, 3, 1, 10, '2025-01-01');
```

### Смотрим, что в orders_sum
```sql
SELECT 
	*
FROM 
	learn_db.orders_sum;

SELECT 
	*
FROM 
	learn_db.orders_sum FINAL;

SELECT 
	product_id,
    order_date,
    SUM(amount) as amount
FROM
	learn_db.orders_sum
GROUP BY 
	product_id,
    order_date;
```

### Вставляем в orders 100 000 000 строк
```sql
INSERT INTO learn_db.orders
SELECT
	number + 10 as order_id,
	round(randUniform(1, 100000)) as user_id,
	round(randUniform(1, 10000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % 366, today() - INTERVAL 1 YEAR)
FROM 
	numbers(100000000);
```

### Смотрим, сколько строк в orders и orders_sum
```sql
SELECT COUNT(*) FROM learn_db.orders_sum;
SELECT COUNT(*) FROM learn_db.orders_sum FINAL;
SELECT COUNT(*) FROM learn_db.orders;
```

### Получаем последние запросы из Datalens
```sql
SELECT * FROM system.query_log WHERE type = 2 and `http_user_agent` = 'DataLens' ORDER BY `event_time` DESC;
```

## Сценарий 2

### Создаем вспомогательную таблицу orders_user_order
```sql
DROP TABLE IF EXISTS learn_db.orders_user_order;
CREATE TABLE learn_db.orders_user_order (
	order_id UInt32,
	user_id UInt32
) ENGINE = MergeTree ORDER BY user_id;
```

### Создаем материализованное представление orders_user_order_mv для наполнения orders_user_order
```sql
DROP TABLE IF EXISTS learn_db.orders_user_order_mv;
CREATE MATERIALIZED VIEW learn_db.orders_user_order_mv TO learn_db.orders_user_order AS
SELECT 
	order_id,
    user_id
FROM learn_db.orders;
```

### Получаем количество строк из orders_user_order
```sql
SELECT COUNT(*) FROM learn_db.orders_user_order;
SELECT COUNT(*) FROM learn_db.orders_user_order_mv;
```

### Вставляем 4 строки в orders и смотрим содержимое orders_user_order 
```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(100000011, 1, 1, 10, '2025-01-01'),
(100000012, 2, 1, 10, '2025-01-01'),
(100000013, 1, 2, 5, '2025-01-01'),
(100000014, 2, 2, 5, '2025-01-01');

SELECT COUNT(*) FROM learn_db.orders_user_order;
SELECT COUNT(*) FROM learn_db.orders_user_order_mv;
```

### Вставляем в orders_user_order все строки из orders, за исключением последних 4
```sql
INSERT INTO `orders_user_order`
(
	`order_id`, 
	`user_id`
)
SELECT
	order_id,
	user_id
FROM 	
	learn_db.orders
WHERE 
	order_id <= 100000010;

SELECT * FROM learn_db.orders;
```

### Замеряем время поиска одного заказа по order_id
```sql
SELECT 
	*
FROM
	learn_db.orders
WHERE 
	order_id = 1000;
```

### Замеряем время поиска всех заказов одного покупателя
```sql
SELECT 
	*
FROM
	learn_db.orders
WHERE 
	user_id = 1000;
```

### Замеряем время поиска всех заказов одного покупателя с применением вспомогательной таблицы
```sql
SELECT 
	*
FROM
	learn_db.orders
WHERE 
	order_id in (
		SELECT
			order_id
		FROM
			learn_db.orders_user_order
		WHERE
			user_id = 1000
	);
```

### Создаем материализованное представление, сохраняющее данные в себе
```sql
DROP TABLE IF EXISTS learn_db.orders_user_order_mv_v2;
CREATE MATERIALIZED VIEW learn_db.orders_user_order_mv_v2 
ENGINE = MergeTree()
ORDER BY (user_id)
AS SELECT 
	order_id,
    user_id
FROM learn_db.orders;
```

### Смотрим содержимое orders_user_order_mv_v2
```sql
SELECT * FROM learn_db.orders_user_order_mv_v2;
SELECT COUNT(*) FROM learn_db.orders_user_order_mv_v2;
```

### Вставляем еще один заказ в orders и смотрим содержимое orders_user_order_mv_v2
```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(100000015, 1, 1, 10, '2025-01-01');

SELECT * FROM learn_db.orders_user_order_mv_v2;
SELECT COUNT(*) FROM learn_db.orders_user_order_mv_v2;
```

### Вставляем в orders_user_order_mv_v2 все заказы кроме последнего
```sql
INSERT INTO learn_db.orders_user_order_mv_v2
SELECT 
	order_id,
    user_id
FROM learn_db.orders
WHERE order_id <> 100000015;
```

## Проверяем реагирование материализованного представления на редактирование данных

###
```sql
SELECT COUNT(*) FROM learn_db.orders_user_order;
SELECT COUNT(*) FROM learn_db.orders;

ALTER TABLE learn_db.orders DELETE WHERE order_id = 1000;

SELECT COUNT(*) FROM learn_db.orders_user_order;
SELECT COUNT(*) FROM learn_db.orders;
```
