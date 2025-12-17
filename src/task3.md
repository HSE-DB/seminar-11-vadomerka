## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=9.381..78.620 rows=499373 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=8334
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=8.538..8.539 rows=499373 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.227 ms
    Execution Time: 89.342 ms
    (7 rows)
    
    *Объясните результат:*
    Используется Bitmap Index Scan по индексу category, так как условие возвращает большое количество строк.
    Данные с категорией A физически распределены по всей таблице, поэтому Bitmap Heap Scan вынужден читать большое число блоков (Heap Blocks: exact=8334), что приводит к высокой стоимости и длительному времени выполнения.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on test_cluster  (cost=5600.73..20212.64 rows=502233 width=39) (actual time=7.233..38.280 rows=499373 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=4162
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5475.17 rows=502233 width=0) (actual time=6.834..6.835 rows=499373 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.203 ms
    Execution Time: 48.928 ms
    (7 rows)
    
    *Объясните результат:*
    После кластеризации строки с одинаковым значением category располагаются физически рядом.
    Это снижает количество читаемых блоков почти в два раза (Heap Blocks: exact=4162), улучшает локальность доступа к данным и существенно сокращает общее время выполнения запроса.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    Кластеризация уменьшила фрагментацию данных и повысила эффективность Bitmap Heap Scan.
    В результате время выполнения запроса сократилось почти вдвое, что демонстрирует преимущество кластеризации для выборок, возвращающих большие диапазоны строк по одному и тому же индексу.