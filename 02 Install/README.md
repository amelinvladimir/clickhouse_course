# Разворот учебного проекта

## Этап 0. Установка git, если не установлен

#### Шаг 1. Скачать дистрибутив с [официального сайта git](https://git-scm.com/downloads).
#### Шаг 2. Установить git из скачанного дистрибутива.

## Этап 1. Установка DBeaver (если не установлен)

Смотри в первом этапе [инструкции](https://github.com/amelinvladimir/sql_course/blob/main/%D0%A3%D1%80%D0%BE%D0%BA%201.2%20%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0%20%D0%9F%D0%9E/README.md)

## Этап 2. Установка Docker Desktop (если не установлен)

### На Windows
[Инструкция по установке](https://github.com/amelinvladimir/docker_course/blob/main/%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0%20Docker%20%D0%BD%D0%B0%20Windows%2010/README.md)
### На Mac OS
[Инструкция по установке](https://github.com/amelinvladimir/docker_course/blob/main/%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0%20Docker%20%D0%BD%D0%B0%20Mac%20OS/README.md)

## Этап 3. Разворот однонодного Clickhouse в Docker версии 25.4
```console
docker run --name clickhouse-course -e CLICKHOUSE_DB=learn_db -e CLICKHOUSE_USER=username -e CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1 -e CLICKHOUSE_PASSWORD=password -p 8123:8123 -p 9000:9000/tcp -d -v clickhouse-logs:/var/log/clickhouse-server -v clickhouse-data:/var/lib/clickhouse clickhouse:25.4
```

## Этап 4. Подключаемся к Clickhouse из DBeaver

Параметры подключения:
* host: localhost
* port: 8123
* DB: learn_db
* user: username
* password: password

В свойствах драйвера устанавливает у параметра socket_timeout значение 300000.

## Этап 5. Создаем витрину данных в Clickhouse

```sql
CREATE TABLE learn_db.mart_student_lesson
(
	`student_profile_id` Int32, -- Идентификатор профиля обучающегося
	`person_id` String, -- GUID обучающегося
	`person_id_int` Int32 CODEC(Delta, ZSTD),
	`educational_organization_id` Int16, -- Идентификатор образовательной организации
	`parallel_id` Int16,
	`class_id` Int16, -- Идентификатор класса в ЭЖД
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
	PRIMARY KEY(lesson_year)
) ENGINE = MergeTree() 
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
    		THEN NULL
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(100000000);
```

Может добавить данных в витрину запросом:
```sql
INSERT INTO learn_db.mart_student_lesson
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
	load_date,
	t,
	teacher_id,
	subject_id,
	subject_name,
	mark
)
SELECT
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
    		THEN NULL
    		ELSE 
    			CASE
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 5 THEN ROUND(randUniform(4, 5))
	    			WHEN ROUND(randUniform(0, 5)) + subject_id < 9 THEN ROUND(randUniform(3, 5))
	    			ELSE ROUND(randUniform(2, 5))
    			END				
    END AS mark
FROM numbers(100000000);

```

PS: 
* адрес дашборда Clickhouse: http://localhost:8123/dashboard
* адрес web интерфейса для выполнения запросов: http://localhost:8123/play

## Этап 6. Разворачиваем Datalens в Docker

* Открываем окно терминала.
* Создаем новую папку для Datalens.
* Переход в созданную паапку.
* Выполняем команды:

```console
git clone https://github.com/datalens-tech/datalens
cd datalens
docker compose up -d
```
[Datalens](https://github.com/datalens-tech/datalens)

## Этап 7. Организуем сетевую связность между Datalens и Clickhouse в Docker
```console
docker network connect datalens_default clickhouse-course
```

## Этап 8. Логинимся в Datalens
* Host: http://localhost:8080/
* User: admin
* Password: admin

## Этап 9. Создаем Workbook
<img width="279" alt="image" src="https://github.com/user-attachments/assets/abc8142a-0b90-43a0-8cd4-84418a610733" />

Название: clickhouse_course

## Этап 10. Создаем подключение к Clickhouse
<img width="208" alt="image" src="https://github.com/user-attachments/assets/bb5bc065-a920-497a-ac18-e1f33a7f1df3" />
<img width="196" alt="image" src="https://github.com/user-attachments/assets/015e75a5-02cd-4506-91a3-9fc900a9c7f0" />
<img width="746" alt="image" src="https://github.com/user-attachments/assets/b289aa28-f3b9-4d51-9174-5a516dcc122a" />

Параметры подключения:
* host: clickhouse-course
* port: 8123
* user: username
* password: password
* TSL: Off

## Этап 11. Создаем чарты и дашборд согласно видеоинструкции
<img width="1173" alt="image" src="https://github.com/user-attachments/assets/b35dabe4-5cc0-45b3-8009-fa066b8da13d" />
