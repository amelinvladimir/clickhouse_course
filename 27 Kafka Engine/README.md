# Получение данных из Kafka

### docker-compose.yaml файл для запуска Kafka
```yml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9093:9093"  # для внутренних нужд, не обязательно
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CREATE_TOPICS: "example-topic:1:1"  # имя:партии:реплики

  create-topic:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - kafka
    command: >
      bash -c "
        kafka-topics --bootstrap-server kafka:9092 --create --topic example-topic --partitions 1 --replication-factor 1 || true
      "
```

### Запуск Kafka
```bash
docker-compose up -d  
```


### Установка библиотеки python для работы с Kafka
```bash
pip install kafka-python
```

### Отправка сообщений в Kafka
### send_message_to_kafka.py
```python
from kafka import KafkaProducer
import json
import time

# Настройки
BOOTSTRAP_SERVERS = ['localhost:9093']  # Порт PLAINTEXT_HOST из docker-compose
TOPIC_NAME = 'example-topic'

# Создание продюсера
producer = KafkaProducer(
    bootstrap_servers=BOOTSTRAP_SERVERS,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Отправка сообщений
for i in range(10):
    message = {'id': i, 'message': f'Hello Kafka from Python #{i}'}
    future = producer.send(TOPIC_NAME, value=message)
    result = future.get(timeout=10)  # ждём подтверждение
    print(f"Sent: {message} -> Partition {result.partition}, Offset {result.offset}")
    time.sleep(1)

producer.flush()
producer.close()


#{'id': 1, 'message': f'Hello Kafka from Python 1'}
```

### Чтение сообщений из Kafka
### get_message_from_kafka.py
```python
from kafka import KafkaConsumer
import json

# Настройки подключения
BOOTSTRAP_SERVERS = ['localhost:9093']  # Должен совпадать с PLAINTEXT_HOST из docker-compose
TOPIC_NAME = 'example-topic'

# Создание consumer'а
consumer = KafkaConsumer(
    TOPIC_NAME,
    bootstrap_servers=BOOTSTRAP_SERVERS,
    auto_offset_reset='earliest',  # читать с самого начала
    enable_auto_commit=False,      # не фиксировать офсеты автоматически (опционально)
    consumer_timeout_ms=10000,     # завершить, если новых сообщений нет 10 сек
    value_deserializer=lambda x: json.loads(x.decode('utf-8'))
)

print(f"Подключено к топику '{TOPIC_NAME}'. Ожидаю сообщения...\n")

try:
    for message in consumer:
        print(f"Partition: {message.partition}, Offset: {message.offset}")
        print(f"Key: {message.key}, Value: {message.value}\n")
except KeyboardInterrupt:
    print("\nОстановка потребителя...")
finally:
    consumer.close()
```

###
```bash

```
