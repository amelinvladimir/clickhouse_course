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
