# 

# Скачиваем набор данных с [https://clickhouse.com/blog/real-world-data-noaa-climate-data](https://clickhouse.com/blog/real-world-data-noaa-climate-data)

### Внутри контейнера скачиваем файлы
```bash
for i in {1900..1930}; do wget https://noaa-ghcn-pds.s3.amazonaws.com/csv.gz/${i}.csv.gz; done
```
