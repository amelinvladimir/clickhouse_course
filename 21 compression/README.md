# 

# Скачиваем набор данных с [https://clickhouse.com/blog/real-world-data-noaa-climate-data](https://clickhouse.com/blog/real-world-data-noaa-climate-data)

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

### Смотрим на размер столбцов
```sql
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE table = 'noaa_codec_v1'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

### Смотрим общий размер
```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE table = 'noaa_codec_v1'
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
