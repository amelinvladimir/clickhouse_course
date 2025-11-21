```sql
DROP TABLE IF EXISTS learn_db.mart_student_lesson ON CLUSTER cluster_2S_2R; 

SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/01/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/01/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '01' FROM ZKPATH '/clickhouse/tables/02/learn_db/mart_student_lesson';
SYSTEM DROP REPLICA '02' FROM ZKPATH '/clickhouse/tables/02/learn_db/mart_student_lesson';

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
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{database}/{table}', '{replica}');

DROP TABLE IF EXISTS learn_db.mart_student_lesson_distributed_person_id_int ON CLUSTER cluster_2S_2R;
CREATE TABLE IF NOT EXISTS learn_db.mart_student_lesson_distributed_person_id_int ON CLUSTER cluster_2S_2R
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
	`mark` Nullable(UInt8) -- Оценка
)
ENGINE = Distributed('cluster_2S_2R', 'learn_db', 'mart_student_lesson', person_id_int);

INSERT INTO learn_db.mart_student_lesson_distributed_person_id_int
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
FROM numbers(1000000);


-- Напишите любой запрос с оконной функцией к распределенной таблице learn_db.mart_student_lesson_distributed_person_id_int. Получите план выполнения запроса и приложите его в качестве результата.

SELECT
	*,
	count(mark) over(partition by subject_id) as subject_mark --всего оценок по предмету
FROM
	learn_db.mart_student_lesson_distributed_person_id_int

EXPLAIN distributed = 1	
SELECT
	*,
	count(mark) over(partition by subject_id) as subject_mark --всего оценок по предмету
FROM
	learn_db.mart_student_lesson_distributed_person_id_int
	
--Expression ((Project names + Projection))
--  Window (Window step for window 'PARTITION BY __table1.subject_id')
--    Sorting (Sorting for window 'PARTITION BY __table1.subject_id')
--      Union
--        Expression ((Before WINDOW + Change column names to column identifiers))
--          ReadFromMergeTree (learn_db.mart_student_lesson)
--        ReadFromRemote (Read from remote replica)
--          BlocksMarshalling
--            Expression ((Before WINDOW + Change column names to column identifiers))
--              ReadFromMergeTree (learn_db.mart_student_lesson)
	
	
-- 2. Напишите запрос к распределенной таблице learn_db.mart_student_lesson_distributed_person_id_int, который выведет следующие столбцы: 
--- id ученика (person_id_int);
--- последняя оценка, полученная учеником (среди всех оценок одного ученика выбираем оценку, у которой макимальная дата lesson_date. Если за максимальную дату несколько оценок, то возьмите максимальуню среди них, то есть с самым большим значением mark)
--Получите план выполнения написанного вами запроса.
--Приложите сам запрос и полученный по нему план.

WITH numbered_marks AS (
	SELECT 
		person_id_int,
		mark,
		row_number() over(partition by person_id_int order by lesson_date desc, mark desc nulls last) as rn
	FROM 
		learn_db.mart_student_lesson_distributed_person_id_int
	WHERE 
		mark IS NOT NULL
)
SELECT
	person_id_int,
	mark
FROM
	numbered_marks
WHERE rn = 1 
```
