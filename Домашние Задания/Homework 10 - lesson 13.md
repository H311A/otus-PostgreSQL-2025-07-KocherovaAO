```
Окружение: Oracle Linux 8.10, PostgreSQL 17.
```

# Работа с индексами.
Для начала подготавливаю новую таблицу для работы. 

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    age INT,
    is_active BOOLEAN DEFAULT true,
    bio TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (username, email, age, is_active, bio)
SELECT 
    'user_' || i,
    'user_' || i || '@example.com',
    (random() * 70 + 15)::int,
    (random() > 0.1), -- 90% true
    'This is a bio for user ' || i
FROM generate_series(1, 100000) i;

VACUUM ANALYZE users;
```

## Создание индексов.
### Простой индекс (B-tree).

```sql
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user_50000@example.com';

                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=77) (actual time=0.796..0.799 rows=1 loops=1)
   Index Cond: ((email)::text = 'user_50000@example.com'::text)
 Planning Time: 1.033 ms
 Execution Time: 0.839 ms
(4 строки)
```
Создан классический B-tree индекс на поле `email`, чтобы ускорить запросы вида `WHERE email = ...`, избегая медленного полного сканирования таблицы. Это оптимальный выбор для поиска по точному совпадению или диапазону. Планировщик выбрал `Index Scan`, используя созданный индекс. Время выполнения менее 1 мс.

### Индекс для полнотекстового поиска (GIN).

```sql
ALTER TABLE users ADD COLUMN bio_tsvector TSVECTOR;

UPDATE users SET bio_tsvector = to_tsvector('english', bio);

CREATE INDEX idx_users_bio_gin ON users USING GIN(bio_tsvector);

EXPLAIN ANALYZE
SELECT username, bio FROM users
WHERE bio_tsvector @@ to_tsquery('english', 'user & 50000');

                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=30.06..37.94 rows=2 width=38) (actual time=0.469..0.472 rows=1 loops=1)
   Recheck Cond: (bio_tsvector @@ '''user'' & ''50000'''::tsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_users_bio_gin  (cost=0.00..30.06 rows=2 width=0) (actual time=0.325..0.326 rows=1 loops=1)
         Index Cond: (bio_tsvector @@ '''user'' & ''50000'''::tsquery)
 Planning Time: 0.437 ms
 Execution Time: 0.510 ms
(7 строк)
```
Добавлен столбец типа `TSVECTOR` для хранения подготовленных данных для поиска. Данные в этом столбце заполнены с помощью функции `to_tsvector`. Создан GIN-индекс на этом новом столбце, чтобы ускорить сложные поисковые запросы по текстовому полю `bio` с использованием операторов полнотекстового поиска `@@, to_tsquery`. GIN лучший выбор для полнотекстового поиска, так как он эффективно работает с составными типами данных, такими как лексемы. Планировщик выбрал `Bitmap Index Scan` по GIN-индексу, а затем `Bitmap Heap Scan` для точечного обращения к нужным строкам в таблице. 

### Частичный индекс.

```sql
CREATE INDEX idx_users_active_age ON users(age) WHERE is_active = true;

EXPLAIN ANALYZE
SELECT username, age FROM users
WHERE is_active = true AND age > 65;

                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=285.27..4044.52 rows=24900 width=14) (actual time=3.954..22.836 rows=25182 loops=1)
   Recheck Cond: ((age > 65) AND is_active)
   Heap Blocks: exact=2029
   ->  Bitmap Index Scan on idx_users_active_age  (cost=0.00..279.04 rows=24900 width=0) (actual time=3.495..3.495 rows=25182 loops=1)
         Index Cond: (age > 65)
 Planning Time: 0.225 ms
 Execution Time: 24.435 ms
(7 строк)
```

Создан индекс не по всей таблице, а только по ее части - по строкам, где `is_active = true`, чтобы уменьшить размер индекса и ускорить выполнение запросов, которые фильтруют именно по активным пользователям. `Bitmap Index Scan on idx_users_active_age` - планировщик использует созданный частичный индекс.

### Индекс на поле с функцией. 

```sql
CREATE INDEX idx_users_lower_username ON users(LOWER(username));

EXPLAIN ANALYZE
SELECT * FROM users WHERE LOWER(username) = LOWER('USER_75000');

                                                             QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=12.29..1372.19 rows=500 width=119) (actual time=0.124..0.126 rows=1 loops=1)
   Recheck Cond: (lower((username)::text) = 'user_75000'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_users_lower_username  (cost=0.00..12.17 rows=500 width=0) (actual time=0.109..0.109 rows=1 loops=1)
         Index Cond: (lower((username)::text) = 'user_75000'::text)
 Planning Time: 0.677 ms
 Execution Time: 0.159 ms
(7 строк)
```

Создан индекс не по самому столбцу `username`, а по результату вычисления функции `LOWER()` от этого столбца для ускорения запросов с регистронезависимым поиском, где используется функция `LOWER()` в условии `WHERE`. Обычный индекс по `username` не поможет для такого запроса. Планировщик выбрал `Bitmap Index Scan` по функциональному индексу.

### Индекс на несколько полей (составной).

```sql
CREATE INDEX idx_users_active_created ON users(is_active, created_at DESC);

EXPLAIN ANALYZE
SELECT username, created_at FROM users
WHERE is_active = true
ORDER BY created_at DESC
LIMIT 10;

                                                                   QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.29..1.59 rows=10 width=18) (actual time=0.102..0.114 rows=10 loops=1)
   ->  Index Scan using idx_users_active_created on users  (cost=0.29..11682.99 rows=89783 width=18) (actual time=0.100..0.109 rows=10 loops=1)
         Index Cond: (is_active = true)
 Planning Time: 0.845 ms
 Execution Time: 0.164 ms
(5 строк)
```

Создан индекс по двум полям `is_active` и `created_at` (с сортировкой по убыванию). Этот индекс подходит для запросов, которые одновременно фильтруют по `is_active` и сортируют по `created_at DESC`. Планировщик может выполнить весь запрос, пройдя по индексу в нужном порядке, без отдельной операции сортировки. Он также может использоваться для поиска только по `is_active` (левому префиксу индекса). Планировщик выбрал `Index Scan`, используя созданный составной индекс. Нет операции `Sort`, данные сразу извлекаются в нужном порядке благодаря индексу.

## Проблемы, которые возникли в процессе выполнения.

Частичный индекс:
```sql
EXPLAIN ANALYZE
SELECT username, age FROM users
WHERE is_active = true AND age > 30;
```
В запросе `WHERE is_active = true AND age > 30` планировщик не использовал индекс и выбрал полное сканирование таблицы.

```sql
                                                 QUERY PLAN
------------------------------------------------------------------------------------------------------------
 Seq Scan on users  (cost=0.00..4698.00 rows=69816 width=14) (actual time=2.051..41.568 rows=70044 loops=1)
   Filter: (is_active AND (age > 30))
   Rows Removed by Filter: 29956
 Planning Time: 1.297 ms
 Execution Time: 48.034 ms
(5 строк)
```
Вероятная причина:   
Соотношение строк, подходящих под условие, очень велико по сравнению с общим размером таблицы. Планировщик посчитал, что перебирать почти всю таблицу дешевле, чем использовать индекс. Это распространенная ситуация, когда индекс не используется, если он не селективен (не отсекает достаточно большую часть данных), поэтому я выполнила запрос для более селективного условия `WHERE is_active = true AND age > 65`.
