# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
    ```sql
    QUERY PLAN
    Seq Scan on t_books  (cost=0.00..3099.00 rows=1 width=33) (actual time=23.189..23.190 rows=1 loops=1)
    Filter: ((title)::text = 'Oracle Core'::text)
    Rows Removed by Filter: 149999
    Planning Time: 3.107 ms
    Execution Time: 23.936 ms
   ```
   
   *Объясните результат:*
   Код без индексов имеет маленький planning time, но относительно большой execution time, потому что запрос выполняется напрямую, обрабаывая все строчки. 

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   ```sql
   CREATE INDEX
   Query returned successfully in 202 msec.
   ```

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   ```sql
   schemaname	tablename	indexname	                  indexdef
   public	         t_books	  t_books_id_pk	         CREATE UNIQUE INDEX t_books_id_pk ON public.t_books USING btree (book_id)
   public	         t_books	  t_books_title_idx	      CREATE INDEX t_books_title_idx ON public.t_books USING btree (title)
   public	         t_books	  t_books_active_idx       CREATE INDEX t_books_active_idx ON public.t_books USING btree (is_active)
   ```
   
   *Объясните результат:*
   ```sql
   Создали btree индексы для ПК (unique) и другого поля. Таблица отражает имя индекса, его тип и привязку.
   ```

6. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ```sql
   ANALYZE

   Query returned successfully in 278 msec.
   ```

8. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```sql
   QUERY PLAN
   Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.812..0.813 rows=1 loops=1)
   Index Cond: ((title)::text = 'Oracle Core'::text)
   Planning Time: 11.263 ms
   Execution Time: 1.434 ms
   ```
   
   *Объясните результат:*
   В сравнении с аналогичным запросом без индекса: план показывает используемый индекс для поиска, в результате время выполнения запроса сократилось почти в 2 раза, при этом по      распределению теперь больше времени уходит на подготовку, чем на сам поиск.

10. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
   QUERY PLAN
   Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.042..0.044 rows=1 loops=1)
   Index Cond: (book_id = 18)
   Planning Time: 0.106 ms
   Execution Time: 0.062 ms
   ```
   
   *Объясните результат:*
   Вопрос выполнен менне чем за 1 милисекунду, поиск происходил по ПК, за счет того, что ключ уникален, поиск осуществляется быстро, путем определения диапазонов в дочерних узлах.

11. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ```sql
   QUERY PLAN
   Seq Scan on t_books  (cost=0.00..2724.00 rows=75555 width=33) (actual time=0.084..26.782 rows=75375 loops=1)
   Filter: is_active
   Rows Removed by Filter: 74625
   Planning Time: 0.264 ms
   Execution Time: 30.059 ms
   ```

   *Объясните результат:*
   Поиск активных книг осуществляется по фильтру без индексации -- такой вариант убирает строки, не подходящие под условие, но проходит по каждой строчке -- в результате выполнения запроса большая часть времени уходит на выполнение непосредственно.

11. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   ```sql
   "total_rows"	"unique_titles"	"unique_categories"	"unique_authors"
   150000	         150000	          6	                 1003
   ```

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    ```sql
    DROP INDEX
    Query returned successfully in 125 msec.
    ```

12. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    ```sql
    CREATE INDEX t_books_a_idx ON t_books(title, category); --a
    CREATE INDEX t_books_b_idx ON t_books(title); --b
    CREATE INDEX t_books_c_idx ON t_books(category,author); --с
    CREATE INDEX t_books_d_idx ON t_books(book_id, author); --d
    ```
    
    *Объясните ваше решение:*
    Часть созданных индексов составные, что позволяет избежать создания нескольких индексов для фильтрации по нескольким столбцам а также увеличить производительность при поиске.

14. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    ```sql
    QUERY PLAN --a
    Index Scan using t_books_b_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.684..0.685 rows=0 loops=1)
    Index Cond: ((title)::text = 'Book_13'::text)
    Filter: ((category)::text = 'Art'::text)
    Rows Removed by Filter: 1
    Planning Time: 3.651 ms
    Execution Time: 1.041 ms
    ```

     ```sql
     QUERY PLAN --b
     Index Scan using t_books_b_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=1.020..1.021 rows=1 loops=1)
     Index Cond: ((title)::text = 'Book_13'::text)
     Planning Time: 3.582 ms
     Execution Time: 1.782 ms
    ```

    ```sql
    QUERY PLAN --c
    Bitmap Heap Scan on t_books  (cost=4.60..110.96 rows=30 width=33) (actual time=0.543..0.544 rows=0 loops=1)
    Recheck Cond: (((category)::text = 'Art'::text) AND ((author)::text = '$2'::text))
    ->  Bitmap Index Scan on t_books_c_idx  (cost=0.00..4.59 rows=30 width=0) (actual time=0.542..0.542 rows=0 loops=1)
    Index Cond: (((category)::text = 'Art'::text) AND ((author)::text = '$2'::text))
    Planning Time: 0.326 ms
    Execution Time: 1.296 ms
    ```

     ```sql
     QUERY PLAN --d
     Index Scan using t_books_d_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=2.026..2.027 rows=0 loops=1)
     Index Cond: ((book_id = 2) AND ((author)::text = '$1'::text))
     Planning Time: 0.460 ms
     Execution Time: 2.118 ms
    ```
    
    *Объясните результаты:*
    Индексы протестированы с помощью команды EXPLAIN ANALYZE. Результаты неоднозначны -- часть запросов даже при наличии индекса имеют время выполнения больше времени планирования. Для каждого запроса был использован индекс, специально созданный под фильтр (a/b/c/d idx). 

16. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=33) (actual time=32.482..32.483 rows=0 loops=1)
    Filter: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Filter: 150000
    Planning Time: 3.950 ms
    Execution Time: 32.863 ms
    ```
    
    *Объясните результат:*
    Регистронезависимый поиск осуществляется за счет оператора ILIKE. Так как в данном запросе поиск подходящих значений происходил по фразе, не существующей в таблице, в результате запроса были отфильтрованы все строки таблицы.

18. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    ```sql
    CREATE INDEX

    Query returned successfully in 176 msec.
    ```

19. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Bitmap Heap Scan on t_books  (cost=20.11..1149.04 rows=750 width=33) (actual time=0.624..0.625 rows=0 loops=1)
    Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)
    ->  Bitmap Index Scan on t_books_up_title_idx  (cost=0.00..19.92 rows=750 width=0) (actual time=0.622..0.623 rows=0 loops=1)
    Index Cond: ((upper((title)::text) >= 'RELATIONAL'::text) AND (upper((title)::text) < 'RELATIONAM'::text))
    Planning Time: 1.036 ms
    Execution Time: 0.992 ms"
    
    *Объясните результат:*
    Результат выполнения тот же самый, а время сократилось значительно. Если индекс используется, это означает, что база данных эффективно использует индекс для поиска значений в    столбце title, приведенных к верхнему регистру. Это ускоряет поиск, так как индекс позволяет обходиться без полного сканирования всех строк.

21. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=33) (actual time=37.898..37.899 rows=1 loops=1)
    Filter: ((title)::text ~~* '%Core%'::text)
    Rows Removed by Filter: 149999
    Planning Time: 0.247 ms
    Execution Time: 38.957 ms
    ```
    
    *Объясните результат:*
    Регистронезависимый поиск ILIKE может привести к полному сканированию таблицы, если индекса нет. Это неэффективно.

23. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ```sql
    ERROR:  cannot drop index t_books_id_pk because constraint t_books_id_pk on table t_books requires it
    HINT:  You can drop constraint t_books_id_pk on table t_books instead.
    CONTEXT:  SQL statement "DROP INDEX t_books_id_pk"
    PL/pgSQL function inline_code_block line 9 at EXECUTE

    SQL state: 2BP01
    ```
    
    *Объясните результат:*
    Индекс, связанный с первичным ключом, не может быть удален напрямую, потому что он используется для обеспечения уникальности значений в столбце. Чтобы его удалить, надо убрать из запроса индекс первичного ключа.

24. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    ```sql
    QUERY PLAN 1
    Bitmap Heap Scan on t_books  (cost=18.23..1147.17 rows=750 width=33) (actual time=0.471..0.471 rows=0 loops=1)
    Filter: (reverse((title)::text) ~~ 'eroC'::text)
     ->  Bitmap Index Scan on t_books_rev_title_idx  (cost=0.00..18.05 rows=750 width=0) (actual time=0.468..0.468 rows=0 loops=1)
    Index Cond: (reverse((title)::text) = 'eroC'::text)
    Planning Time: 10.610 ms
    Execution Time: 1.138 ms

    QUERY PLAN 2
    Bitmap Heap Scan on t_books  (cost=30.25..85.46 rows=15 width=33) (actual time=0.325..0.326 rows=1 loops=1)
    Recheck Cond: ((title)::text ~~ '%Core'::text)
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..30.25 rows=15 width=0) (actual time=0.298..0.298 rows=1 loops=1)
    Index Cond: ((title)::text ~~ '%Core'::text)
    Planning Time: 6.183 ms
    Execution Time: 1.480 ms
    ```
    
    *Объясните результаты:*
    Вариант с оптимизацией за счет триггеров проходит быстрее в данном случае, так как он не ищет точного совпадения.

26. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Index Scan using t_books_b_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.739..0.741 rows=1 loops=1)
    Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 5.863 ms
    Execution Time: 1.813 ms
    ```
    
    *Объясните результат:*
    Поиск по точному совпадению проходит быстрее, чем в предыдущих случаях (что на самом деле не казалось очевидным..). В случае суффиксного поиска база данных должна сначала преобразовать строку, поэтому planning time больше, потом уже выполнить поиск. Время планирования запроса в точном поиске сокращается за счет существующего индекса.

28. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.366..0.367 rows=0 loops=1)
    Recheck Cond: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Index Recheck: 1
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.323..0.324 rows=1 loops=1)
    Index Cond: ((title)::text ~~* 'Relational%'::text)
    Planning Time: 4.806 ms
    Execution Time: 1.474 ms
    ```
   

30. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```sql
     CREATE INDEX t_books_idx ON t_books(book_id DESC)
    ```
    
    *План выполнения:*
    ```sql
    QUERY PLAN
    Index Scan using t_books_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.538..0.539 rows=1 loops=1)
    Index Cond: (book_id = 10)
    Planning Time: 0.846 ms
    Execution Time: 0.577 ms
    ```
    
    *Объясните результат:*
    Индекс с обратной сортировкой может работать быстрее, так как создает дополнительное условие на нужный порядок расставления. Скорее всего, сортировка по убыванию поможет, если id нужной книги будет ближе к концу
