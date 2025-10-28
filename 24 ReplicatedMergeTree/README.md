# Код, выполняемый на 1ой реплике

### Создаем реплицируемую таблицу на кластере
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R; 
CREATE TABLE learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R
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
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/mart_student_lesson', '{replica}');
```

### Удаляем таблицу на кластере
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R;
```

### Создаем реплицируемую таблицу на одной ноде кластера, после чего создаем такую же таблицу на второй ноде
```sql
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
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/mart_student_lesson', '{replica}');
```

### Вставляем данные
```sql
INSERT INTO learn_db.mart_student_lesson 
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
FROM numbers(1);
```

### Проверяем, что данные появились в таблице, после чего делаем такую же проверку на второй ноде
```sql
SELECT * FROM learn_db.mart_student_lesson;
```

### Удаляем одну строку, предварительно вместо фигурных скубок указав id строки
```sql
ALTER TABLE learn_db.mart_student_lesson DELETE WHERE student_profile_id = {...};
```

### Проверяем, что строка удалена. Выполняем скрипт на первой и второй нодах
```sql
SELECT * FROM learn_db.mart_student_lesson;
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

###
```sql
```

###
```sql
```

# Код, выполняемый на 2ой реплике

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

