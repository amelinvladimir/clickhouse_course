# Разворот учебного кластера ClickHouse

## Замечания по выполнению последующих команд в зависимости от ОС

### Если вы работает на MacOS или Ubuntu, то просто откройте Теминал и выполните все команды в нем
### Если вы работает на Windows, то откройте Git Bash

### Открываем рабочую папку и в ней выполняем скрипт
```bash
mkdir cluster_2S_2R
cd cluster_2S_2R

# Create clickhouse-keeper directories
for i in {01..03}; do
  mkdir -p fs/volumes/clickhouse-keeper-${i}/etc/clickhouse-keeper
done

# Create clickhouse-server directories
for i in {01..04}; do
  mkdir -p fs/volumes/clickhouse-${i}/etc/clickhouse-server
done
```

### Создаем в папке cluster_2S_2R файл docker-compose.yml
```yml
version: '3.8'
services:
  clickhouse-01:
    image: "clickhouse/clickhouse-server:latest"
    user: "101:101"
    container_name: clickhouse-01
    hostname: clickhouse-01
    volumes:
      - ${PWD}/fs/volumes/clickhouse-01/etc/clickhouse-server/config.d/config.xml:/etc/clickhouse-server/config.d/config.xml
      - ${PWD}/fs/volumes/clickhouse-01/etc/clickhouse-server/users.d/users.xml:/etc/clickhouse-server/users.d/users.xml
    ports:
      - "127.0.0.1:8123:8123"
      - "127.0.0.1:9000:9000"
    depends_on:
      - clickhouse-keeper-01
      - clickhouse-keeper-02
      - clickhouse-keeper-03
  clickhouse-02:
    image: "clickhouse/clickhouse-server:latest"
    user: "101:101"
    container_name: clickhouse-02
    hostname: clickhouse-02
    volumes:
      - ${PWD}/fs/volumes/clickhouse-02/etc/clickhouse-server/config.d/config.xml:/etc/clickhouse-server/config.d/config.xml
      - ${PWD}/fs/volumes/clickhouse-02/etc/clickhouse-server/users.d/users.xml:/etc/clickhouse-server/users.d/users.xml
    ports:
      - "127.0.0.1:8124:8123"
      - "127.0.0.1:9001:9000"
    depends_on:
      - clickhouse-keeper-01
      - clickhouse-keeper-02
      - clickhouse-keeper-03
  clickhouse-03:
    image: "clickhouse/clickhouse-server:latest"
    user: "101:101"
    container_name: clickhouse-03
    hostname: clickhouse-03
    volumes:
      - ${PWD}/fs/volumes/clickhouse-03/etc/clickhouse-server/config.d/config.xml:/etc/clickhouse-server/config.d/config.xml
      - ${PWD}/fs/volumes/clickhouse-03/etc/clickhouse-server/users.d/users.xml:/etc/clickhouse-server/users.d/users.xml
    ports:
      - "127.0.0.1:8125:8123"
      - "127.0.0.1:9002:9000"
    depends_on:
      - clickhouse-keeper-01
      - clickhouse-keeper-02
      - clickhouse-keeper-03
  clickhouse-04:
    image: "clickhouse/clickhouse-server:latest"
    user: "101:101"
    container_name: clickhouse-04
    hostname: clickhouse-04
    volumes:
      - ${PWD}/fs/volumes/clickhouse-04/etc/clickhouse-server/config.d/config.xml:/etc/clickhouse-server/config.d/config.xml
      - ${PWD}/fs/volumes/clickhouse-04/etc/clickhouse-server/users.d/users.xml:/etc/clickhouse-server/users.d/users.xml
    ports:
      - "127.0.0.1:8126:8123"
      - "127.0.0.1:9003:9000"
    depends_on:
      - clickhouse-keeper-01
      - clickhouse-keeper-02
      - clickhouse-keeper-03
  clickhouse-keeper-01:
    image: "clickhouse/clickhouse-keeper:latest-alpine"
    user: "101:101"
    container_name: clickhouse-keeper-01
    hostname: clickhouse-keeper-01
    volumes:
      - ${PWD}/fs/volumes/clickhouse-keeper-01/etc/clickhouse-keeper/keeper_config.xml:/etc/clickhouse-keeper/keeper_config.xml
    ports:
      - "127.0.0.1:9181:9181"
  clickhouse-keeper-02:
    image: "clickhouse/clickhouse-keeper:latest-alpine"
    user: "101:101"
    container_name: clickhouse-keeper-02
    hostname: clickhouse-keeper-02
    volumes:
      - ${PWD}/fs/volumes/clickhouse-keeper-02/etc/clickhouse-keeper/keeper_config.xml:/etc/clickhouse-keeper/keeper_config.xml
    ports:
      - "127.0.0.1:9182:9181"
  clickhouse-keeper-03:
    image: "clickhouse/clickhouse-keeper:latest-alpine"
    user: "101:101"
    container_name: clickhouse-keeper-03
    hostname: clickhouse-keeper-03
    volumes:
      - ${PWD}/fs/volumes/clickhouse-keeper-03/etc/clickhouse-keeper/keeper_config.xml:/etc/clickhouse-keeper/keeper_config.xml
    ports:
      - "127.0.0.1:9183:9181"
```

### 
```bash
```

### 
```bash
```

### 
```bash
```


