# Сжатие и кодирование данных

## Скачиваем набор данных с [https://clickhouse.com/blog/real-world-data-noaa-climate-data](https://clickhouse.com/blog/real-world-data-noaa-climate-data)

### Создаем папку и переходим в нее
```bash
mkdir weather_data
cd weather_data
```

### Внутри контейнера скачиваем файлы
```bash
for i in {1900..1930}; do wget https://noaa-ghcn-pds.s3.amazonaws.com/csv.gz/${i}.csv.gz; done
```

### Объединяем в один файл
```bash
for i in {1900..1930}
do
clickhouse-local --query "SELECT station_id,
       toDate32(date) as date,
       anyIf(value, measurement = 'TAVG') as tempAvg,
       anyIf(value, measurement = 'TMAX') as tempMax,
       anyIf(value, measurement = 'TMIN') as tempMin,
       anyIf(value, measurement = 'PRCP') as precipitation,
       anyIf(value, measurement = 'SNOW') as snowfall,
       anyIf(value, measurement = 'SNWD') as snowDepth,
       anyIf(value, measurement = 'PSUN') as percentDailySun,
       anyIf(value, measurement = 'AWND') as averageWindSpeed,
       anyIf(value, measurement = 'WSFG') as maxWindSpeed,
       toUInt8OrZero(replaceOne(anyIf(measurement, startsWith(measurement, 'WT') AND value = 1), 'WT', '')) as weatherType
FROM file('$i.csv.gz', CSV, 'station_id String, date String, measurement String, value Int64, mFlag String, qFlag String, sFlag String, obsTime String')
WHERE qFlag = ''
GROUP BY station_id, date
ORDER BY station_id, date FORMAT TSV" >> "noaa.tsv";
done
```

### Скачиваем еще один файл
```bash
wget http://noaa-ghcn-pds.s3.amazonaws.com/ghcnd-stations.txt
```

### Обогощаем данные
```bash
clickhouse-local --query "WITH stations AS (SELECT id, lat, lon, elevation, name FROM file('ghcnd-stations.txt', Regexp, 'id String, lat Float64, lon Float64, elevation Float32, name String'))
SELECT station_id,
       date,
       tempAvg,
       tempMax,
       tempMin,
       precipitation,
       snowfall,
       snowDepth,
       percentDailySun,
       averageWindSpeed,
       maxWindSpeed,
       weatherType,
       tuple(lon, lat) as location,
       elevation,
       name
FROM file('noaa.tsv', TSV,
          'station_id String, date Date32, tempAvg Int32, tempMax Int32, tempMin Int32, precipitation Int32, snowfall Int32, snowDepth Int32, percentDailySun Int8, averageWindSpeed Int32, maxWindSpeed Int32, weatherType UInt8') as noaa LEFT OUTER
         JOIN stations ON noaa.station_id = stations.id FORMAT TSV SETTINGS format_regexp='^(.{11})\s+(\-?\d{1,2}\.\d{4})\s+(\-?\d{1,3}\.\d{1,4})\s+(\-?\d*\.\d*)\s+(.*?)\s+.*$'" > noaa_enriched.tsv
```

### Создаем таблицу
```sql
CREATE TABLE noaa
(
   `station_id` LowCardinality(String),
   `date` Date32,
   `tempAvg` Int32 COMMENT 'Average temperature (tenths of a degrees C)',
   `tempMax` Int32 COMMENT 'Maximum temperature (tenths of degrees C)',
   `tempMin` Int32 COMMENT 'Minimum temperature (tenths of degrees C)',
   `precipitation` UInt32 COMMENT 'Precipitation (tenths of mm)',
   `snowfall` UInt32 COMMENT 'Snowfall (mm)',
   `snowDepth` UInt32 COMMENT 'Snow depth (mm)',
   `percentDailySun` UInt8 COMMENT 'Daily percent of possible sunshine (percent)',
   `averageWindSpeed` UInt32 COMMENT 'Average daily wind speed (tenths of meters per second)',
   `maxWindSpeed` UInt32 COMMENT 'Peak gust wind speed (tenths of meters per second)',
   `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
   `location` Point,
   `elevation` Float32,
   `name` LowCardinality(String)
) ENGINE = MergeTree() ORDER BY (station_id, date);
```

### Наполняем таблицу данными
```sql
INSERT INTO noaa(
		station_id, 
		date, 
		tempAvg, 
		tempMax, 
		tempMin, 
		precipitation, 
		snowfall, 
		snowDepth, 
		percentDailySun, 
		averageWindSpeed, 
		maxWindSpeed, 
		weatherType, 
		location, 
		elevation, 
		name
	)
FROM
	INFILE '/weather/noaa_enriched.tsv' FORMAT TSV;
```

### Запрос для проверки объема читаемых данных
```sql
SELECT
    tempMax / 10 AS maxTemp,
    location,
    name,
    date
FROM default.noaa
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5
```

# Эксперементируем с кодеками и сжатием

### Создаем таблицу с первыми вариантами кодеков и сжатия и наполянем ее данными
```sql
DROP TABLE IF EXISTS default.noaa_codec_v1;
CREATE TABLE default.noaa_codec_v1
(
   `station_id` String COMMENT 'Id of the station at which the measurement as taken',
   `date` Date32,
   `tempAvg` Int64 COMMENT 'Average temperature (tenths of a degrees C)',
   `tempMax` Int64 COMMENT 'Maximum temperature (tenths of degrees C)',
   `tempMin` Int64 COMMENT 'Minimum temperature (tenths of degrees C)',
   `precipitation` Int64 COMMENT 'Precipitation (tenths of mm)',
   `snowfall` Int64 COMMENT 'Snowfall (mm)',
   `snowDepth` Int64 COMMENT 'Snow depth (mm)',
   `percentDailySun` Int64 COMMENT 'Daily percent of possible sunshine (percent)',
   `averageWindSpeed` Int64 COMMENT 'Average daily wind speed (tenths of meters per second)',
   `maxWindSpeed` Int64 COMMENT 'Peak gust wind speed (tenths of meters per second)',
   `weatherType` String,
   `location` Point,
   `elevation` Float64,
   `name` String
) ENGINE = MergeTree() ORDER BY (station_id, date) AS
SELECT * FROM default.noaa;
```

### Смотрим на размер столбцов в 1 версии таблицы
```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v1'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

### Смотрим общий размер таблицы версии 1
```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v1'
```

### Выполняем диагностический запрос
```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM default.noaa_codec_v1
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

# Выбор оптимального типа данных

### Смотрим максимальные и минимальные значений в столбцах. Выполнять в clickhouse-client
```sql
SELECT
    COLUMNS('Wind|temp|snow|pre') APPLY min,
    COLUMNS('Wind|temp|snow|pre') APPLY max
FROM default.noaa
FORMAT Vertical
```

### Создаем таблицы 2 версии
```sql
DROP TABLE IF EXISTS default.noaa_codec_v2;
CREATE TABLE default.noaa_codec_v2
(
  `station_id` String COMMENT 'Id of the station at which the measurement as taken',
  `date` Date32,
  `tempAvg` Int16 COMMENT 'Average temperature (tenths of a degrees C)',
  `tempMax` Int16 COMMENT 'Maximum temperature (tenths of degrees C)',
  `tempMin` Int16 COMMENT 'Minimum temperature (tenths of degrees C)',
  `precipitation` UInt16 COMMENT 'Precipitation (tenths of mm)',
  `snowfall` UInt16 COMMENT 'Snowfall (mm)',
  `snowDepth` UInt16 COMMENT 'Snow depth (mm)',
  `percentDailySun` UInt8 COMMENT 'Daily percent of possible sunshine (percent)',
  `averageWindSpeed` UInt16 COMMENT 'Average daily wind speed (tenths of meters per second)',
  `maxWindSpeed` UInt16 COMMENT 'Peak gust wind speed (tenths of meters per second)',
  `weatherType` String,
  `location` Point,
  `elevation` Int16,
  `name` String
) ENGINE = MergeTree() ORDER BY (station_id, date) AS 
SELECT * FROM default.noaa;
```

### Смотрим на размер столбцов во 2 версии таблицы
```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v2'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

### Смотрим общий размер таблицы версии 2
```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v2';
```

### Выполняем диагностический запрос
```sql
SELECT
    tempMax / 10 AS maxTemp,
    location,
    name,
    date
FROM default.noaa_codec_v2
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5
```

### Выполняем диагностический запрос
```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM default.noaa_codec_v2
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

# Применяем LowCardinality и Enum

### Создаем таблицы 3 версии
```sql
DROP TABLE IF EXISTS default.noaa_codec_v3;
CREATE TABLE default.noaa_codec_v3
(
 `station_id` LowCardinality(String) COMMENT 'Id of the station at which the measurement as taken',
 `date` Date32,
 `tempAvg` Int16 COMMENT 'Average temperature (tenths of a degrees C)',
 `tempMax` Int16 COMMENT 'Maximum temperature (tenths of degrees C)',
 `tempMin` Int16 COMMENT 'Minimum temperature (tenths of degrees C)',
 `precipitation` UInt16 COMMENT 'Precipitation (tenths of mm)',
 `snowfall` UInt16 COMMENT 'Snowfall (mm)',
 `snowDepth` UInt16 COMMENT 'Snow depth (mm)',
 `percentDailySun` UInt8 COMMENT 'Daily percent of possible sunshine (percent)',
 `averageWindSpeed` UInt16 COMMENT 'Average daily wind speed (tenths of meters per second)',
 `maxWindSpeed` UInt16 COMMENT 'Peak gust wind speed (tenths of meters per second)',
 `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
 `lat` Float32,
 `lon` Float32,
 `elevation` Int16,
 `name` LowCardinality(String)
) ENGINE = MergeTree() ORDER BY (station_id, date) AS 
SELECT 
	station_id, 
	date,
	tempAvg,
	tempMax,
	tempMin,
	precipitation,
	snowfall,
	snowDepth,
	percentDailySun,
	averageWindSpeed,
	maxWindSpeed,
	weatherType,
	location.2 as lat,
	location.1 as lon,
	elevation,
	name
FROM
	default.noaa;
```

### Смотрим на размер столбцов в 3 версии таблицы
```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v3'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

### Смотрим общий размер таблицы версии 3
```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v3'
```

### Выполняем диагностический запрос
```sql
SELECT
    tempMax / 10 AS maxTemp,
    lat,
    lon,
    name,
    date
FROM default.noaa_codec_v3
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5
```

### Выполняем диагностический запрос
```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM default.noaa_codec_v3
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

### 
```sql
SELECT * 
FROM system.columns
WHERE table = 'noaa_codec_v3';
```

# Применяем различные кодеки общего и специального назначения

### Создаем таблицы 4 версии
```sql
DROP TABLE IF EXISTS default.noaa_codec_v4;
CREATE TABLE default.noaa_codec_v4
(
    `station_id` LowCardinality(String),
    `date` Date32 CODEC(Delta, ZSTD),
    `tempAvg` Int16 CODEC(Delta, ZSTD),
    `tempMax` Int16 CODEC(Delta, ZSTD),
    `tempMin` Int16 CODEC(Delta, ZSTD),
    `precipitation` UInt16 CODEC(Delta, ZSTD),
    `snowfall` UInt16 CODEC(Delta, ZSTD),
    `snowDepth` UInt16 CODEC(Delta, ZSTD),
    `percentDailySun` UInt8 CODEC(Delta, ZSTD),
    `averageWindSpeed` UInt16 CODEC(Delta, ZSTD),
    `maxWindSpeed` UInt16 CODEC(Delta, ZSTD),
    `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
    `lat` Float32,
    `lon` Float32,
    `elevation` Int16,
    `name` LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (station_id, date) AS
SELECT
	station_id,
	date,
	tempAvg,
	tempMax,
	tempMin,
	precipitation,
	snowfall,
	snowDepth,
	percentDailySun,
	averageWindSpeed,
	maxWindSpeed,
	weatherType,
	location.2 as lat,
	location.1 as lon,
	elevation,
	name
FROM
	default.noaa;
```

### Смотрим на размер столбцов в 4 версии таблицы
```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v4'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

### Смотрим общий размер таблицы версии 4
```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    sum(data_uncompressed_bytes) / sum(data_compressed_bytes) AS compression_ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_v4'
```

### Смотрим информацию обо всех столбцах в 4 версии таблицы 
```sql
SELECT
    *
FROM system.columns
WHERE table = 'noaa_codec_v4'
```

### Выполняем диагностический запрос
```sql
SELECT
    tempMax / 10 AS maxTemp,
    lat,
    lon,
    name,
    date
FROM default.noaa_codec_v4
WHERE tempMax > 500
ORDER BY
    tempMax DESC,
    date ASC
LIMIT 5
```

### Выполняем диагностический запрос
```sql
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM default.noaa_codec_v3
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

### Смотрим, сколько нулевых значений в столбце precipitation
```sql
SELECT
    countIf(precipitation = 0) AS num_empty,
    countIf(precipitation > 0) AS num_non_zero,
    num_empty / (num_empty + num_non_zero) AS ratio
FROM default.noaa
```

### Смотрим, сколько нулевых значений в столбце snowDepth
```sql
SELECT
    countIf(snowDepth = 0) AS num_empty,
    countIf(snowDepth > 0) AS num_non_zero,
    num_empty / (num_empty + num_non_zero) AS ratio
FROM default.noaa
```

### Смотрим, сколько нулевых значений в столбце tempMax
```sql
SELECT
    countIf(tempMax = 0) AS num_empty,
    countIf(tempMax > 0) AS num_non_zero,
    num_empty / (num_empty + num_non_zero) AS ratio
FROM default.noaa
```

### Находим лучший вариант кодека для каждого столбца 
```sql
SELECT
    name,
    if(argMin(compression_codec, data_compressed_bytes) != '', argMin(compression_codec, data_compressed_bytes), 'DEFAULT') AS best_codec,
    formatReadableSize(min(data_compressed_bytes)) AS compressed_size
FROM system.columns
WHERE table LIKE 'noaa%'
GROUP BY name
```

# Финальный вариант

### Создаем таблицы финальной версии
```sql
DROP TABLE IF EXISTS default.noaa_codec_optimal;
CREATE TABLE default.noaa_codec_optimal
(
   `station_id` LowCardinality(String),
   `date` Date32 CODEC(DoubleDelta, ZSTD(1)),
   `tempAvg` Int16 CODEC(T64, ZSTD(1)),
   `tempMax` Int16 CODEC(T64, ZSTD(1)),
   `tempMin` Int16 CODEC(T64, ZSTD(1)) ,
   `precipitation` UInt16 CODEC(T64, ZSTD(1)) ,
   `snowfall` UInt16 CODEC(T64, ZSTD(1)) ,
   `snowDepth` UInt16 CODEC(ZSTD(1)),
   `percentDailySun` UInt8,
   `averageWindSpeed` UInt16 CODEC(T64, ZSTD(1)),
   `maxWindSpeed` UInt16 CODEC(T64, ZSTD(1)),
   `weatherType` Enum8('Normal' = 0, 'Fog' = 1, 'Heavy Fog' = 2, 'Thunder' = 3, 'Small Hail' = 4, 'Hail' = 5, 'Glaze' = 6, 'Dust/Ash' = 7, 'Smoke/Haze' = 8, 'Blowing/Drifting Snow' = 9, 'Tornado' = 10, 'High Winds' = 11, 'Blowing Spray' = 12, 'Mist' = 13, 'Drizzle' = 14, 'Freezing Drizzle' = 15, 'Rain' = 16, 'Freezing Rain' = 17, 'Snow' = 18, 'Unknown Precipitation' = 19, 'Ground Fog' = 21, 'Freezing Fog' = 22),
   `lat` Float32,
   `lon` Float32,
   `elevation` Int16,
   `name` LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (station_id, date) AS
SELECT
	station_id,
	date,
	tempAvg,
	tempMax,
	tempMin,
	precipitation,
	snowfall,
	snowDepth,
	percentDailySun,
	averageWindSpeed,
	maxWindSpeed,
	weatherType,
	location.2 as lat,
	location.1 as lon,
	elevation,
	name
FROM
	default.noaa;
```

### Смотрим на размер столбцов в финальной версии таблицы
```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_optimal'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

### Смотрим общий размер таблицы финальной версии 
```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio,
    sum(data_compressed_bytes) as compressed,
    sum(data_uncompressed_bytes) as uncompressed
FROM system.columns
WHERE table = 'noaa_codec_optimal'
```

### Выполняем диагностический запрос
```sql 
SELECT
    elevation_range,
    uniq(station_id) AS num_stations,
    max(tempMax) / 10 AS max_temp,
    min(tempMin) / 10 AS min_temp,
    sum(precipitation) AS total_precipitation,
    avg(percentDailySun) AS avg_percent_sunshine,
    max(maxWindSpeed) AS max_wind_speed,
    sum(snowfall) AS total_snowfall
FROM default.noaa_codec_optimal
WHERE (date > '1907-01-01') AND (station_id IN ('AG000060590', 'ASN00075063', 3))
GROUP BY floor(elevation, -2) AS elevation_range
ORDER BY elevation_range ASC
```

