# Разбор ДЗ

### Создание таблицы
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson;
CREATE TABLE learn_db.mart_student_lesson
(
    -- Идентификаторы студентов
    `student_profile_id` Int64,                    -- Идентификатор профиля обучающегося
    `person_id` String,                           -- GUID обучающегося
    `person_id_int` Int64,
    
    -- Образовательная организация и класс
    `educational_organization_id` Int16,          -- Идентификатор образовательной организации
    `parallel_id` Int16,
    `class_id` Int16,                             -- Идентификатор класса
    
    -- Временные метки урока
    `lesson_date` Date32,                         -- Дата урока
    `lesson_month_digits` String,
    `lesson_month_text` String,
    `lesson_year` UInt16,
    `load_date` Date,                             -- Дата загрузки данных
    
    -- Метаданные урока
    `t` Int16 CODEC(Delta, ZSTD),
    `teacher_id` Int32,                           -- Идентификатор учителя
    `subject_id` Int16,                           -- Идентификатор предмета
    `subject_name` String,
    
    -- Оценка
    `mark` Nullable(UInt8),                       -- Оценка
    
    PRIMARY KEY (lesson_date)
) 
ENGINE = MergeTree()

-- =============================================
-- НАПОЛНЕНИЕ ТАБЛИЦЫ ТЕСТОВЫМИ ДАННЫМИ
-- =============================================
AS SELECT
    -- Генерация идентификаторов студентов
    floor(randUniform(2, 10000000)) as student_profile_id,
    cast(student_profile_id as String) as person_id,
    cast(person_id as Int32) as person_id_int,
    
    -- Генерация данных об организации и классе
    student_profile_id / 365000 as educational_organization_id,
    student_profile_id / 73000 as parallel_id,
    student_profile_id / 2000 as class_id,
    
    -- Генерация дат уроков
    cast(now() - randUniform(2, 60*60*24*365) as date) as lesson_date,  -- Дата урока
    formatDateTime(lesson_date, '%Y-%m') as lesson_month_digits,
    formatDateTime(lesson_date, '%Y %M') AS lesson_month_text,
    toYear(lesson_date) as lesson_year, 
    lesson_date + rand() % 3 as load_date,                              -- Дата загрузки данных
    
    -- Генерация метаданных
    floor(randUniform(2, 137)) as t,
    educational_organization_id * 136 + t as teacher_id,
    floor(t/9) as subject_id,
    
    -- Названия предметов
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
    
    -- Генерация оценок
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
	
## По крайней мере у одного поля измените тип данных с целью уменьшения занимаемого места	
### Получаем максимальные и минимальные значения в каждой колонке
```sql
SELECT COLUMNS('.*') APPLY min, COLUMNS('.*') APPLY max FROM learn_db.mart_student_lesson;
```

### У полей student_profile_id и person_id_int правим тип с Int64 на UInt32
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_v2;
CREATE TABLE learn_db.mart_student_lesson_v2
(
    -- Идентификаторы студентов
    `student_profile_id` UInt32,                    -- Идентификатор профиля обучающегося
    `person_id` String,                           -- GUID обучающегося
    `person_id_int` UInt32,
    
    -- Образовательная организация и класс
    `educational_organization_id` Int16,          -- Идентификатор образовательной организации
    `parallel_id` Int16,
    `class_id` Int16,                             -- Идентификатор класса
    
    -- Временные метки урока
    `lesson_date` Date32,                         -- Дата урока
    `lesson_month_digits` String,
    `lesson_month_text` String,
    `lesson_year` UInt16,
    `load_date` Date,                             -- Дата загрузки данных
    
    -- Метаданные урока
    `t` Int16 CODEC(Delta, ZSTD),
    `teacher_id` Int32,                           -- Идентификатор учителя
    `subject_id` Int16,                           -- Идентификатор предмета
    `subject_name` String,
    
    -- Оценка
    `mark` Nullable(UInt8),                       -- Оценка
    
    PRIMARY KEY (lesson_date)
) 
ENGINE = MergeTree() AS 
SELECT * FROM learn_db.mart_student_lesson;
```

### Проверяем результат изменений
```sql
SELECT
	table,
	name,
	type,
	data_compressed_bytes,
	data_uncompressed_bytes
FROM
	system.columns
WHERE
	name in ('student_profile_id', 'person_id_int')
	and database = 'learn_db'
ORDER BY name, data_compressed_bytes desc;
```


## Примените по крайней мере к одному столбцу LowCardinality
### Применяем LowCardinality к lesson_month_digits, lesson_month_text, subject_name
 ```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_v3;
CREATE TABLE learn_db.mart_student_lesson_v3
(
    -- Идентификаторы студентов
    `student_profile_id` UInt32,                    -- Идентификатор профиля обучающегося
    `person_id` String,                           -- GUID обучающегося
    `person_id_int` UInt32,
    
    -- Образовательная организация и класс
    `educational_organization_id` Int16,          -- Идентификатор образовательной организации
    `parallel_id` Int16,
    `class_id` Int16,                             -- Идентификатор класса
    
    -- Временные метки урока
    `lesson_date` Date32,                         -- Дата урока
    `lesson_month_digits` LowCardinality(String),
    `lesson_month_text` LowCardinality(String),
    `lesson_year` UInt16,
    `load_date` Date,                             -- Дата загрузки данных
    
    -- Метаданные урока
    `t` Int16 CODEC(Delta, ZSTD),
    `teacher_id` Int32,                           -- Идентификатор учителя
    `subject_id` Int16,                           -- Идентификатор предмета
    `subject_name` LowCardinality(String),
    
    -- Оценка
    `mark` Nullable(UInt8),                       -- Оценка
    
    PRIMARY KEY (lesson_date)
) 
ENGINE = MergeTree() AS 
SELECT * FROM learn_db.mart_student_lesson;
```

SELECT
	table,
	name,
	type,
	data_compressed_bytes,
	data_uncompressed_bytes
FROM
	system.columns
WHERE
	name in ('lesson_month_digits', 'lesson_month_text', 'subject_name')
	and database = 'learn_db'
	and table in ('mart_student_lesson_v2', 'mart_student_lesson_v3')
ORDER BY name, data_compressed_bytes desc;



-- По крайней мере у одного столбца измените кодировку
DROP TABLE IF EXISTS learn_db.mart_student_lesson_v4;
CREATE TABLE learn_db.mart_student_lesson_v4
(
    -- Идентификаторы студентов
    `student_profile_id` UInt32,                    -- Идентификатор профиля обучающегося
    `person_id` String,                           -- GUID обучающегося
    `person_id_int` UInt32,
    
    -- Образовательная организация и класс
    `educational_organization_id` Int16,          -- Идентификатор образовательной организации
    `parallel_id` Int16 CODEC(ZSTD),
    `class_id` Int16 CODEC(T64, ZSTD),                             -- Идентификатор класса
    
    -- Временные метки урока
    `lesson_date` Date32 CODEC(Delta, ZSTD),                         -- Дата урока
    `lesson_month_digits` LowCardinality(String),
    `lesson_month_text` LowCardinality(String),
    `lesson_year` UInt16,
    `load_date` Date CODEC(Delta, ZSTD),                             -- Дата загрузки данных
    
    -- Метаданные урока
    `t` Int16 CODEC(Delta, ZSTD),
    `teacher_id` Int32 CODEC(T64, ZSTD),                           -- Идентификатор учителя
    `subject_id` Int16 CODEC(T64, LZ4),                           -- Идентификатор предмета
    `subject_name` LowCardinality(String),
    
    -- Оценка
    `mark` Nullable(UInt8),                       -- Оценка
    
    PRIMARY KEY (lesson_date)
) 
ENGINE = MergeTree() AS 
SELECT * FROM learn_db.mart_student_lesson;

SELECT
	table,
	name,
	type,
	data_compressed_bytes,
	data_uncompressed_bytes,
	compression_codec
FROM
	system.columns
WHERE
	name in ('parallel_id', 'class_id', 'teacher_id', 'subject_id', 'lesson_date', 'load_date')
	and database = 'learn_db'
	and table in ('mart_student_lesson_v3', 'mart_student_lesson_v4')
ORDER BY name, data_compressed_bytes desc;
