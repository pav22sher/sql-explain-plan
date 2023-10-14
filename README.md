# План выполнения запросов PostgreSQL

EXPLAIN [ ( option [, ...] ) ] statement

Option:
* ANALYZE [ boolean ] - выполнит команду и отобразит фактическое время работы и другую статистику.
* VERBOSE [ boolean ] - доп. информацию о плане.
* COSTS [ boolean ] - информация о стоймости(cost).
* SETTINGS [ boolean ] - информация о параметрах конфигурации(work_mem, enable_seqscan, enable_indexscan).
* BUFFERS [ boolean ] - информация об использовании буфера.
* SUMMARY [ boolean ] - сводная информация (Planning Time, Execution Time)
* FORMAT { TEXT | XML | JSON | YAML } - формат вывода

* GENERIC_PLAN [ boolean ] - общий план
* WAL [ boolean ] - информация о создании записей WAL
* TIMING [ boolean ] - фактическое время запуска и время потраченное на каждый узел

EXPLAIN - Эта команда отображает план выполнения, созданный планировщиком PostgreSQL для предоставленного оператора. 
План выполнения показывает, как будут сканироваться таблицы, на которые ссылается оператор 
— простым последовательным сканированием, сканированием индекса и т. д. 
— и если ссылаются на несколько таблиц, какие алгоритмы соединения 
будут использоваться для объединения необходимых строк из каждой таблицы.
Наиболее важной частью отображения является предполагаемая стоимость выполнения инструкции, 
которая представляет собой предположение планировщика о том, 
сколько времени потребуется для выполнения инструкции 
(измеряется в произвольных единицах стоимости, но обычно означает выборку страниц с диска).

### Тестовая таблица в миллион строк
```sql
CREATE TABLE foo (c1 integer, c2 text);
INSERT INTO foo
SELECT i, md5(random()::text)
FROM generate_series(1, 1000000) AS i;
```
### EXPLAIN - план выполнения
```sql
EXPLAIN SELECT * FROM foo;
--Seq Scan on foo  (cost=0.00..18334.00 rows=1000000 width=37)
```
* Seq Scan — последовательное, блок за блоком, чтение данных.
* cost - некое сферическое в вакууме понятие, призванное оценить затратность операции.
  0.00 — затраты на получение первой строки.
  18334.00 — затраты на получение всех строк.
* rows — приблизительное количество возвращаемых строк при выполнении операции.
  width — средний размер одной строки в байтах.

### ANALYZE - обновляет статистику
Попробуем добавить 10 строк
```sql
INSERT INTO foo
  SELECT i, md5(random()::text)
  FROM generate_series(1, 10) AS i;
EXPLAIN SELECT * FROM foo;
--Seq Scan on foo  (cost=0.00..18334.00 rows=1000000 width=37)
ANALYZE foo;
EXPLAIN SELECT * FROM foo;
--Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37)
```
ANALYZE:
* Считывается определённое количество строк таблицы(default_statistics_target), выбранных случайным образом
* Собирается статистика значений по каждой из колонок таблицы

### EXPLAIN (ANALYZE) - план выполнения и реальное выполнение 
Если выполнять Insert, Update, Delete, то чтобы команды не были выполнены используйте
```sql
BEGIN;
EXPLAIN ANALYZE update foo set c1 = 0 where c1 = 1;
ROLLBACK;
```

```sql
EXPLAIN (ANALYZE) SELECT * FROM foo;
--Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.037..55.733 rows=1000010 loops=1)
--Planning Time: 0.065 ms
--Execution Time: 80.387 ms
```
* actual time — реальное время в миллисекундах, затраченное для получения первой строки и всех строк соответственно.
* rows — реальное количество строк Seq Scan.
* loops — сколько раз пришлось выполнить операцию Seq Scan.
* Total runtime — общее время выполнения запроса.

### EXPLAIN(BUFFERS) - кэш БД
```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT * FROM foo;
--Seq Scan on foo (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.525..734.754 rows=1000010 loops=1)
--Buffers: shared read=8334
--Total runtime: 1253.177 ms
```
* Buffers: shared read — количество блоков, считанное с диска.
* Buffers: shared hit — количество блоков, считанных из кэша.
* Объём кэша определяется константой shared_buffers в файле postgresql.conf

### Filter
```sql
EXPLAIN SELECT * FROM foo WHERE c1 > 500;
--Seq Scan on foo  (cost=0.00..20834.13 rows=999531 width=37)
--  Filter: (c1 > 500)
```
* Filter: (c1 > 500) - каждая запись(Seq Scan) сравнивается с условием c1 > 500. 
  Если условие выполняется, запись вводится в результат. Иначе — отбрасывается.
### Index
```sql
CREATE INDEX ON foo(c1);
EXPLAIN (ANALYZE) SELECT * FROM foo WHERE c1 > 500;
--Seq Scan on foo  (cost=0.00..20834.13 rows=999517 width=37) (actual time=0.042..78.489 rows=999500 loops=1)
--  Filter: (c1 > 500)
--  Rows Removed by Filter: 510
--Planning Time: 0.096 ms
--Execution Time: 102.344 ms
```
Заставим использовать индекс
```sql
SET enable_seqscan TO off;
EXPLAIN (ANALYZE) SELECT * FROM foo WHERE c1 > 500;
--Index Scan using foo_c1_idx on foo  (cost=0.42..36800.97 rows=999517 width=37) (actual time=0.049..127.180 rows=999500 loops=1)
--  Index Cond: (c1 > 500)
--Planning Time: 0.087 ms
--Execution Time: 151.654 ms
--cost и время выполнения запроса увеличились -> планировщик не глуп!
SET enable_seqscan TO on;
```
```sql
EXPLAIN SELECT * FROM foo WHERE c1 < 500;
--Index Scan using foo_c1_idx on foo  (cost=0.42..25.04 rows=492 width=37)
--  Index Cond: (c1 < 500)
```
Поиск по второму полю
```sql
EXPLAIN SELECT *  FROM foo WHERE c1 < 500 AND c2 LIKE 'abcd%';
--Index Scan using foo_c1_idx on foo  (cost=0.42..26.26 rows=1 width=37)
--  Index Cond: (c1 < 500)
--  Filter: (c2 ~~ 'abcd%'::text)
```
```sql
EXPLAIN (ANALYZE) SELECT * FROM foo WHERE c2 LIKE 'abcd%';
--Gather  (cost=1000.00..14552.39 rows=100 width=37) (actual time=3.991..75.947 rows=23 loops=1)
--  Workers Planned: 2
--  Workers Launched: 2
--  ->  Parallel Seq Scan on foo  (cost=0.00..13542.39 rows=42 width=37) (actual time=5.845..32.429 rows=8 loops=3)
--        Filter: (c2 ~~ 'abcd%'::text)
--        Rows Removed by Filter: 333329
--Planning Time: 0.087 ms
--Execution Time: 75.975 ms
```
```sql
CREATE INDEX ON foo(c2);
EXPLAIN (ANALYZE) SELECT * FROM foo WHERE c2 LIKE 'abcd%';
--Gather  (cost=1000.00..14552.39 rows=100 width=37) (actual time=3.716..81.143 rows=23 loops=1)
--  Workers Planned: 2
--  Workers Launched: 2
--  ->  Parallel Seq Scan on foo  (cost=0.00..13542.39 rows=42 width=37) (actual time=3.287..30.691 rows=8 loops=3)
--        Filter: (c2 ~~ 'abcd%'::text)
--        Rows Removed by Filter: 333329
--Planning Time: 0.070 ms
--Execution Time: 81.171 ms
```
Опять Seq Scan? Индекс не используется потому, что база для текстовых полей использует формат UTF-8.
```sql
CREATE INDEX ON foo(c2 text_pattern_ops);
EXPLAIN SELECT * FROM foo WHERE c2 LIKE 'abcd%';
--Index Scan using foo_c2_idx1 on foo  (cost=0.42..8.45 rows=100 width=37)
--  Index Cond: ((c2 ~>=~ 'abcd'::text) AND (c2 ~<~ 'abce'::text))
--  Filter: (c2 ~~ 'abcd%'::text)
```

* Seq Scan — читается вся таблица.
* Index Scan — используется индекс для условий WHERE.
* Index Only Scan — читается только индекс.
* Bitmap Index Scan — сначала Index Scan, затем контроль выборки по таблице.
```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100;
--1) извлекает эти строки из самой таблицы
--Bitmap Heap Scan on tenk1  (cost=5.07..229.20 rows=101 width=244)
--   Recheck Cond: (unique1 < 100)
--   0) посещает индекс, чтобы найти расположение строк, соответствующих условию индекса
--   ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=101 width=0)
--         Index Cond: (unique1 < 100)
```

### ORDER BY
```sql
DROP INDEX foo_c1_idx;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM foo ORDER BY c1;
--Gather Merge  (cost=63789.95..161019.97 rows=833342 width=37) (actual time=144.119..307.011 rows=1000010 loops=1)
--  Workers Planned: 2
--  Workers Launched: 2
--  Buffers: shared hit=8428, temp read=5763 written=5786
--  ->  Sort  (cost=62789.92..63831.60 rows=416671 width=37) (actual time=88.759..120.467 rows=333337 loops=3)
--        Sort Key: c1
--        Sort Method: external merge  Disk: 24536kB
--        Worker 0:  Sort Method: external merge  Disk: 10784kB
--        Worker 1:  Sort Method: external merge  Disk: 10784kB
--        Buffers: shared hit=8428, temp read=5763 written=5786
--        ->  Parallel Seq Scan on foo  (cost=0.00..12500.71 rows=416671 width=37) (actual time=0.016..22.845 rows=333337 loops=3)
--              Buffers: shared hit=8334
--Planning Time: 0.070 ms
--Execution Time: 336.421 ms
```
* Сначала производится Seq Scan таблицы foo. Затем сортировка Sort. 
  В выводе команды EXPLAIN знак -> указывает на иерархию действий (node). 
  Чем раньше выполняется действие, тем с большим отступом оно отображается.
* Sort Key — условие сортировки.
* Sort Method: external merge Disk — при сортировке используется временный файл на диске объёмом 24536kB.
  temp read=5763 written=5786 - во временный файл было записано 5786 и прочитано 5763 блоков по 8Kb.
Операции с файловой системой более медленные, чем операции в оперативной памяти.
```sql
show work_mem; --текущая величина
SET work_mem TO '200MB'; --увеличить объём используемой памяти
EXPLAIN (ANALYZE) SELECT * FROM foo ORDER BY c1;
--Sort  (cost=117993.01..120493.04 rows=1000010 width=37) (actual time=248.669..274.536 rows=1000010 loops=1)
--  Sort Key: c1
--  Sort Method: quicksort  Memory: 102702kB
--  ->  Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.011..57.132 rows=1000010 loops=1)
--Planning Time: 0.067 ms
--Execution Time: 307.241 ms
```
* Sort Method: quicksort  Memory: 102702kB - сортировка целиком проведена в оперативной памяти.
* work_mem - указывает объем памяти, который будет использоваться внутренними операциями сортировки и хеш-таблицами.
Индекс:
```sql
CREATE INDEX ON foo(c1);
EXPLAIN (ANALYZE) SELECT * FROM foo ORDER BY c1;
--Index Scan using foo_c1_idx on foo  (cost=0.42..34317.58 rows=1000010 width=37) (actual time=0.040..108.552 rows=1000010 loops=1)
--Planning Time: 1.228 ms
--Execution Time: 132.487 ms
```
```sql
DROP INDEX foo_c2_idx1;
EXPLAIN (ANALYZE,BUFFERS) SELECT * FROM foo WHERE c2 LIKE 'ab%' LIMIT 10;
--Limit  (cost=0.00..2083.41 rows=10 width=37) (actual time=0.034..0.229 rows=10 loops=1)
--  Buffers: shared hit=15
--  ->  Seq Scan on foo  (cost=0.00..20834.13 rows=100 width=37) (actual time=0.033..0.227 rows=10 loops=1)
--        Filter: (c2 ~~ 'ab%'::text)
--        Rows Removed by Filter: 1686
--        Buffers: shared hit=15
--Planning Time: 0.674 ms
--Execution Time: 0.239 ms
```
* Производится сканирование Seq Scan строк таблицы и сравнение Filter их с условием. 
  Как только наберётся 10 записей, удовлетворяющих условию, сканирование закончится.
* Rows Removed by Filter - для того, чтобы получить 10 строк результата, пришлось прочитать не всю таблицу,
  а только 1696 записи, из них 1686 были отвергнуты.

### JOIN
Создадим новую таблицу, соберём для неё статистику.
```sql
CREATE TABLE bar (c1 integer, c2 boolean);
INSERT INTO bar
  SELECT i, i%2=1
  FROM generate_series(1, 500000) AS i;
ANALYZE bar;
```
Запрос по двум таблицам
```sql
--EXPLAIN (ANALYZE) SELECT * FROM foo JOIN bar ON foo.c1=bar.c1;
--3) для каждой строки for вычисляется хэш и сравнивается с хэшем таблицы bar по условию Hash Cond
--Hash Join  (cost=13463.00..40547.14 rows=500000 width=42) (actual time=98.986..466.112 rows=500010 loops=1)
--  Hash Cond: (foo.c1 = bar.c1)
--  2) сканируется вся таблица foo
--  ->  Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.009..55.650 rows=1000010 loops=1)
--  1) для каждой строки вычисляется hash
--  ->  Hash  (cost=7213.00..7213.00 rows=500000 width=5) (actual time=98.218..98.219 rows=500000 loops=1)
--        Buckets: 524288  Batches: 1  Memory Usage: 22163kB -- использована памяти для размещения хеш таблицы 
--        0) сканируется вся таблица bar
--        ->  Seq Scan on bar  (cost=0.00..7213.00 rows=500000 width=5) (actual time=0.026..25.083 rows=500000 loops=1)
--Planning Time: 0.792 ms
--Execution Time: 480.325 ms
```
Добавим индекс
```sql
CREATE INDEX ON bar(c1);
EXPLAIN (ANALYZE) SELECT * FROM foo JOIN bar ON foo.c1=bar.c1;
--Merge Join  (cost=1.34..39919.37 rows=500000 width=42) (actual time=0.025..358.296 rows=500010 loops=1)
--    Merge Cond: (foo.c1 = bar.c1)
--    ->  Index Scan using foo_c1_idx on foo  (cost=0.42..34317.58 rows=1000010 width=37) (actual time=0.013..121.656 rows=500011 loops=1)
--    ->  Index Scan using bar_c1_idx on bar  (cost=0.42..15212.42 rows=500000 width=5) (actual time=0.009..122.819 rows=500010 loops=1)
--    Planning Time: 2.197 ms
--    Execution Time: 371.486 ms
```
Merge Join и Index Scan по индексам обеих таблиц дают впечатляющий прирост производительности.
```sql
EXPLAIN (ANALYZE) SELECT * FROM foo LEFT JOIN bar ON foo.c1=bar.c1;
--Hash Left Join  (cost=13463.00..40547.14 rows=1000010 width=42) (actual time=105.179..525.166 rows=1000010 loops=1)
--  Hash Cond: (foo.c1 = bar.c1)
--  ->  Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.010..77.637 rows=1000010 loops=1)
--  ->  Hash  (cost=7213.00..7213.00 rows=500000 width=5) (actual time=104.336..104.336 rows=500000 loops=1)
--        Buckets: 524288  Batches: 1  Memory Usage: 22163kB
--        ->  Seq Scan on bar  (cost=0.00..7213.00 rows=500000 width=5) (actual time=0.024..25.652 rows=500000 loops=1)
--Planning Time: 0.190 ms
--Execution Time: 551.154 ms
```
Seq Scan? Посмотрим, какие результаты будут, если запретить Seq Scan.
```sql
SET enable_seqscan TO off; 
EXPLAIN (ANALYZE)
SELECT * FROM foo LEFT JOIN bar ON foo.c1=bar.c1;
--Merge Left Join  (cost=1.34..58280.02 rows=1000010 width=42) (actual time=0.020..299.802 rows=1000010 loops=1)
--    Merge Cond: (foo.c1 = bar.c1)
--    ->  Index Scan using foo_c1_idx on foo  (cost=0.42..34317.58 rows=1000010 width=37) (actual time=0.008..99.601 rows=1000010 loops=1)
--    ->  Index Scan using bar_c1_idx on bar  (cost=0.42..15212.42 rows=500000 width=5) (actual time=0.008..49.568 rows=500010 loops=1)
--    Planning Time: 0.287 ms
--    Execution Time: 325.750 ms
```
По мнению планировщика, использование индексов затратнее, чем использование хэшей. 
Такое возможно при достаточно большом объёме выделенной памяти. Помните, мы увеличивали work_mem?
```sql
SET enable_seqscan TO ON;
SET work_mem TO '15MB';
--EXPLAIN (ANALYZE) SELECT * FROM foo LEFT JOIN bar ON foo.c1=bar.c1;
--Merge Left Join  (cost=1.34..58280.02 rows=1000010 width=42) (actual time=0.013..298.890 rows=1000010 loops=1)
--    Merge Cond: (foo.c1 = bar.c1)
--    ->  Index Scan using foo_c1_idx on foo  (cost=0.42..34317.58 rows=1000010 width=37) (actual time=0.005..99.043 rows=1000010 loops=1)
--    ->  Index Scan using bar_c1_idx on bar  (cost=0.42..15212.42 rows=500000 width=5) (actual time=0.005..49.487 rows=500010 loops=1)
--    Planning Time: 0.170 ms
```
Что будет, если запретить Index Scan
```sql
SET enable_indexscan TO off;
EXPLAIN (ANALYZE) SELECT * FROM foo LEFT JOIN bar ON foo.c1=bar.c1;
--Hash Left Join  (cost=15417.00..60081.14 rows=1000010 width=42) (actual time=134.595..638.848 rows=1000010 loops=1)
--  Hash Cond: (foo.c1 = bar.c1)
--  ->  Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.009..56.284 rows=1000010 loops=1)
--  ->  Hash  (cost=7213.00..7213.00 rows=500000 width=5) (actual time=133.693..133.694 rows=500000 loops=1)
--        Buckets: 524288  Batches: 2  Memory Usage: 13118kB
--        ->  Seq Scan on bar  (cost=0.00..7213.00 rows=500000 width=5) (actual time=0.025..27.135 rows=500000 loops=1)
--Planning Time: 0.275 ms
--Execution Time: 664.638 ms
SET enable_indexscan TO on;
```
* cost явно увеличился и причина в Batches: 2.
  Весь хэш не поместился в память, его пришлось разбить на 2 пакета.
* 

### Ссылки
* https://www.postgresql.org/docs/current/sql-explain.html
* https://habr.com/ru/articles/203320/