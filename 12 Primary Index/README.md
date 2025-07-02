# Первичный разряженный индекс в Clickhouse

### Создаем таблицу без индекса
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
	PRIMARY KEY tuple()
) ENGINE = MergeTree()
PARTITION BY (educational_organization_id)
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

### Выполняем запрос в clickhouse-client
```sql
SELECT 
	avg(mark) as avg_m,
	teacher_id
FROM 
	learn_db.mart_student_lesson
WHERE 
	lesson_date = subtractDays(today(), 10)
GROUP BY
	teacher_id
ORDER BY avg_m desc
LIMIT 10;
```

### Создаем вторую таблицу с первичным индексом по lesson_date и теми же данными
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_idx;
CREATE TABLE learn_db.mart_student_lesson_idx
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
	PRIMARY KEY lesson_date
) ENGINE = MergeTree()
AS
SELECT
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
	mark
FROM
	learn_db.mart_student_lesson;
```

### Выполняем запрос в clickhouse-client по второй таблице
```sql
SELECT 
	avg(mark) as avg_m,
	teacher_id
FROM 
	learn_db.mart_student_lesson_idx
WHERE 
	lesson_date = subtractDays(today(), 10)
GROUP BY
	teacher_id
ORDER BY avg_m desc
LIMIT 10;
```

### Получаем строки из 1ой гранулы
```sql
SELECT 
	*
FROM 
	learn_db.mart_student_lesson_idx
ORDER BY lesson_date, person_id_int, mark
LIMIT 8192
OFFSET 8192 * 0;
```

### Получаем список всех частей данных таблицы mart_student_lesson_idx
```sql
SELECT path FROM system.parts where table = 'mart_student_lesson_idx' AND active = 1;
```

### Смотрим на гранулы в частях данных
```sql
SELECT * FROM mergeTreeIndex('learn_db', 'mart_student_lesson_idx');
```

### Смотрим на засечки в каждой часте данных
```sql
SELECT
    part_name,
    max(mark_number) as entries
FROM mergeTreeIndex('learn_db', 'mart_student_lesson_idx')
GROUP BY part_name;
```

### Смотрим план выполнения запроса со статистикой по применению индекса
```sql
EXPLAIN indexes = 1
SELECT 
	avg(mark) as avg_m,
	teacher_id
FROM 
	learn_db.mart_student_lesson_idx
WHERE 
	lesson_date = subtractDays(today(), 10)
GROUP BY
	teacher_id
ORDER BY avg_m desc
LIMIT 10;
```

### Пересоздаем таблицу с размером гранулы 4096
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_idx;
CREATE TABLE learn_db.mart_student_lesson_idx
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
	PRIMARY KEY lesson_date
) ENGINE = MergeTree()
SETTINGS index_granularity = 4096
AS
SELECT
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
	mark
FROM
	learn_db.mart_student_lesson;
```

### Пересоздаем таблицу с индексом по трем столбцам
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_idx;
CREATE TABLE learn_db.mart_student_lesson_idx
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
	PRIMARY KEY (lesson_date, person_id_int, mark)
) ENGINE = MergeTree()
AS
SELECT
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
	mark
FROM
	learn_db.mart_student_lesson;
```

### Смотрим эффективность фильтрации по второму полю в индексе
```sql
EXPLAIN indexes = 1
SELECT 
	avg(mark) as avg_m,
	teacher_id
FROM 
	learn_db.mart_student_lesson_idx
WHERE 
	person_id_int = 1000
GROUP BY
	teacher_id
ORDER BY avg_m desc
LIMIT 10;
```

### Смотрим, в какое количество частей данный попадает на самом деле person_id_int = 1000
```sql
SELECT DISTINCT 
	_part
FROM 
	learn_db.mart_student_lesson_idx
WHERE 
	person_id_int = 1000;
```

### Полуаем строки первой гранулы
```sql
SELECT 
	lesson_date, person_id_int, mark
FROM 
	learn_db.mart_student_lesson_idx
ORDER BY 
	lesson_date, person_id_int, mark
LIMIT 8192
OFFSET 8192 * 0;
```

### Получаем количество уникальных значений lesson_date и mark
```sql
SELECT count(DISTINCT lesson_date) FROM learn_db.mart_student_lesson_idx;
SELECT count(DISTINCT mark) FROM learn_db.mart_student_lesson_idx;
```

### Пересоздаем таблицу с первым полем в первичном индексе, в котором меньше всего уникальных значений
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_idx;
CREATE TABLE learn_db.mart_student_lesson_idx
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
	PRIMARY KEY (mark, lesson_date, person_id_int)
) ENGINE = MergeTree()
AS
SELECT
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
	mark
FROM
	learn_db.mart_student_lesson;
```
