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
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.049..0.050 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.039..0.040 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 3.265 ms
   Execution Time: 0.219 ms
   ```

   *Объясните результат:*
   PostgreSQL использует Bitmap Index Scan на BRIN индексе для поиска NULL значений. BRIN индекс эффективно определил, что в таблице нет записей с NULL в category (rows=0). Bitmap Heap Scan используется для проверки условия после сканирования индекса. Время выполнения очень быстрое (0.219 ms).

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
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2393.32 rows=1 width=33) (actual time=10.759..10.759 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1225
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=77077 width=0) (actual time=0.097..0.097 rows=12250 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.653 ms
   Execution Time: 10.854 ms
   ```

   *Объясните результат (обратите внимание на bitmap scan):*
   Используется Bitmap Heap Scan с индексом только по category (t_books_brin_cat_idx). BRIN индекс не очень точен - он просканировал все 1225 блоков (lossy blocks), что привело к проверке всех 150000 строк. Фильтр по author применяется уже после сканирования heap. Это показывает, что BRIN индексы лучше работают с физически упорядоченными данными и диапазонными запросами.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category
   FROM t_books
   ORDER BY category;
   ```

   *План выполнения:*
   ```
   Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=33.226..33.227 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=33.126..33.128 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.010..12.129 rows=150000 loops=1)
   Planning Time: 1.196 ms
   Execution Time: 33.533 ms
   ```

   *Объясните результат:*
   PostgreSQL выполняет полное сканирование таблицы (Seq Scan), затем использует HashAggregate для получения уникальных значений и Sort для сортировки. BRIN индекс не используется, так как для DISTINCT требуется сканирование всех значений, а BRIN индекс хранит только минимум/максимум для блоков страниц.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*)
   FROM t_books
   WHERE author LIKE 'S%';
   ```

   *План выполнения:*
   ```
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=9.628..9.629 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=9.623..9.623 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 1.843 ms
   Execution Time: 9.712 ms
   ```

   *Объясните результат:*
   PostgreSQL использует Seq Scan вместо BRIN индекса. Это происходит потому, что BRIN индекс не эффективен для операций LIKE с шаблонами, так как хранит только минимальные/максимальные значения для диапазонов, а не поддерживает поиск по префиксу. Найдено 0 строк, так как все авторы имеют формат 'Author_XXX'.

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
   ```
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=31.432..31.432 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=31.421..31.422 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 0.836 ms
   Execution Time: 31.549 ms
   ```

   *Объясните результат:*
   Несмотря на наличие индекса t_books_lower_title_idx, PostgreSQL использует Seq Scan. Это потому, что обычный B-tree индекс не оптимизирован для LIKE с шаблонами. Для эффективного поиска по префиксу нужен был бы индекс с text_pattern_ops, либо GIN/GiST индекс для полнотекстового поиска. Найдена 1 строка - 'Oracle Core'.

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
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2393.32 rows=1 width=33) (actual time=0.882..0.883 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8843
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=77077 width=0) (actual time=0.067..0.067 rows=730 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.673 ms
   Execution Time: 0.979 ms
   ```

   *Объясните результат:*
   Теперь используется составной BRIN индекс t_books_brin_cat_auth_idx, который учитывает оба условия. По сравнению с шагом 7 (10.854 ms), производительность улучшилась до 0.979 ms - время выполнения сократилось более чем в 10 раз. Составной индекс позволил эффективнее отфильтровать диапазоны: обработано только 73 блока вместо 1225, что значительно сократило количество проверенных строк (8843 против 150000).