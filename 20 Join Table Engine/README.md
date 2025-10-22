# Движок таблиц типа Join

### Пересоздаем таблицу learn_db.mart_student_lesson и наполняем ее данными
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

### Получаем список школ с идентификаторами и названиями
```sql
SELECT DISTINCT
	educational_organization_id,
	'Школа № ' || educational_organization_id as educational_organization_name
FROM	
	learn_db.mart_student_lesson;
```

### Создаем таблицу с движком Join
```sql
DROP TABLE IF EXISTS educational_organization_id_join;
CREATE TABLE educational_organization_id_join(
	`educational_organization_id` Int16, 
	`educational_organization_name` String
) ENGINE = Join(ANY, LEFT, educational_organization_id);
```

### Наполняем таблицу с движком Join данными
```sql
INSERT INTO educational_organization_id_join
SELECT DISTINCT
	educational_organization_id,
	'Школа № ' || educational_organization_id as educational_organization_name
FROM	
	learn_db.mart_student_lesson;
```

### Получаем данные из созданной таблицы
```sql
SELECT * FROM educational_organization_id_join;
```

### Смотрим план выполнения запроса получения данных из таблицы с движком Join
```sql
EXPLAIN header = 1, actions = 1, indexes = 1
SELECT * FROM educational_organization_id_join;
```

### Выполняем запрос, в котором присоединяем таблицу с движком типа Join
```sql
SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN educational_organization_id_join j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

### Получаем граф конвеера запроса и визуализируем в [https://dreampuf.github.io/GraphvizOnline/](https://dreampuf.github.io/GraphvizOnline/)
```sql
EXPLAIN pipeline graph = 1, compact = 0 
SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN educational_organization_id_join j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

### Получаем информацию о времени выполнения шагов запроса с соединением таблиц
```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
SELECT * FROM system.processors_profile_log WHERE query_id = '[id запроса]' order by processor_uniq_id;
```

### Получаем суммарное время выполнения всех шагов запроса
```sql
SELECT sum(elapsed_us) FROM system.processors_profile_log WHERE query_id = '[id запроса]' AND processor_uniq_id LIKE 'JoiningTransform_%';
```

### Создаем 2ой вариант таблицы со списком школ с движком MergeTree
```sql
DROP TABLE IF EXISTS educational_organization_id_mergetree;
CREATE TABLE educational_organization_id_mergetree(
	`educational_organization_id` Int16, 
	`educational_organization_name` String
) ENGINE = MergeTree()
ORDER BY educational_organization_id;
```

### Наполняем таблицу с движком MergeTree данными
```sql
INSERT INTO educational_organization_id_mergetree
SELECT DISTINCT
	educational_organization_id,
	'Школа № ' || educational_organization_id as educational_organization_name
FROM	
	learn_db.mart_student_lesson;
```

Смотрим план выполнения запроса соеднения с таблицей с движком MergeTree
###
```sql
EXPLAIN header = 1, actions = 1, indexes = 1
SELECT * FROM educational_organization_id_mergetree;

SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN educational_organization_id_mergetree j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

### Получаем конвеер графа выполнения запроса с соединением с таблицой с движком MergeTree 
```sql
EXPLAIN pipeline graph = 1, compact = 0
SELECT 
	j.educational_organization_name,
	avg(mark) as avg_mark
FROM
	learn_db.mart_student_lesson l
	ANY LEFT JOIN educational_organization_id_mergetree j
		USING (educational_organization_id)
GROUP BY 
	j.educational_organization_name;
```

### Получаем информацию о времени выполнения шагов запроса с соединением таблиц
```sql
SELECT * FROM system.query_log ORDER BY event_time DESC;
SELECT * FROM system.processors_profile_log WHERE query_id = '[id запроса]' order by processor_uniq_id;
```

### Получаем суммарное время выполнения шагов запроса, выполняющих подготовку к соединению таблицы
```sql
SELECT sum(elapsed_us) FROM system.processors_profile_log WHERE query_id = '[id запроса]' AND processor_uniq_id LIKE 'JoiningTransform_%';
```
