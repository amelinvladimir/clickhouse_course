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

### 
```bash
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


