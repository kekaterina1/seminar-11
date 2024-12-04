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
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.018..0.018 rows=0 loops=1)
  Recheck Cond: (category IS NULL)
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.015..0.015 rows=0 loops=1)
        Index Cond: (category IS NULL)
   Planning Time: 0.388 ms
   Execution Time: 0.056 ms
   
   *Объясните результат:*
  сработал индекс, так как нулевое значение является минимальным значением, то при брин индексе находится в самом начале(в первом блоке). такого значения в таблице нет, значит запрос выполнился очень быстро(быстро определил, что нужной строки нет).

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
   Bitmap Heap Scan on t_books  (cost=12.16..2372.65 rows=1 width=33) (actual time=26.455..26.456 rows=0 loops=1)
  Recheck Cond: ((category)::text = 'INDEX'::text)
  Rows Removed by Index Recheck: 150000
  Filter: ((author)::text = 'SYSTEM'::text)
  Heap Blocks: lossy=1225
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=75699 width=0) (actual time=0.133..0.134 rows=12250 loops=1)
        Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.252 ms
   Execution Time: 26.488 ms
   
   *Объясните результат (обратите внимание на bitmap scan):*
   вновь созданный индекс не используется в запросе, и процесс не ускоряется (без второго индкса время было таким же). это связано с тем, что записи в строках не отсортированы, то есть поиск по значению в каждом блоке потребует чтение всех записей. при неотсортированных записях brin индекс неэффективен.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=58.151..58.153 rows=6 loops=1)
  Sort Key: category
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=58.063..58.065 rows=6 loops=1)
        Group Key: category
        Batches: 1  Memory Usage: 24kB
        ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.007..16.125 rows=150000 loops=1)
   Planning Time: 0.136 ms
   Execution Time: 58.196 ms
   
   *Объясните результат:*
   так как необходимо вывести уникальные значения, необходимо сравнить все записи, брин ндекс не поможет в поиске уникальных значений, так как не дает информации о повторении записей

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=24.260..24.262 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=24.252..24.252 rows=0 loops=1)
        Filter: ((author)::text ~~ 'S%'::text)
        Rows Removed by Filter: 150000
   Planning Time: 0.239 ms
   Execution Time: 25.507 ms
   
   *Объясните результат:*
   записи в таблице не отсортированы, поэтому использование индекса неэффективно, используется последовательное сканирование. плюс регулярное выражение не даёт возможности использовать брин индекс (нужны префиксные индексы, тогда будет эффективно).

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
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=82.994..82.996 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=82.980..82.984 rows=1 loops=1)
        Filter: (lower((title)::text) ~~ 'o%'::text)
        Rows Removed by Filter: 149999
   Planning Time: 0.358 ms
   Execution Time: 83.034 ms
   
   *Объясните результат:*
   индексы вообще не использовались, тк поиск по регулярному выражению. с префиксными индексами было бы быстрее

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
   Bitmap Heap Scan on t_books  (cost=12.16..2372.65 rows=1 width=33) (actual time=1.622..1.623 rows=0 loops=1)
  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
  Rows Removed by Index Recheck: 8846
  Heap Blocks: lossy=73
  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=75699 width=0) (actual time=0.036..0.036 rows=730 loops=1)
        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 1.285 ms
   Execution Time: 1.658 ms
   
   *Объясните результат:*
   запрос выполнился намного быстрее в сравнении с ситуацией, когда были два отдельных индекса, так как в индексе хранится максимальное и минимальное значение сразу двух записей (индекс ускоряет поиск по обоим условиям)