### Пересоздаем таблицу и наполняем ее данными
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson; 
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
	`mark` Nullable(UInt8), -- Оценка
	PRIMARY KEY(lesson_date)
) ENGINE = MergeTree()
AS SELECT
	floor(randUniform(2, 10000000)) as student_profile_id,
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
    		THEN NULL
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(10000000);
```

### Запрос для исследования
```sql
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

## AST

### Получаем абстрактное синтаксическое дерево 
```sql
EXPLAIN AST 
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

### Получаем абстрактное синтаксическое дерево в виде графа. [Страница для визуализации графа](https://dreampuf.github.io/GraphvizOnline/)
```sql
EXPLAIN AST graph = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

### Получаем абстрактное синтаксическое дерево с флагом оптимизации (проверяет корректность указанных таблиц и колонок)
```sql
EXPLAIN AST optimize = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

## Query Tree

### Получаем дерево запроса
```sql
EXPLAIN QUERY TREE 
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

### Получаем дерево запроса без оптимизаций (проходов)
```sql
EXPLAIN QUERY TREE run_passes = 0
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

### Получаем дерево запроса со всеми оптимизациями и выводом списка примененных оптимизаций (проходов)
```sql
EXPLAIN QUERY TREE run_passes = 1, dump_passes = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

###  Получаем дерево запроса со применением 2-х оптимизаций и выводом списка примененных оптимизаций (проходов)
```sql
EXPLAIN QUERY TREE dump_passes = 1, passes = 2
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

###  Получаем AST со применением 2-х оптимизаций и выводом списка примененных оптимизаций (проходов)
```sql
EXPLAIN QUERY TREE dump_passes = 1, passes = 2, dump_tree = 0, dump_ast = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

## План выполнения запроса

### Получение плана выполнения запроса
```sql
EXPLAIN PLAN -- или EXPLAIN
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

### Получение плана выполнения запроса с описанием обрабатываемых колонок на каждом этапе
```sql
EXPLAIN PLAN header = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

### Получение плана выполнения запроса с описанием каждого этапа
```sql
EXPLAIN PLAN actions = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

### Получение плана выполнения запроса с информацией о результатах применения индексов
```sql
EXPLAIN PLAN indexes = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

### Получение плана выполнения запроса в формате json
```sql
EXPLAIN PLAN json = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

### Получение плана выполнения запроса в лаконичной форме без описаний
```sql
EXPLAIN PLAN description = 0
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc
```

## PIPELINE

### Получение pipeline запроса
```sql
EXPLAIN PIPELINE	
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

### Получение pipeline запроса для представления в виде графа
```sql
EXPLAIN PIPELINE graph = 1
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

### Получение pipeline запроса для представления в виде графа в расширенном формате
```sql
EXPLAIN PIPELINE graph = 1, compact = 0
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

## Время выполнения этапов запроса

### Выполняем запрос
```sql
SELECT 
	subject_name,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	lesson_date >= today() - 10
GROUP BY
	subject_name
ORDER BY 
	avg_mark desc;
```

### Находим запись о выполнении запроса в query_log и берем его query_id
```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
```

### Подставляем query_id в запрос и получаем время выполнения и другую информацию по каждому этапу запроса
```sql
SELECT * FROM system.processors_profile_log WHERE query_id = '...';
```

## Пример оптимизации

### Оригинальный запрос
```sql
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

### Проверяем эффективность применения индексов
```sql
EXPLAIN indexes = 1
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

### Смотрим, какие этапы выполнения запроса заняли больше всего времени
```sql
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;

SELECT * FROM system.query_log ORDER BY event_time DESC;
SELECT * FROM system.processors_profile_log WHERE query_id = '[подставьте query_id запроса]' order by processor_uniq_id;
```

### Получаем граф конвеера запроса. [Страница для визуализации графа](https://dreampuf.github.io/GraphvizOnline/)
```sql
EXPLAIN PIPELINE graph = 1, compact = 0
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```

### Для оптимизации пересоздаем таблицу с другим первичным индексом
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson; 
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
	`mark` Nullable(UInt8), -- Оценка
	PRIMARY KEY(subject_id, lesson_date)
) ENGINE = MergeTree()
AS SELECT
	floor(randUniform(2, 10000000)) as student_profile_id,
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
    		THEN NULL
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(10000000);
```

### Смотрим, как поменялось время выполнения этапов запроса
```sql
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;

SELECT * FROM system.query_log ORDER BY event_time DESC;
SELECT * FROM system.processors_profile_log WHERE query_id = '[подставьте query_id запроса]' order by processor_uniq_id;
```

### Проверяем, как поменялась эффективность применения индекса
```sql
EXPLAIN indexes = 1
SELECT 
	educational_organization_id,
	ROUND(AVG(mark), 2) as avg_mark
FROM
	learn_db.mart_student_lesson
WHERE
	subject_id = 3
GROUP BY
	educational_organization_id
ORDER BY 
	avg_mark desc;
```
