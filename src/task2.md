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
    ```
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.069..0.070 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.056..0.056 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 3.506 ms
    Execution Time: 0.207 ms
    ```

    *Объясните результат:*
    PostgreSQL использует GIN индекс для полнотекстового поиска. GIN индекс эффективно находит документы, содержащие слово 'expert'. Найдена 1 запись - 'Expert PostgreSQL Architecture'. Использование GIN индекса делает полнотекстовый поиск очень быстрым (0.207 ms), обработан только 1 блок данных.

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
     ```
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.214..0.214 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 1.436 ms
     Execution Time: 0.480 ms
     ```

     *Объясните результат:*
     PostgreSQL использует Index Scan по первичному ключу t_lookup_pk. B-tree индекс первичного ключа эффективно находит нужную запись за одну операцию. Время выполнения 0.480 ms.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```

     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=1.448..1.451 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 2.536 ms
     Execution Time: 1.495 ms
     ```

     *Объясните результат:*
     Используется Index Scan по первичному ключу. Интересно, что время выполнения (1.495 ms) немного выше, чем у обычной таблицы (0.480 ms). Это может быть связано с кэшированием или особенностями физического расположения данных в конкретный момент выполнения запроса. Для поиска по первичному ключу кластеризация не дает преимущества, так как B-tree индекс и так эффективен.

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
     ```
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.171..0.171 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 2.032 ms
     Execution Time: 0.300 ms
     ```

     *Объясните результат:*
     Используется Index Scan по индексу t_lookup_value_idx. Запрос выполнен быстро (0.300 ms), но запись не найдена (rows=0), так как в таблице нет значения 'T_BOOKS', все значения имеют формат 'Value_XXX'.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```

     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.080..0.080 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.902 ms
     Execution Time: 0.129 ms
     ```

     *Объясните результат:*
     Используется Index Scan по индексу t_lookup_clustered_value_idx. Время выполнения (0.129 ms) быстрее, чем у обычной таблицы (0.300 ms). Кластеризованная таблица показывает лучшую производительность благодаря более эффективному расположению данных в памяти и на диске.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:

     *Сравнение:*
     - **Обычная таблица**: Execution Time: 0.300 ms, Planning Time: 2.032 ms
     - **Кластеризованная таблица**: Execution Time: 0.129 ms, Planning Time: 0.902 ms

     **Выводы:**
     - Кластеризованная таблица показала улучшение производительности в 2.3 раза (0.300 ms → 0.129 ms)
     - Время планирования также сократилось более чем в 2 раза (2.032 ms → 0.902 ms)
     - Кластеризация особенно эффективна для запросов, которые обращаются к физически близким данным
     - Для поиска по индексированным полям (не являющимся ключом кластеризации) преимущество менее заметно, но всё равно присутствует благодаря лучшей локальности данных
     - Основное преимущество кластеризации проявляется при последовательном чтении больших объемов данных и при range-запросах