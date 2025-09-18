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
Для ускорения поиска по полю `year_published`. Без индекса поиск по году будет выполняться через полное сканирование таблицы, что медленно на больших данных.
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
`Index Scan` означает, что PostgreSQL использовал наш индекс для поиска.  
Вместо перебора всех строк система быстро нашла нужные года через индекс. Это сильно ускоряет запросы с фильтрацией по индексированному полю.
### Индекс для полнотекстового поиска (GIN).
`LIKE` и `ILIKE` медленные на больших текстах. GIN-индекс ускорит поиск слов внутри текста.
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
GIN-индекс используется для поиска лексем в тексте. `Bitmap Index Scan` означает, что индекс быстро нашел ссылки на строки, содержащие слово "интересный".
### Индекс на часть таблицы.
#### Индекс только для активных пользователей. 
Если часто запрашиваем только активные книги, нет смысла индексировать неактивные. Частичный индекс уменьшает размер индекса и ускоряет работу.
```sql
CREATE INDEX idx_users_active_age ON users(age) WHERE is_active = true;

EXPLAIN ANALYZE
SELECT username, age FROM users
WHERE is_active = true AND age > 30;

                                                 QUERY PLAN
------------------------------------------------------------------------------------------------------------
 Seq Scan on users  (cost=0.00..4698.00 rows=69816 width=14) (actual time=2.051..41.568 rows=70044 loops=1)
   Filter: (is_active AND (age > 30))
   Rows Removed by Filter: 29956
 Planning Time: 1.297 ms
 Execution Time: 48.034 ms
(5 строк)
```
#### Индекс
```sql

```
