# Партиционирование таблиц

### Создаем не партиционированную таблицу с заказами orders
```sql
DROP TABLE IF EXISTS learn_db.orders;
CREATE TABLE learn_db.orders (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
ORDER BY (product_id);
```

### Вставляем в таблицу с заказам 10 000 000 строк
```sql
INSERT INTO learn_db.orders
SELECT
	number as order_id,
	round(randUniform(1, 100000)) as user_id,
	round(randUniform(1, 10000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % (365 * 10 + 2), today() - INTERVAL 10 YEAR),
FROM 
	numbers(10000000);
```

### Создаем таблицу с заказам партционированную по годам
```sql
DROP TABLE IF EXISTS learn_db.orders_partition_by_year;
CREATE TABLE learn_db.orders_partition_by_year (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
PARTITION BY toYear(order_date)
ORDER BY (product_id);
```

### Вставляем данные в партиционированную таблицу
```sql
INSERT INTO learn_db.orders_partition_by_year
SELECT * FROM learn_db.orders;
```

### Получаем список частей данных и партиций по двум таблицам с заказами
```sql
SELECT DISTINCT _partition_id, _part FROM learn_db.orders ORDER BY _partition_id, _part;
SELECT DISTINCT _partition_id, _part FROM learn_db.orders_partition_by_year ORDER BY _partition_id, _part;
```

### Смотрим статистику по партициям в двух таблицах с заказами
```sql
SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders') AND active
GROUP BY partition
ORDER BY partition ASC;

SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders_partition_by_year') AND active
GROUP BY partition
ORDER BY partition ASC;
```

### Сравниваем скорость выполнения запроса из таблицы без партиционирования с запросом из таблицы с партиционированием с фильтрацией по полю, по которому таблица партиционирована
```sql
SELECT 
	*
FROM
	learn_db.orders
WHERE
	`product_id` between 1000 and 3000
	AND `order_date` BETWEEN '2024-01-01' AND '2024-12-31';

SELECT 
	*
FROM
	learn_db.orders_partition_by_year
WHERE
	`product_id` between 1000 and 3000
	AND `order_date` BETWEEN '2024-01-01' AND '2024-12-31';
```

### Исследуем планы выполнения запросов
```sql
EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders
WHERE
	`product_id` between 1000 and 3000
	AND `order_date` BETWEEN '2024-01-01' AND '2024-12-31';

EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders_partition_by_year
WHERE
	`product_id` between 1000 and 3000
	AND `order_date` BETWEEN '2024-01-01' AND '2024-12-31';
```

### Исследуем планы выполнения запросов без фильтрации по полю, по которому партиционирована таблица
```sql
EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders
WHERE
	`product_id` between 1000 and 1000;

EXPLAIN indexes = 1
SELECT 
	*
FROM
	learn_db.orders_partition_by_year
WHERE
	`product_id` between 1000 and 1000;
```

### Реализована автоматическая миграция данных: записи заказов старше 12 месяцев перемещаются на более экономичные, но медленные диски
```sql
DROP TABLE IF EXISTS learn_db.orders_partition_by_year;
CREATE TABLE learn_db.orders_partition_by_year (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
PARTITION BY toStartOfMonth(order_date)
ORDER BY (product_id)
TTL order_date + INTERVAL 12 MONTH TO VOLUME 'slow_but_cheap';
```

### Реализовано автоматическое удаление данных: записи заказов старше 12 месяцев удаляются
```sql
DROP TABLE IF EXISTS learn_db.orders_partition_by_year;
CREATE TABLE learn_db.orders_partition_by_year (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
PARTITION BY toStartOfMonth(order_date)
ORDER BY (product_id)
TTL order_date + INTERVAL 12 MONTH DELETE;
```

## Управление партициями и частями данных

### Получаем список всех партиций
```sql
SELECT
    partition,
    count() AS parts,
    sum(rows) AS rows
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders_partition_by_year') AND active
GROUP BY partition
ORDER BY partition ASC;
```

### Получаем список всех активных частей данных из партиций за 2015 год
```sql
SELECT
    *
FROM system.parts
WHERE (database = 'learn_db') AND (`table` = 'orders_partition_by_year')
	AND `partition` = '2015'
	AND active = true;
```

### Открепляем партицию за 2015 год
```sql
ALTER TABLE learn_db.orders_partition_by_year DETACH PARTITION '2015';
```

### Прикрепляем открепленную партицию за 2015 год
```sql
ALTER TABLE learn_db.orders_partition_by_year ATTACH PARTITION 2015;
ALTER TABLE learn_db.orders_partition_by_year ATTACH PARTITION ALL;
```

### Открепляем и прикрепляем обратно часть данных
```sql
ALTER TABLE learn_db.orders_partition_by_year DETACH PART '2015_112_112_0';
ALTER TABLE learn_db.orders_partition_by_year ATTACH PART '2015_112_112_0';
```

### Открепляем партицию за 2015 год и затем удаляем ее
```sql
ALTER TABLE learn_db.orders_partition_by_year DETACH PARTITION '2015';
ALTER TABLE learn_db.orders_partition_by_year DROP DETACHED PARTITION '2015'
SETTINGS allow_drop_detached = 1;
```
