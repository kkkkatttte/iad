# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
   ```sql
   Query returned successfully in 260 msec.
   ```

5. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
   QUERY PLAN
   Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.023..6.874 rows=1 loops=1)
   Filter: (book_id = 18)
   Rows Removed by Filter: 49998
   Planning Time: 0.919 ms
   Execution Time: 6.911 ms
   ```
   
   *Объясните результат:*
   Запрос работает медленнее, чем поиск по исходной таблице из-за отсутствия индексов конкретно в этой таблице, идет обработка всех строк

7. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```sql
   QUERY PLAN
   Append  (cost=0.00..3101.01 rows=3 width=33) (actual time=8.429..24.075 rows=1 loops=1)
   ->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=8.428..8.429 rows=1 loops=1)
   Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
   Rows Removed by Filter: 49998
   ->  Seq Scan on t_books_part_2  (cost=0.00..1034.00 rows=1 width=33) (actual time=8.756..8.756 rows=0 loops=1)
   Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
   Rows Removed by Filter: 50000
   ->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=6.885..6.885 rows=0 loops=1)
   Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
   Rows Removed by Filter: 50001
   Planning Time: 0.227 ms
   Execution Time: 24.133 ms
   ```
   
   *Объясните результат:*
   Выполнение данного запроса длиннее, чем поиск по индексу: партиция для поиска не указана, поэтому поиск происходит по всем строкам, смысла от партицирования в данном случае нет.

9. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   ```sql
   Query returned successfully in 147 msec.
   ```

11. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```sql
    QUERY PLAN
    Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.369..1.016 rows=1 loops=1)
    ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.368..0.369 rows=1 loops=1)
    Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.326..0.326 rows=0 loops=1)
    Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.319..0.319 rows=0 loops=1)
    Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 1.743 ms
    Execution Time: 1.068 ms
    ```
   
   *Объясните результат:*
   Указания на партицию все еще нет, но запрос работает значительно быстрее из-за использования индекса.

11. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   ```sql
    Query returned successfully in 106 msec.
```

11. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
```
   
   *Результат:*
   ```sql
   Query returned successfully in 100 msec.
```

11. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.343..0.960 rows=1 loops=1)
    ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.342..0.343 rows=1 loops=1)
    Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.316..0.316 rows=0 loops=1)
    Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.298..0.298 rows=0 loops=1)
    Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 3.048 ms
    Execution Time: 1.016 ms
    ```

    
    *Объясните результат:*
    Увеличился planning time, вероятно из-за индексирования отдельных партиций, хотя время выполнения не сильно уменьшилось, так как партиции были проиндексированы при индексации всей таблицы

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 69 msec.
    ```

14. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 120 msec.
    ```

16. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.298..0.299 rows=1 loops=1)
    Index Cond: (book_id = 11011)
    Planning Time: 1.437 ms
    Execution Time: 0.363 ms
    ```
    
    *Объясните результат:*
    Выполнение кода быстрое за счет индекса по ПК, поиск пошел сразу по партиции, нужная партиция определилась самостоятельно (указана в запросе исходная таблица) -- за счет уникальности значения поиск происходил по нужному дочернему узлу.

18. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 125 msec.
    ```

20. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    ```sql
    Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.298..0.299 rows=1 loops=1)
    Index Cond: (book_id = 11011)
    Planning Time: 0.421 ms
    Execution Time: 32.153 ms
    ```
    
    *Объясните результат:*
    Наиболее эффективный метод SEQ Scan был отключен, поэтому поиск просходил с помощью Index Scan. Время выполнения сильно увеличилось, данный метод не является эффективным для запроса.

22. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 207 msec.
    ```

23. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    HashAggregate  (cost=3474.00..3484.00 rows=1000 width=42) (actual time=43.764..43.998 rows=1003 loops=1)
    Group Key: author
    Batches: 1  Memory Usage: 193kB
    ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=21) (actual time=0.011..15.048 rows=150000 loops=1)
    Planning Time: 2.110 ms
    Execution Time: 44.072 ms
    ```
    
    *Объясните результат:*
    Прошло сканирование по всем строкам таблицы (15000), затем группировка. Агрегация начинает работать после группировку

25. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Limit  (cost=0.29..31.71 rows=10 width=10) (actual time=0.555..0.820 rows=10 loops=1)
    ->  Unique  (cost=0.29..3141.30 rows=1000 width=10) (actual time=0.554..0.817 rows=10 loops=1)
    ->  Index Only Scan using t_books_author on t_books  (cost=0.29..2766.30 rows=150000 width=10) (actual time=0.552..0.701 rows=1330 loops=1)
    Heap Fetches: 0
    Planning Time: 2.360 ms
    Execution Time: 0.832 ms
    ```
    
    *Объясните результат:*
    Время работы уменьшается за счет сканирования только индексированных строк, при этом limit на работу по идее не влияет: скан все еще проходит по всем строкам, несмотря на то, что нужное количество найдено.

27. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..13.28 rows=15 width=21) (actual time=0.017..0.018 rows=1 loops=1)
    Index Cond: ((author >= 'T'::text) AND (author < 'U'::text))
    Filter: ((author)::text ~~ 'T%'::text)
    Heap Fetches: 0
    Planning Time: 3.303 ms
    Execution Time: 0.052 ms
    ```
    
    *Объясните результат:*
    Идет скан по двум столбцам, что уменьшает время работы

29. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 166 msec.
    ```

31. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 276 msec.
    ```

33. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.15 rows=1 width=21) (actual time=0.193..0.195 rows=1 loops=1)
    Index Cond: (category IS NULL)
    Planning Time: 0.470 ms
    Execution Time: 0.227 ms
    ```
    
    *Объясните результат:*
    По индексу сразу нашлась 1 строка, удовлетворяющая условию

35. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 125 msec.
    ```

37. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.98 rows=1 width=21) (actual time=0.023..0.024 rows=1 loops=1)
    Planning Time: 0.786 ms
    Execution Time: 0.034 ms
    ```
    
    *Объясните результат:*
    По индексу сразу нашлась 1 строка, удовлетворяющая условию

39. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
    ```sql
    Query returned successfully in 102 msec.

    Query returned successfully in 135 msec.

    ERROR:  Key (title)=(Unique Science Book) already exists.duplicate key value violates unique constraint "t_books_selective_unique_idx" 

    ERROR:  duplicate key value violates unique constraint "t_books_selective_unique_idx"
    SQL state: 23505
    Detail: Key (title)=(Unique Science Book) already exists.

    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Объясните результат:*
    Индекс не создан, так как не соблюдается уникальность сочетания категории и названия, при этом можно создать индексы с разным названием в одной категории или с одним, но в разных категориях.
