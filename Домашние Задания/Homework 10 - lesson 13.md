```
Окружение: Oracle Linux 8.10, PostgreSQL 17.
```
# Работа с индексами.
Для начала подготавливаю новую таблицу для работы. 
```sql
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    year_published INT,
    content TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO books (title, author, year_published, content, is_active)
SELECT 
    'Book ' || i,
    'Author ' || (i % 100 + 1),
    1950 + (i % 70),
    'This is the content of book number ' || i || '. It is about some interesting topic.',
    CASE WHEN i % 10 > 0 THEN true ELSE false END
FROM generate_series(1, 10000) i;
```

## Создание индексов.
### Простой индекс (B-tree).
Для ускорения поиска по полю `year_published`. Без индекса поиск по году будет выполняться через полное сканирование таблицы, что медленно на больших данных.
```sql
CREATE INDEX idx_books_year ON books(year_published);

EXPLAIN ANALYZE
SELECT * FROM books WHERE year_published = 1985;

                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on books  (cost=5.39..186.72 rows=143 width=111) (actual time=0.171..0.509 rows=143 loops=1)
   Recheck Cond: (year_published = 1985)
   Heap Blocks: exact=143
   ->  Bitmap Index Scan on idx_books_year  (cost=0.00..5.36 rows=143 width=0) (actual time=0.112..0.112 rows=143 loops=1)
         Index Cond: (year_published = 1985)
 Planning Time: 1.313 ms
 Execution Time: 0.585 ms
(7 строк)
```
`Index Scan` означает, что PostgreSQL использовал наш индекс для поиска.  
Вместо перебора всех строк система быстро нашла нужные года через индекс. Это сильно ускоряет запросы с фильтрацией по индексированному полю.
### Индекс для полнотекстового поиска (GIN).
`LIKE` и `ILIKE` медленные на больших текстах. GIN-индекс ускорит поиск слов внутри текста.
```sql
ALTER TABLE books ADD COLUMN content_tsvector tsvector;
UPDATE books SET content_tsvector = to_tsvector('russian', content);

CREATE INDEX idx_books_content_gin ON books USING GIN(content_tsvector);

EXPLAIN ANALYZE
SELECT title, content 
FROM books 
WHERE content_tsvector @@ to_tsquery('russian', 'интересный');

                                                           QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on books  (cost=13.08..160.41 rows=50 width=85) (actual time=0.030..0.031 rows=0 loops=1)
   Recheck Cond: (content_tsvector @@ '''интересн'''::tsquery)
   ->  Bitmap Index Scan on idx_books_content_gin  (cost=0.00..13.07 rows=50 width=0) (actual time=0.018..0.019 rows=0 loops=1)
         Index Cond: (content_tsvector @@ '''интересн'''::tsquery)
 Planning Time: 30.421 ms
 Execution Time: 0.078 ms
(6 строк)
```
GIN-индекс используется для поиска лексем в тексте. `Bitmap Index Scan` означает, что индекс быстро нашел ссылки на строки, содержащие слово "интересный" .
```sql

```

```sql

```
