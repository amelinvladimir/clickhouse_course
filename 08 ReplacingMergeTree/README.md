# Применение движка ReplacingMergeTree

### Удаляем и создаем таблицу orders с движком ReplacingMergeTree
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32
)
ENGINE = ReplacingMergeTree()
ORDER BY (order_id);
```

### Вставляем 4 строки в таблицу, соответствующие одному заказу
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 100, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 90, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 80, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs)
VALUES
(1, 'created', 110, 1);
```

### Получаем все строки из таблицы заказов и только актуальную строку
```sql
SELECT * FROM orders o;
SELECT * FROM orders o FINAL;
```

### Пересоздаем таблицу orders, добавив колонку с номером версии строки
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	version UInt32
)
ENGINE = ReplacingMergeTree(version)
ORDER BY (order_id);
```

### Вставляем 4 строки в таблицу, соответствующие одному заказу
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 100, 1, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 90, 1, 3);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 80, 1, 4);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version)
VALUES
(1, 'created', 70, 1, 2);
```

### Получаем все строки из таблицы заказов и только актуальную строку
```sql
SELECT * FROM orders o;
SELECT * FROM orders o FINAL;
```

### Пересоздаем таблицу orders, добавив колонку с пометкой, что строка удалена
```sql
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
	order_id UInt32,
	status String,
	amount Decimal(18, 2),
	pcs UInt32,
	version UInt32,
	is_deleted UInt8
)
ENGINE = ReplacingMergeTree(version, is_deleted)
ORDER BY (status, order_id);
```

### Вставляем 4 строки, меняющие состояние заказа с номером 1
```sql
INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 100, 1, 1, 0);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 90, 1, 3, 0);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 80, 1, 4, 1);

INSERT INTO learn_db.orders
(order_id, status, amount, pcs, version, is_deleted)
VALUES
(1, 'created', 70, 1, 2, 0);
```

### Получаем все строки из таблицы заказов и только актуальную строку
```sql
SELECT * FROM orders o;
SELECT * FROM orders o FINAL;
```

## На сколько запрос замедляется при использовании FINAL

### Создаем и наполняем таблицу с движком MergeTree
```sql
DROP TABLE learn_db.mart_student_lesson;
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32, -- Идентификатор профиля обучающегося
	`person_id` String, -- GUID обучающегося
	`person_id_int` Int32 CODEC(Delta, ZSTD),
	`educational_organization_id` Int16, -- Идентификатор образовательной организации
	`parallel_id` Int16,
	`class_id` Int16, -- Идентификатор класса
	`lesson_date` Date32, -- Дата урока
	`lesson_month_digits` String,
	`lesson_month_text` String,
	`lesson_year` UInt16,
	`load_date` Date, -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`subject_id` Int16 CODEC(Delta, ZSTD), -- Идентификатор предмета
	`subject_name` String,
	`mark` Int8, -- Оценка
	PRIMARY KEY(lesson_date, person_id_int, mark)
) ENGINE = MergeTree()
PARTITION BY educational_organization_id
AS SELECT
	floor(randUniform(2, 1300000)) as student_profile_id,
	cast(student_profile_id as String) as person_id,
	cast(person_id as Int32) as  person_id_int,
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date, -- Дата урока
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3, -- Дата загрузки данных
    floor(randUniform(2, 137)) as t,
    educational_organization_id * 136 + t as teacher_id,
    floor(t/9) as subject_id,
    CASE subject_id
    	WHEN 1 THEN 'Математика'
    	WHEN 2 THEN 'Русский язык'
    	WHEN 3 THEN 'Литература'
    	WHEN 4 THEN 'Физика'
    	WHEN 5 THEN 'Химия'
    	WHEN 6 THEN 'География'
    	WHEN 7 THEN 'Биология'
    	WHEN 8 THEN 'Физическая культура'
    	ELSE 'Информатика'
    END as subject_name,
    CASE 
    	WHEN randUniform(0, 2) > 1
    		THEN -1
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(10000000);
```

### Создаем и наполняем таблицу с движком ReplacingMergeTree
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_replacing_merge_tree;
CREATE TABLE learn_db.mart_student_lesson_replacing_merge_tree
(
	`student_profile_id` Int32, -- Идентификатор профиля обучающегося
	`person_id` String, -- GUID обучающегося
	`person_id_int` Int32 CODEC(Delta, ZSTD),
	`educational_organization_id` Int16, -- Идентификатор образовательной организации
	`parallel_id` Int16,
	`class_id` Int16, -- Идентификатор класса
	`lesson_date` Date32, -- Дата урока
	`lesson_month_digits` String,
	`lesson_month_text` String,
	`lesson_year` UInt16,
	`load_date` Date, -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`subject_id` Int16 CODEC(Delta, ZSTD), -- Идентификатор предмета
	`subject_name` String,
	`mark` Int8, -- Оценка
	`version` UInt32,
	`is_deleted` UInt8,
	PRIMARY KEY(lesson_date, person_id_int, mark)
) ENGINE = ReplacingMergeTree(version, is_deleted)
PARTITION BY educational_organization_id
AS SELECT
	student_profile_id,
	person_id,
	person_id_int,
	educational_organization_id,
	parallel_id,
	class_id,
	lesson_date,
	lesson_month_digits,
	lesson_month_text,
	lesson_year,
	load_date,
	t,
	teacher_id,
	subject_id,
	subject_name,
	mark,
	1 as version,
	0 as is_deleted
FROM
	learn_db.mart_student_lesson;
```

### Считаем количество оценок в таблице с движком MergeTree
```sql
SELECT 
	mark, 
	count(*) 
FROM learn_db.mart_student_lesson
GROUP BY
	mark;
```

### Считаем количество оценок в таблице с движком ReplacingMergeTree без применения FINAL
```sql
SELECT 
	mark, 
	count(*) 
FROM learn_db.mart_student_lesson_replacing_merge_tree
GROUP BY
	mark;
```

### Считаем количество оценок в таблице с движком ReplacingMergeTree c применением FINAL
```sql
SELECT 
	mark, 
	count(*) 
FROM learn_db.mart_student_lesson_replacing_merge_tree FINAL
GROUP BY
	mark;
```
