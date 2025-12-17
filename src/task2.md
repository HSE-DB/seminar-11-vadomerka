## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
     Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.082..0.083 rows=1 loops=1)
     Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
     Heap Blocks: exact=1
     ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.074..0.074 rows=1 loops=1)
          Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
     Planning Time: 1.635 ms
     Execution Time: 0.117 ms
     (7 rows)
    
    *Объясните результат:*
     GIN-индекс по to_tsvector(title) используется через Bitmap Index Scan, так как полнотекстовый поиск может возвращать несколько совпадений.
     Индекс быстро находит строки, содержащие лексему expert, после чего Bitmap Heap Scan извлекает реальные строки таблицы и выполняет финальную проверку (Recheck Cond), что обеспечивает очень малое время выполнения.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.012..0.012 rows=1 loops=1)
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.179 ms
     Execution Time: 0.027 ms
     (4 rows)
     
     *Объясните результат:*
     Поиск по первичному ключу в обычной таблице выполняется через Index Scan по B-tree индексу.
     Так как ключ уникален, оптимизатор ожидает ровно одну строку, и доступ к данным происходит напрямую по индексу с минимальными затратами времени.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.088..0.089 rows=1 loops=1)
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.200 ms
     Execution Time: 0.109 ms
     (4 rows)

     
     *Объясните результат:*
     План выполнения идентичен обычной таблице — используется Index Scan по первичному ключу.
     Кластеризация не даёт заметного выигрыша для точечного поиска одной строки, так как и в обычной таблице индекс уже обеспечивает прямой доступ к нужному блоку данных.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.046..0.047 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.286 ms
     Execution Time: 0.062 ms
     (4 rows)
     
     *Объясните результат:*
     Поиск по item_value выполняется через вторичный индекс t_lookup_value_idx.
     Так как значение отсутствует в таблице, индекс всё равно используется для быстрой проверки, что приводит к мгновенному завершению запроса без чтения большого объёма данных.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.058..0.059 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.324 ms
     Execution Time: 0.079 ms
     (4 rows)
     
     *Объясните результат:*
     Для кластеризованной таблицы используется аналогичный Index Scan по индексу значения.
     Поскольку поиск не возвращает строк, физический порядок данных (кластеризация) не оказывает влияния на план и время выполнения запроса.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Производительность поиска по значению в обычной и кластеризованной таблицах практически одинаковая.
     Это объясняется тем, что запрос использует индекс и не извлекает данные массово: преимущества кластеризации проявляются только при чтении диапазонов строк, а не при точечном или пустом поиске.