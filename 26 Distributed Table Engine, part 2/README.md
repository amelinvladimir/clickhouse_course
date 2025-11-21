# План выполнения запроса к распределенной таблице

### Останавливаем кластер и затем в конфигурационных файлах серверов ClickHouse 
fs/volumes/clickhouse-01/etc/clickhouse-server/config.d/config.xml
fs/volumes/clickhouse-02/etc/clickhouse-server/config.d/config.xml
fs/volumes/clickhouse-03/etc/clickhouse-server/config.d/config.xml
fs/volumes/clickhouse-04/etc/clickhouse-server/config.d/config.xml
### делаем правки:
```xml
```

### Пересоздаем таблицу с оценками на кластере
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_mergetree ON CLUSTER cluster_2S_2R; 

CREATE TABLE learn_db.mart_student_lesson_mergetree ON CLUSTER cluster_2S_2R
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
) ENGINE = MergeTree();
```

### Создаем распределенную таблицу с оценками
```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson_mergetree_distributed ON CLUSTER cluster_2S_2R; 
CREATE TABLE IF NOT EXISTS learn_db.mart_student_lesson_mergetree_distributed ON CLUSTER cluster_2S_2R
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
	`mark` Nullable(UInt8)
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'mart_student_lesson_mergetree', person_id_int);
```

### Вставляем данные в распределенную таблицу с оценками
```sql
INSERT INTO learn_db.mart_student_lesson_mergetree_distributed
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

### Проверяем количество строк в распределенной таблице и в первом шарде
```sql
SELECT COUNT(*) FROM learn_db.mart_student_lesson_mergetree_distributed;
SELECT COUNT(*) FROM learn_db.mart_student_lesson_mergetree;
```

### Считаем количество уникальных учителей в распределенной таблице
```sql
SELECT 
	uniq(teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

### Смотрим информацию в system.query_log по запросу к распределенной таблице
```sql
SELECT 
	*
FROM 	
	system.query_log
ORDER BY
	event_time DESC;
```

### Смотрим план запроса к распределенной таблице
```sql
EXPLAIN distributed=1
SELECT 
	uniq(teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

# Соединение распределенной и не распределенной таблиц

### Создаем на всех нодах кластера таблицу с учителями
```sql
DROP TABLE IF EXISTS learn_db.teachers ON CLUSTER cluster_2S_2R; 

SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/01/learn_db/teachers';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/01/learn_db/teachers';
SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/02/learn_db/teachers';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/02/learn_db/teachers';

CREATE TABLE learn_db.teachers ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String,
	PRIMARY KEY(teacher_id)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}');
```

### Создаем распределенную таблицу с учителями
```sql
DROP TABLE IF EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R; 
CREATE TABLE IF NOT EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'teachers', teacher_id);
```

### Вставляем данные в таблицу с учителями (выполняем на каждой ноде кластера)
```sql
INSERT INTO learn_db.teachers 
SELECT DISTINCT
	teacher_id,
	CONCAT('Учитель № ', teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

### Проверяем количество строк в распределенной таблице и в локальной таблице
```sql
SELECT COUNT(*) FROM learn_db.teachers_distributed;
SELECT COUNT(*) FROM learn_db.teachers;
SELECT *, _shard_num FROM learn_db.teachers_distributed;
```

### Выполняем запрос с соединением распределенной и не распределенной таблиц
```sql
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

### Смотрим план выполнения запроса с соединением распределенной и не распределенной таблиц
```sql
EXPLAIN
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

### Смотрим план выполнения запроса с соединением распределенной и не распределенной таблиц с добавлением флага distributed = 1
```sql
EXPLAIN distributed=1
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

# Соединяем 2 распределенные таблицы

### Пересоздаем таблицу с учителями
```sql
DROP TABLE IF EXISTS learn_db.teachers ON CLUSTER cluster_2S_2R; 

SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/01/learn_db/teachers';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/01/learn_db/teachers';
SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/02/learn_db/teachers';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/02/learn_db/teachers';

CREATE TABLE learn_db.teachers ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String,
	PRIMARY KEY(teacher_id)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}');
```

### Пересоздаем распределенную таблицу с учителями
```sql
DROP TABLE IF EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R; 
CREATE TABLE IF NOT EXISTS learn_db.teachers_distributed ON CLUSTER cluster_2S_2R
(
	`teacher_id` Int32 CODEC(Delta, ZSTD), -- Идентификатор учителя
	`teacher_name` String
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'teachers', teacher_id);
```

### Создаем локальную (не на всем кластере) временную таблицу с учителями
```sql
DROP TABLE IF EXISTS learn_db.teachers_tmp;
CREATE TABLE learn_db.teachers_tmp AS learn_db.teachers
ENGINE = MergeTree();
```

### Наполняем данными временную таблицу с учителями
```sql
INSERT INTO learn_db.teachers_tmp
SELECT DISTINCT
	teacher_id,
	CONCAT('Учитель № ', teacher_id)
FROM 
	learn_db.mart_student_lesson_mergetree_distributed;
```

### Наполняем распределенную таблицу с учителями данными
```sql
INSERT INTO learn_db.teachers_distributed
SELECT 
	teacher_id,
	teacher_name
FROM 
	learn_db.teachers_tmp;
```

### Смотрим, сколько данных всего в распределенной таблице и в локальной
```sql
SELECT COUNT(*) FROM learn_db.teachers_distributed;
SELECT COUNT(*) FROM learn_db.teachers;
SELECT *, _shard_num FROM learn_db.teachers_distributed;
```

### Пробуем выполнить соединение двух распределенных таблиц с помощью INNER JOIN
```sql
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	INNER JOIN learn_db.teachers_distributed t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

### Выполняем соединение двух распределенных таблиц с помощью GLOBAL JOIN
```sql
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	GLOBAL JOIN learn_db.teachers_distributed t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

### Смотрим план запроса с соединением двух распределенных таблиц с помощью GLOBAL JOIN
```sql
EXPLAIN distributed=1
SELECT 
	t.teacher_name,
	AVG(sl.mark) as avg_mark,
	COUNT(sl.mark) as cnt_mark
FROM
	learn_db.mart_student_lesson_mergetree_distributed sl
	GLOBAL JOIN learn_db.teachers_distributed t
		on sl.teacher_id = t.teacher_id
GROUP BY
	t.teacher_name;
```

