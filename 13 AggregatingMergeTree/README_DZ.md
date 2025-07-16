```sql
-- =============================================
-- Задание 1: Создание агрегирующей таблицы
-- =============================================

-- Удаляем таблицу orders, если она уже существует
DROP TABLE IF EXISTS learn_db.orders;

-- Создаем таблицу orders с основными данными заказов
CREATE TABLE learn_db.orders (
    order_id UInt32,
    user_id UInt32,
    product_id UInt32,
    amount Decimal(18, 2),
    order_date Date
) ENGINE = MergeTree()
ORDER BY (product_id, order_date);

-- Удаляем агрегирующую таблицу, если она уже существует
DROP TABLE IF EXISTS learn_db.orders_stat_agg;

-- Создаем агрегирующую таблицу для хранения статистики по заказам
CREATE TABLE learn_db.orders_stat_agg (
    product_id UInt32,
    order_date Date,    
    uniq_orders AggregateFunction(uniq, UInt32),
    min_amount AggregateFunction(min, Decimal(18, 2)),
    max_amount AggregateFunction(max, Decimal(18, 2)),
    median_amount AggregateFunction(median, Decimal(18, 2)),
    quantile_amount AggregateFunction(quantile, Decimal(18, 2))
)
ENGINE = AggregatingMergeTree()
ORDER BY (product_id, order_date);

-- =============================================
-- Задание 2: Наполнение таблицы тестовыми данными
-- =============================================

-- Вставляем 10 миллионов тестовых записей в таблицу orders
INSERT INTO learn_db.orders
SELECT
    number + 10 AS order_id,
    round(randUniform(1, 1000)) AS user_id,
    round(randUniform(1, 1000)) AS product_id,
    randUniform(1, 5000) AS amount,
    date_add(DAY, rand() % 366, today() - INTERVAL 1 YEAR) AS order_date
FROM 
    numbers(10000000);

-- Проверяем количество записей в таблице
SELECT count() FROM learn_db.orders;

-- =============================================
-- Задание 3: Вставка агрегированных данных в таблицу orders_stat_agg
-- =============================================

-- Вставляем агрегированные данные в таблицу статистики
INSERT INTO learn_db.orders_stat_agg
(`product_id`, `order_date`, `uniq_orders`, `min_amount`, `max_amount`, `median_amount`, `quantile_amount`)
SELECT
    product_id,
    order_date,
    uniqState(order_id),
    minState(amount),
    maxState(amount),
    medianState(amount),
    quantileState(amount)
FROM
    learn_db.orders
GROUP BY
    product_id,
    order_date;

-- =============================================
-- Задание 4: Анализ агрегированных данных
-- =============================================

-- Выбираем агрегированные данные с применением функций merge
SELECT 
    product_id,
    order_date,
    minMerge(min_amount) AS min_amount,
    medianMerge(median_amount) AS median_amount,
    quantileMerge(0.9)(quantile_amount) AS quantile_90_amount
FROM
    learn_db.orders_stat_agg
GROUP BY
    product_id,
    order_date;

-- =============================================
-- Задание 5: Оптимизация и анализ производительности
-- =============================================

-- Запрос к агрегирующей таблице (оптимизированный вариант)
SELECT 
    product_id,
    order_date,
    medianMerge(median_amount) AS median_amount,
    quantileMerge(0.9)(quantile_amount) AS quantile_90_amount
FROM
    learn_db.orders_stat_agg
GROUP BY
    product_id,
    order_date;

-- Запрос к исходной таблице (для сравнения производительности)
SELECT 
    product_id,
    order_date,
    median(amount) AS median_amount,
    quantile(0.9)(amount) AS quantile_90_amount
FROM
    learn_db.orders
GROUP BY
    product_id,
    order_date;
```
