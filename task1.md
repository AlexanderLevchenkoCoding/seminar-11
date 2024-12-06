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
   
|QUERY PLAN|
|----------|
|Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.007..0.008 rows=0 loops=1)|
|  Recheck Cond: (category IS NULL)|
|  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.006..0.006 rows=0 loops=1)|
|        Index Cond: (category IS NULL)|
|Planning Time: 0.166 ms|
|Execution Time: 0.058 ms|



   
   *Объясните результат:*
   В начале планировщик не хотел использовать 'Bitmap Heap Scan', но после проведённого речека уже стал использоватсья, о чём говорит последующее 'Bitmap Index Scan'

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
   |QUERY PLAN|
   |----------|
   |Bitmap Heap Scan on t_books  (cost=12.17..2406.32 rows=1 width=33) (actual time=9.458..9.459 rows=0 loops=1)|
   |  Recheck Cond: ((category)::text = 'INDEX'::text)|
   |  Rows Removed by Index Recheck: 150000|
   |  Filter: ((author)::text = 'SYSTEM'::text)|
   |  Heap Blocks: lossy=1224|
   |  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.17 rows=78010 width=0) (actual time=0.047..0.048 rows=12240 loops=1)|
   |        Index Cond: ((category)::text = 'INDEX'::text)|
   |Planning Time: 0.149 ms|
   |Execution Time: 9.498 ms|

   
   *Объясните результат (обратите внимание на bitmap scan):*
   Сначала используется "Bitmap HEap Scan", но после фильтрации оптимизатрор уже начинает использовать созданный индекс и проходится с использованием 'Bitmap Index Scan'

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   |QUERY PLAN|
   |----------|
   |Sort  (cost=3099.14..3099.15 rows=6 width=7) (actual time=22.555..22.557 rows=6 loops=1)|
   |  Sort Key: category|
   |  Sort Method: quicksort  Memory: 25kB|
   |  ->  HashAggregate  (cost=3099.00..3099.06 rows=6 width=7) (actual time=22.536..22.538 rows=6 loops=1)|
   |        Group Key: category|
   |        Batches: 1  Memory Usage: 24kB|
   |        ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.008..6.157 rows=150000 loops=1)|
   |Planning Time: 0.189 ms|
   |Execution Time: 22.657 ms|

   
   *Объясните результат:*
   ну зднесь у нас сортировка + дистинкт, т.е. индекс автоматически не используется. Ну а после этого уже просто обычное параллельное сканирование
9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   
   |QUERY PLAN|
   |----------|
   |Aggregate  (cost=3099.03..3099.05 rows=1 width=8) (actual time=6.842..6.843 rows=1 loops=1)|
   |  ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=14 width=0) (actual time=6.838..6.838 rows=0 loops=1)|
   |        Filter: ((author)::text ~~ 'S%'::text)|
   |        Rows Removed by Filter: 150000|
   |Planning Time: 2.243 ms|
   |Execution Time: 6.860 ms|

   
   *Объясните результат:*
   Опять же используется параллельное сканирование по причине наличия оператора like и префиксного поиска

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
   
   |QUERY PLAN|
   |----------|
   |Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=25.145..25.147 rows=1 loops=1)|
   |  ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=25.137..25.139 rows=1 loops=1)|
   |        Filter: (lower((title)::text) ~~ 'o%'::text)|
   |        Rows Removed by Filter: 149999|
   |Planning Time: 0.413 ms|
   |Execution Time: 25.168 ms|

   
   *Объясните результат:*
   аналогично предыдущему пункту

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
   
   |QUERY PLAN|
|----------|
|Bitmap Heap Scan on t_books  (cost=12.17..2406.32 rows=1 width=33) (actual time=0.613..0.613 rows=0 loops=1)|
|  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))|
|  Rows Removed by Index Recheck: 8793|
|  Heap Blocks: lossy=72|
|  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.17 rows=78010 width=0) (actual time=0.014..0.014 rows=720 loops=1)|
|        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))|
|Planning Time: 0.130 ms|
|Execution Time: 0.628 ms|

   
   *Объясните результат:*
   используется `Bitmap Index Scan`, потому что в 13 пункте был создан составной индекс по нужным колонкам