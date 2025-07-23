# Инкрементальное материализованное представление

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

###
```sql
```


###
```sql
```


###
```sql
```


###
```sql
```


###
```sql
```


###
```sql
```


###
```sql
```

