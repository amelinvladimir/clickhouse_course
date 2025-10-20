# Конфигурирование сервера

### Меняем таблицу, в которую будут сохраняться логи по выполненным запросам
### /etc/clickhouse-server/config.d/query_log.xml
```xml
    <query_log>
        <!-- What table to insert data. If table is not exist, it will be created.
             When query log structure is changed after system update,
              then old table will be renamed and new table will be created automatically.
        -->
        <database>system</database>
        <table>query_log_new</table>
        <!--
            PARTITION BY expr: https://clickhouse.com/docs/en/table_engines/mergetree-family/custom_partitioning_key/
            Example:
                event_date
                toMonday(event_date)
                toYYYYMM(event_date)
                toStartOfHour(event_time)
        -->
        <partition_by>toYYYYMM(event_date)</partition_by>
    </query_log>
```

### Создаем роль в /etc/clickhouse-server/users.xml
```xml
    <roles>
        <learn_db_full>
            <grants>
                <query>GRANT ALL ON learn_db.*</query>
            </grants>
        </learn_db_full>
    </roles>
```

### Генерируем пароль и хэш от него
```bash
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDNf0r6vRl24Ix3tv2IgPmNPO2ATa2krvt80DdcTatLj john@example.com
```

### Создаем пользователя в /etc/clickhouse-server/users.xml
```xml
    <users>
        <amelinvd>
            <password_sha256_hex>58bb1ef6b1f95a54406279acaadbb595da3b8a87ece71c1f4204a9313abd3188</password_sha256_hex>
            <grants>
                <query>GRANT learn_db_full</query>
            </grants>
        </amelinvd>
    </users>
```
