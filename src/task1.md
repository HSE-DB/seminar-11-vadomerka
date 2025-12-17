# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
    Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.064..0.065 rows=0 loops=1)
      Recheck Cond: (category IS NULL)
      ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.061..0.062 rows=0 loops=1)
            Index Cond: (category IS NULL)
   Planning Time: 0.309 ms
   Execution Time: 0.095 ms
   (6 rows)
   
   *Объясните результат:*
   BRIN-индекс используется через Bitmap Index Scan, так как он хорошо подходит для условий вида IS NULL при физической упорядоченности данных.
   Индекс определяет диапазоны страниц, где могут находиться строки с NULL, после чего Bitmap Heap Scan перепроверяет строки на уровне таблицы, что и отражено в Recheck Cond.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
    Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=8.052..8.053 rows=0 loops=1)
      Recheck Cond: ((category)::text = 'INDEX'::text)
      Rows Removed by Index Recheck: 150000
      Filter: ((author)::text = 'SYSTEM'::text)
      Heap Blocks: lossy=1225
      ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.085..0.085 rows=12250 loops=1)
            Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.237 ms
   Execution Time: 8.076 ms
   (9 rows)
   
   *Объясните результат (обратите внимание на bitmap scan):*
   BRIN-индекс по category возвращает очень широкий диапазон блоков, так как значения category распределены неравномерно и плохо коррелируют с физическим порядком строк.
   В результате bitmap становится lossy (поблочный, а не построчный), что приводит к массовой перепроверке строк (Rows Removed by Index Recheck) и дополнительной фильтрации по author, не покрытому индексом.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
    Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=18.572..18.573 rows=6 loops=1)
      Sort Key: category
      Sort Method: quicksort  Memory: 25kB
      ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=18.557..18.558 rows=6 loops=1)
            Group Key: category
            Batches: 1  Memory Usage: 24kB
            ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.005..5.199 rows=150000 loops=1)
   Planning Time: 0.067 ms
   Execution Time: 18.619 ms
   (9 rows)
   
   *Объясните результат:*
   Запрос требует получить все уникальные значения category, поэтому PostgreSQL выполняет полный Seq Scan.
   BRIN-индекс здесь не используется, так как агрегирование (DISTINCT) и сортировка по всем данным выгоднее выполнить на основе полного сканирования с HashAggregate, чем через индекс с низкой селективностью.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
    Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=6.090..6.091 rows=1 loops=1)
      ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=6.087..6.088 rows=0 loops=1)
            Filter: ((author)::text ~~ 'S%'::text)
            Rows Removed by Filter: 150000
   Planning Time: 0.246 ms
   Execution Time: 6.109 ms
   (6 rows)
   
   *Объясните результат:*
   Условие author LIKE 'S%' не использует BRIN-индекс, поскольку такой индекс неэффективен для префиксного поиска по строкам.
   Оптимизатор выбирает последовательное сканирование, так как требуется проверить каждую строку, а ожидаемое число совпадений крайне мало.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
    Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=24.752..24.754 rows=1 loops=1)
      ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=24.746..24.748 rows=1 loops=1)
            Filter: (lower((title)::text) ~~ 'o%'::text)
            Rows Removed by Filter: 149999
   Planning Time: 0.280 ms
   Execution Time: 24.778 ms
   (6 rows)
   
   *Объясните результат:*
   Несмотря на наличие функционального индекса по LOWER(title), PostgreSQL использует Seq Scan, так как условие LIKE 'o%' возвращает слишком мало строк и индекс не даёт выигрыша по стоимости.
   Дополнительно, функция LOWER() применяется ко всем строкам, что делает последовательное сканирование более предсказуемым по времени.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
    Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=0.609..0.610 rows=0 loops=1)
      Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
      Rows Removed by Index Recheck: 8828
      Heap Blocks: lossy=73
      ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.020..0.021 rows=730 loops=1)
            Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.170 ms
   Execution Time: 0.650 ms
   (8 rows)

   *Объясните результат:*
   Составной BRIN-индекс по (category, author) значительно уменьшает число подходящих блоков.
   Bitmap становится менее lossy, резко сокращается количество перепроверяемых строк и блоков, что приводит к заметному снижению времени выполнения запроса по сравнению с одиночным BRIN-индексом.