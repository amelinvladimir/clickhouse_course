# Проекции

### Пересоздаем таблицу и наполняем ее данными
```sqlDROP TABLE IF EXISTS learn_db.mart_student_lesson; 
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32 , -- Идентификатор профиля обучающегося
	`person_id` String , -- GUID обучающегося
	`person_id_int` Int32 ,
	`educational_organization_id` Int16 , -- Идентификатор образовательной организации
	`parallel_id` Int16 ,
	`class_id` Int16 , -- Идентификатор класса
	`lesson_date` Date32 , -- Дата урока
	`lesson_month_digits` String ,
	`lesson_month_text` String ,
	`lesson_year` UInt16 ,
	`load_date` Date , -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 , -- Идентификатор учителя
	`subject_id` Int16 , -- Идентификатор предмета
	`subject_name` String ,
	`mark` Nullable(UInt8) , -- Оценка
	PROJECTION educational_organization_id_pk_projection
	(
	    SELECT 
	    	subject_name,
	    	mark,
	    	educational_organization_id,
	    	lesson_date
	    ORDER BY educational_organization_id, lesson_date
	),
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

### Находим первый запрос, который приходит из DataLens
```sql
SELECT * FROM system.query_log WHERE http_user_agent = 'DataLens' ORDER BY event_time DESC;
```

### Исследуем эффективность выполнения первого запроса
```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.lesson_date BETWEEN toDate32('2024-08-31') AND toDate32('2025-08-30') AND t1.educational_organization_id IN (1)
GROUP BY res_0
LIMIT 1000001;
```

### Получаем второй запрос и исследуем его эффективность
```sql
SELECT * FROM system.query_log WHERE http_user_agent = 'DataLens' ORDER BY event_time DESC;

EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.educational_organization_id IN (1) AND t1.lesson_date BETWEEN toDate32('2024-08-31') AND toDate32('2025-08-30')
GROUP BY res_0
LIMIT 1000001;
```

### Пробуем оптимизировать второй запрос, добавив в первичный индекс поле educational_organization_id
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson; 
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32 , -- Идентификатор профиля обучающегося
	`person_id` String , -- GUID обучающегося
	`person_id_int` Int32 ,
	`educational_organization_id` Int16 , -- Идентификатор образовательной организации
	`parallel_id` Int16 ,
	`class_id` Int16 , -- Идентификатор класса
	`lesson_date` Date32 , -- Дата урока
	`lesson_month_digits` String ,
	`lesson_month_text` String ,
	`lesson_year` UInt16 ,
	`load_date` Date , -- Дата загрузки данных
	`t` Int16 CODEC(Delta, ZSTD),
	`teacher_id` Int32 , -- Идентификатор учителя
	`subject_id` Int16 , -- Идентификатор предмета
	`subject_name` String ,
	`mark` Nullable(UInt8) , -- Оценка
	PRIMARY KEY(lesson_date, educational_organization_id)
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

### Проверяем эффективность второго запроса
```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.educational_organization_id IN (1) AND t1.lesson_date BETWEEN toDate32('2024-08-31') AND toDate32('2025-08-30')
GROUP BY res_0
LIMIT 1000001;
```

### Смотрим, сколько времени выполняются шаги запроса
```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
SELECT * FROM system.processors_profile_log WHERE query_id = '...' order by processor_uniq_id;
SELECT SUM(elapsed_us) FROM system.processors_profile_log WHERE query_id = '...';
```

### Добавляем и материализуем проекцию с другим первичным индексом
```sql
ALTER TABLE learn_db.mart_student_lesson
ADD PROJECTION educational_organization_id_pk_projection
(
    SELECT 
    	subject_name,
    	mark,
    	educational_organization_id,
    	lesson_date
    ORDER BY educational_organization_id, lesson_date
);

-- Materialize it so existing data is built in the new order
ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE PROJECTION educational_organization_id_pk_projection;
```

### Проверяем как эффективность второго запроса и время выполнения его шагов
```sql
EXPLAIN indexes = 1
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.educational_organization_id IN (1) AND t1.lesson_date BETWEEN toDate32('2024-08-31') AND toDate32('2025-08-30')
GROUP BY res_0
LIMIT 1000001;

SELECT * FROM system.query_log ORDER BY event_time DESC;
SELECT * FROM system.processors_profile_log WHERE query_id = '...' order by processor_uniq_id;
SELECT SUM(elapsed_us) FROM system.processors_profile_log WHERE query_id = '...';
```

### Возможности работы с проекциями
```sql
-- создание
ALTER TABLE learn_db.mart_student_lesson
ADD PROJECTION educational_organization_id_pk_projection
(
    SELECT 
    	subject_name,
    	mark,
    	educational_organization_id,
    	lesson_date
    ORDER BY educational_organization_id, lesson_date
);

-- материализация
ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE PROJECTION educational_organization_id_pk_projection;

-- очистка данных из проекции
ALTER TABLE learn_db.mart_student_lesson
CLEAR PROJECTION educational_organization_id_pk_projection;

-- удаление проекции
ALTER TABLE learn_db.mart_student_lesson
DROP PROJECTION educational_organization_id_pk_projection;
```

### Получение информации о проекциях. Сохраняем результат второго запроса
```sql
SELECT * FROM system.projections;
SELECT * FROM system.projection_parts;
```

### Выполняем запрос и сохраняем результат
```sql
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.educational_organization_id IN (1) AND t1.lesson_date BETWEEN toDate32('2024-08-31') AND toDate32('2025-08-30')
GROUP BY res_0
LIMIT 1000001;
```

### Удаляем часть данных из оригинальной таблицы
```sql
ALTER TABLE learn_db.mart_student_lesson
DELETE WHERE mark = 2;
```

### Проверяем, изменился ли размер частей данных после удаления строк из оригинальной таблицы
```sql
SELECT * FROM system.projection_parts;
```

### Проверяем, изменился результат запроса после удаления строк из оригинальной таблицы
```sql
SELECT t1.subject_name AS res_0, avg(t1.mark) AS res_1
FROM learn_db.mart_student_lesson AS t1
WHERE t1.educational_organization_id IN (1) AND t1.lesson_date BETWEEN toDate32('2024-08-31') AND toDate32('2025-08-30')
GROUP BY res_0
LIMIT 1000001;
```

