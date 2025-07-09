# Применение движка AggregatingMergeTree в Clickhouse

### Пересоздаем таблицу orders
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	user_id UInt32,
	product_id UInt32,
	amount Decimal(18, 2),
	order_date Date
)
ENGINE = MergeTree()
ORDER BY (product_id, order_date);
```

### Вставляем в таблицу orders 4 строки
```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(1, 1, 1, 10, '2025-01-01'),
(2, 2, 1, 10, '2025-01-01'),
(3, 1, 2, 5, '2025-01-01'),
(4, 2, 2, 5, '2025-01-01');
```

### Считаем количество уникальных строк в orders
```sql
SELECT uniq(user_id) FROM learn_db.orders;
```

### Пересоздаем таблицу orders_agg
```sql
DROP TABLE IF EXISTS learn_db.orders_agg;
CREATE TABLE learn_db.orders_agg (
	product_id UInt32,
	order_date Date,
	amount AggregateFunction(sum, Decimal(18, 2)),
    users AggregateFunction(uniq, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (product_id, order_date);
```

### Наполняем таблицу orders_agg агрегированными данными из таблицы orders
```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
GROUP BY 
	product_id,
	order_date;
```

### Смотрим содержимое таблицы orders_agg
```sql
SELECT *, _part FROM learn_db.orders_agg;
```

### Вставляем еще одну строку в orders 
```sql
INSERT INTO learn_db.orders
(order_id, user_id, product_id, amount, order_date)
VALUES
(5, 3, 1, 10, '2025-01-01');
```

### Вставляем эту же строку в orders_agg
```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
WHERE 
	order_id = 5
GROUP BY 
	product_id,
	order_date;
```

### Смотрим содержимое таблицы orders_agg
```sql
SELECT *, _part FROM learn_db.orders_agg;
```

### Выполняем принудительный процесс слияния частей в orders_agg и смотрим ее содержимое
```sql
OPTIMIZE TABLE learn_db.orders_agg FINAL;
SELECT *, _part FROM learn_db.orders_agg;
```

### Выполняем аналитические запросы к orders_agg
```sql
SELECT 
	uniqMerge(users)
FROM 
	learn_db.orders_agg;

SELECT 
	product_id,
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;

SELECT 
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg;
	
SELECT 
	product_id,
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;
```

## Замеряем скорость выполнения запросов к таблице с движком AggregatingMergeTree

### Очищаем таблицы orders и orders_agg
```sql
TRUNCATE learn_db.orders;
TRUNCATE learn_db.orders_agg;

select count(*) from learn_db.orders_agg;
select count(*) from learn_db.orders;
```

### Наполняем таблицу orders  100 000 000 строк
```sql
INSERT INTO learn_db.orders
SELECT
	number + 10 as order_id,
	round(randUniform(1, 1000)) as user_id,
	round(randUniform(1, 1000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % 366, today() - INTERVAL 1 YEAR)
FROM 
	numbers(100000000);
```

### Наполняем orders_agg 366 000 строк
```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
GROUP BY 
	product_id,
	order_date;
```

### Сравниеваем скорость выполнения запросов
```sql
SELECT 
	product_id,
	sum(amount),
	uniq(user_id)
FROM 
	learn_db.orders
GROUP BY 
	product_id;

SELECT 
	product_id,
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;
```



### Очищаем таблицы orders и orders_agg
```sql
TRUNCATE learn_db.orders;
TRUNCATE learn_db.orders_agg;

select count(*) from learn_db.orders_agg;
select count(*) from learn_db.orders;
```

### Наполняем таблицу orders  100 000 000 строк, но количество уникальных товаров увеличено на порядок
```sql
INSERT INTO learn_db.orders
SELECT
	number + 10 as order_id,
	round(randUniform(1, 1000)) as user_id,
	round(randUniform(1, 10000)) as product_id,
	randUniform(1, 5000) as amount,
	date_add(day, rand() % 366, today() - INTERVAL 1 YEAR)
FROM 
	numbers(100000000);
```

### Наполняем orders_agg 3 660 000 строк
```sql
INSERT INTO learn_db.orders_agg
SELECT
	product_id,
	order_date,
	sumState(amount) as amount,
	uniqState(user_id) as users
FROM 
	learn_db.orders
GROUP BY 
	product_id,
	order_date;
```

### Сравниваем скорость выполнения запросов
```sql
SELECT 
	product_id,
	sum(amount),
	uniq(user_id)
FROM 
	learn_db.orders
GROUP BY 
	product_id;

SELECT 
	product_id,
	sumMerge(amount),
	uniqMerge(users)
FROM 
	learn_db.orders_agg
GROUP BY 
	product_id;
```
