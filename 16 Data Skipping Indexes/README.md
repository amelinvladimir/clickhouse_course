# Индекс пропуска данных типа minmax

### Создаем таблицы с уроками и оценками учеников
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
	PRIMARY KEY(class_id)
) ENGINE = MergeTree();
```

### Вставляем данные в таблицу со сравнением скорости вставки с индексами пропуска данных и без
```sql
-- сравниваем скорость вставки в таблицу с индексом и без

-- 4.017
-- 7.038
INSERT INTO mart_student_lesson
(
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
	load_date, t, 
	teacher_id, 
	subject_id, 
	subject_name, 
	mark
)
SELECT
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

### Получаем данные с фильтрацией по первичному индексу
```sql
-- 0.049
-- result 15366
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	class_id = 200;

explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	class_id = 200;
```

### Получаем данные по столбцу student_profile_id без применения индекса
```sql
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;

explain indexes = 1
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;
```

### Создаем индекс пропуска данных по полю student_profile_id
```sql
ALTER TABLE learn_db.mart_student_lesson ADD INDEX idx_student_profile_id student_profile_id TYPE minmax GRANULARITY 2;
ALTER TABLE learn_db.mart_student_lesson MATERIALIZE INDEX idx_student_profile_id;
```

### Замеряем изменения в параметрах выполнения запроса
```sql
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;

explain indexes = 1
SELECT 	*
FROM
	learn_db.mart_student_lesson
WHERE 
	student_profile_id = 8;
```

### Смотрим, в какой папке находятся части данных таблицы learn_db.mart_student_lesson
```sql
SELECT * FROM system.parts WHERE table = 'mart_student_lesson';
```

# Индекс пропуска данных типа set

### Делаем первоначальные замеры скорости
```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;

explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

### Создаем и материализуем индекс
```sql
ALTER TABLE learn_db.mart_student_lesson
ADD INDEX teacher_id_set_index (teacher_id) TYPE set(100) GRANULARITY 1;


ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX teacher_id_set_index;
```

### Проверяем, поменялось ли время выполнения запроса
```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;

explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

### Убираем ограничение на максимальное количество уникальных значений 
```sql
ALTER TABLE learn_db.mart_student_lesson DROP INDEX IF EXISTS teacher_id_set_index;

ALTER TABLE learn_db.mart_student_lesson
ADD INDEX teacher_id_set_index (teacher_id) TYPE set(0) GRANULARITY 1;
ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX teacher_id_set_index;
```

### Проверяем, поменялось ли время выполнения запроса и план выполнения
```sql
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;

explain indexes = 1
SELECT 
	*
FROM
	learn_db.mart_student_lesson
WHERE 
	teacher_id = 2222;
```

# Индексы пропуска данных фильтр Блума

### Создаем и наполняем таблицу git.file_changes
```sql
DROP DATABASE IF EXISTS git;
CREATE DATABASE git;

CREATE TABLE git.file_changes
(
    change_type Enum('Add' = 1, 'Delete' = 2, 'Modify' = 3, 'Rename' = 4, 'Copy' = 5, 'Type' = 6),
    path LowCardinality(String),
    old_path LowCardinality(String),
    file_extension LowCardinality(String),
    lines_added UInt32,
    lines_deleted UInt32,
    hunks_added UInt32,
    hunks_removed UInt32,
    hunks_changed UInt32,
    commit_hash String,
    author LowCardinality(String),
    time DateTime,
    commit_message String,
    commit_files_added UInt32,
    commit_files_deleted UInt32,
    commit_files_renamed UInt32,
    commit_files_modified UInt32,
    commit_lines_added UInt32,
    commit_lines_deleted UInt32,
    commit_hunks_added UInt32,
    commit_hunks_removed UInt32,
    commit_hunks_changed UInt32
) ENGINE = MergeTree ORDER BY time;

INSERT INTO git.file_changes SELECT *
FROM s3('https://datasets-documentation.s3.amazonaws.com/github/commits/clickhouse/file_changes.tsv.xz', 'TSV', 'change_type Enum(\'Add\' = 1, \'Delete\' = 2, \'Modify\' = 3, \'Rename\' = 4, \'Copy\' = 5, \'Type\' = 6), path LowCardinality(String), old_path LowCardinality(String), file_extension LowCardinality(String), lines_added UInt32, lines_deleted UInt32, hunks_added UInt32, hunks_removed UInt32, hunks_changed UInt32, commit_hash String, author LowCardinality(String), time DateTime, commit_message String, commit_files_added UInt32, commit_files_deleted UInt32, commit_files_renamed UInt32, commit_files_modified UInt32, commit_lines_added UInt32, commit_lines_deleted UInt32, commit_hunks_added UInt32, commit_hunks_removed UInt32, commit_hunks_changed UInt32')

```

### Замеряем время выполнения и смотрим план выполнения запроса поиск по подстроке
```sql
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';

explain indexes = 1
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';
```

### Создаем n-грам индекс фильтра Блума
```sql
ALTER TABLE git.file_changes
ADD INDEX commit_message_tokenbf_v1_index commit_message TYPE ngrambf_v1(3, 10000, 3, 7) GRANULARITY 1

ALTER TABLE git.file_changes
MATERIALIZE INDEX commit_message_tokenbf_v1_index;
```

### Проверяем поменялось ли время выполнения запроса и план выполнения запроса
```sql
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';

explain indexes = 1
SELECT * FROM git.file_changes WHERE commit_message LIKE '%MYSQL%';
```

## Индексы пропуска данных фильтр Блума по токенам

### Создаем и материализуем индекс по токенам фильтра Блума
```sql
ALTER TABLE git.file_changes
ADD INDEX commit_message_tokenbf_v1_idx commit_message TYPE tokenbf_v1(512, 3, 0) GRANULARITY 1;

ALTER TABLE git.file_changes
MATERIALIZE INDEX commit_message_tokenbf_v1_idx;
```

### Замеряем скорость выполнения запроса и смотрим план выполнения запроса с поиском по токену
```sql
explain indexes = 1
SELECT * FROM git.file_changes WHERE hasToken(commit_message, '9019');

SELECT * FROM git.file_changes WHERE hasToken(commit_message, '9019');
```

## Индексы пропуска данных фильтр Блума

### Замеряем скорость выполнения запроса и смотрим план выполнения запроса с поиском по равенству строки
```sql
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'

explain indexes = 1
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'
```

### Создаем и материализуем индекс фильтра Блума
```sql
ALTER TABLE git.file_changes
ADD INDEX commit_hash_bloom_filter_idx commit_hash TYPE bloom_filter(0.001) GRANULARITY 1

ALTER TABLE git.file_changes
MATERIALIZE INDEX commit_hash_bloom_filter_idx;
```

### Проверяем поменялось ли время выполнения запроса и план выполнения запроса 
```sql
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'

explain indexes = 1
SELECT 
	*
FROM 
	git.file_changes
WHERE 
	commit_hash = 'b841a96c3998b7fcf5c53cfd3dd502122785d8f3'
```

# Изменение скорости вставки данных

### Добавляем индекс пропуска данных в таблицу learn_db.mart_student_lesson
```sql
ALTER TABLE learn_db.mart_student_lesson
ADD INDEX student_profile_id_bloom_filter_idx student_profile_id TYPE bloom_filter(0.001) GRANULARITY 1

ALTER TABLE learn_db.mart_student_lesson
MATERIALIZE INDEX student_profile_id_bloom_filter_idx;
```


### Вставляем в таблицу 10 000 000 строк
```sql

INSERT INTO mart_student_lesson
(
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
	load_date, t, 
	teacher_id, 
	subject_id, 
	subject_name, 
	mark
)
SELECT
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
