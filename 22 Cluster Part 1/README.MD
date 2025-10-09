# Разворот учебного кластера ClickHouse

## Замечания по выполнению последующих команд в зависимости от ОС

### Если вы работает на MacOS или Ubuntu, то просто откройте Теминал и выполните все команды в нем
### Если вы работает на Windows, то откройте Git Bash

### Открываем рабочую папку и в ней выполняем скрипт создания папок
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

### Выполняем в терминале скрипт создания конфигурационных файлов
```bash
for i in {01..04}; do
  mkdir -p fs/volumes/clickhouse-${i}/etc/clickhouse-server/config.d
  mkdir -p fs/volumes/clickhouse-${i}/etc/clickhouse-server/users.d
  touch fs/volumes/clickhouse-${i}/etc/clickhouse-server/config.d/config.xml
  touch fs/volumes/clickhouse-${i}/etc/clickhouse-server/users.d/users.xml
done
```

### Наполняем конфигурационные файлы config.d
```xml
<clickhouse replace="true">
    <logger>
        <level>debug</level>
        <log>/var/log/clickhouse-server/clickhouse-server.log</log>
        <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
        <size>1000M</size>
        <count>3</count>
    </logger>
    <display_name>cluster_2S_2R node 1</display_name>
    <listen_host>0.0.0.0</listen_host>
    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>
    <user_directories>
        <users_xml>
            <path>users.xml</path>
        </users_xml>
        <local_directory>
            <path>/var/lib/clickhouse/access/</path>
        </local_directory>
    </user_directories>
    <distributed_ddl>
        <path>/clickhouse/task_queue/ddl</path>
    </distributed_ddl>
    <remote_servers>
        <cluster_2S_2R>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>clickhouse-01</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>clickhouse-03</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>clickhouse-02</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>clickhouse-04</host>
                    <port>9000</port>
                </replica>
            </shard>
        </cluster_2S_2R>
    </remote_servers>
    <zookeeper>
        <node>
            <host>clickhouse-keeper-01</host>
            <port>9181</port>
        </node>
        <node>
            <host>clickhouse-keeper-02</host>
            <port>9181</port>
        </node>
        <node>
            <host>clickhouse-keeper-03</host>
            <port>9181</port>
        </node>
    </zookeeper>
    <macros>
        <shard>01</shard>
        <replica>01</replica>
    </macros>
</clickhouse>
```

### 
```bash
```


