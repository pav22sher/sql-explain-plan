# ���� ���������� �������� PostgreSQL

EXPLAIN [ ( option [, ...] ) ] statement

Option:
* ANALYZE [ boolean ] - �������� ������� � ��������� ����������� ����� ������ � ������ ����������.
* VERBOSE [ boolean ] - ���. ���������� � �����.
* COSTS [ boolean ] - ���������� � ���������(cost).
* SETTINGS [ boolean ] - ���������� � ���������� ������������(work_mem, enable_seqscan, enable_indexscan).
* BUFFERS [ boolean ] - ���������� �� ������������� ������.
* SUMMARY [ boolean ] - ������� ���������� (Planning Time, Execution Time)
* FORMAT { TEXT | XML | JSON | YAML } - ������ ������

* GENERIC_PLAN [ boolean ] - ����� ����
* WAL [ boolean ] - ���������� � �������� ������� WAL
* TIMING [ boolean ] - ����������� ����� ������� � ����� ����������� �� ������ ����

EXPLAIN - ��� ������� ���������� ���� ����������, ��������� ������������� PostgreSQL ��� ���������������� ���������. 
���� ���������� ����������, ��� ����� ������������� �������, �� ������� ��������� �������� 
� ������� ���������������� �������������, ������������� ������� � �. �. 
� � ���� ��������� �� ��������� ������, ����� ��������� ���������� 
����� �������������� ��� ����������� ����������� ����� �� ������ �������.
�������� ������ ������ ����������� �������� �������������� ��������� ���������� ����������, 
������� ������������ ����� ������������� ������������ � ���, 
������� ������� ����������� ��� ���������� ���������� 
(���������� � ������������ �������� ���������, �� ������ �������� ������� ������� � �����).

### �������� ������� � ������� �����
```sql
CREATE TABLE foo (c1 integer, c2 text);
INSERT INTO foo
SELECT i, md5(random()::text)
FROM generate_series(1, 1000000) AS i;
```
### EXPLAIN - ���� ����������
```sql
EXPLAIN SELECT * FROM foo;
--Seq Scan on foo  (cost=0.00..18334.00 rows=1000000 width=37)
```
* Seq Scan � ����������������, ���� �� ������, ������ ������.
* cost - ����� ����������� � ������� �������, ���������� ������� ����������� ��������.
  0.00 � ������� �� ��������� ������ ������.
  18334.00 � ������� �� ��������� ���� �����.
* rows � ��������������� ���������� ������������ ����� ��� ���������� ��������.
  width � ������� ������ ����� ������ � ������.

### ANALYZE - ��������� ����������
��������� �������� 10 �����
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
* ����������� ����������� ���������� ����� �������(default_statistics_target), ��������� ��������� �������
* ���������� ���������� �������� �� ������ �� ������� �������

### EXPLAIN (ANALYZE) - ���� ���������� � �������� ���������� 
���� ��������� Insert, Update, Delete, �� ����� ������� �� ���� ��������� �����������
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
* actual time � �������� ����� � �������������, ����������� ��� ��������� ������ ������ � ���� ����� ��������������.
* rows � �������� ���������� ����� Seq Scan.
* loops � ������� ��� �������� ��������� �������� Seq Scan.
* Total runtime � ����� ����� ���������� �������.

### EXPLAIN(BUFFERS) - ��� ��
```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT * FROM foo;
--Seq Scan on foo (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.525..734.754 rows=1000010 loops=1)
--Buffers: shared read=8334
--Total runtime: 1253.177 ms
```
* Buffers: shared read � ���������� ������, ��������� � �����.
* Buffers: shared hit � ���������� ������, ��������� �� ����.
* ����� ���� ������������ ���������� shared_buffers � ����� postgresql.conf

### Filter
```sql
EXPLAIN SELECT * FROM foo WHERE c1 > 500;
--Seq Scan on foo  (cost=0.00..20834.13 rows=999531 width=37)
--  Filter: (c1 > 500)
```
* Filter: (c1 > 500) - ������ ������(Seq Scan) ������������ � �������� c1 > 500. 
  ���� ������� �����������, ������ �������� � ���������. ����� � �������������.
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
�������� ������������ ������
```sql
SET enable_seqscan TO off;
EXPLAIN (ANALYZE) SELECT * FROM foo WHERE c1 > 500;
--Index Scan using foo_c1_idx on foo  (cost=0.42..36800.97 rows=999517 width=37) (actual time=0.049..127.180 rows=999500 loops=1)
--  Index Cond: (c1 > 500)
--Planning Time: 0.087 ms
--Execution Time: 151.654 ms
--cost � ����� ���������� ������� ����������� -> ����������� �� ����!
SET enable_seqscan TO on;
```
```sql
EXPLAIN SELECT * FROM foo WHERE c1 < 500;
--Index Scan using foo_c1_idx on foo  (cost=0.42..25.04 rows=492 width=37)
--  Index Cond: (c1 < 500)
```
����� �� ������� ����
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
����� Seq Scan? ������ �� ������������ ������, ��� ���� ��� ��������� ����� ���������� ������ UTF-8.
```sql
CREATE INDEX ON foo(c2 text_pattern_ops);
EXPLAIN SELECT * FROM foo WHERE c2 LIKE 'abcd%';
--Index Scan using foo_c2_idx1 on foo  (cost=0.42..8.45 rows=100 width=37)
--  Index Cond: ((c2 ~>=~ 'abcd'::text) AND (c2 ~<~ 'abce'::text))
--  Filter: (c2 ~~ 'abcd%'::text)
```

* Seq Scan � �������� ��� �������.
* Index Scan � ������������ ������ ��� ������� WHERE.
* Index Only Scan � �������� ������ ������.
* Bitmap Index Scan � ������� Index Scan, ����� �������� ������� �� �������.
```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100;
--1) ��������� ��� ������ �� ����� �������
--Bitmap Heap Scan on tenk1  (cost=5.07..229.20 rows=101 width=244)
--   Recheck Cond: (unique1 < 100)
--   0) �������� ������, ����� ����� ������������ �����, ��������������� ������� �������
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
* ������� ������������ Seq Scan ������� foo. ����� ���������� Sort. 
  � ������ ������� EXPLAIN ���� -> ��������� �� �������� �������� (node). 
  ��� ������ ����������� ��������, ��� � ������� �������� ��� ������������.
* Sort Key � ������� ����������.
* Sort Method: external merge Disk � ��� ���������� ������������ ��������� ���� �� ����� ������� 24536kB.
  temp read=5763 written=5786 - �� ��������� ���� ���� �������� 5786 � ��������� 5763 ������ �� 8Kb.
�������� � �������� �������� ����� ���������, ��� �������� � ����������� ������.
```sql
show work_mem; --������� ��������
SET work_mem TO '200MB'; --��������� ����� ������������ ������
EXPLAIN (ANALYZE) SELECT * FROM foo ORDER BY c1;
--Sort  (cost=117993.01..120493.04 rows=1000010 width=37) (actual time=248.669..274.536 rows=1000010 loops=1)
--  Sort Key: c1
--  Sort Method: quicksort  Memory: 102702kB
--  ->  Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.011..57.132 rows=1000010 loops=1)
--Planning Time: 0.067 ms
--Execution Time: 307.241 ms
```
* Sort Method: quicksort  Memory: 102702kB - ���������� ������� ��������� � ����������� ������.
* work_mem - ��������� ����� ������, ������� ����� �������������� ����������� ���������� ���������� � ���-���������.
������:
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
* ������������ ������������ Seq Scan ����� ������� � ��������� Filter �� � ��������. 
  ��� ������ �������� 10 �������, ��������������� �������, ������������ ����������.
* Rows Removed by Filter - ��� ����, ����� �������� 10 ����� ����������, �������� ��������� �� ��� �������,
  � ������ 1696 ������, �� ��� 1686 ���� ����������.

### JOIN
�������� ����� �������, ������ ��� �� ����������.
```sql
CREATE TABLE bar (c1 integer, c2 boolean);
INSERT INTO bar
  SELECT i, i%2=1
  FROM generate_series(1, 500000) AS i;
ANALYZE bar;
```
������ �� ���� ��������
```sql
--EXPLAIN (ANALYZE) SELECT * FROM foo JOIN bar ON foo.c1=bar.c1;
--3) ��� ������ ������ for ����������� ��� � ������������ � ����� ������� bar �� ������� Hash Cond
--Hash Join  (cost=13463.00..40547.14 rows=500000 width=42) (actual time=98.986..466.112 rows=500010 loops=1)
--  Hash Cond: (foo.c1 = bar.c1)
--  2) ����������� ��� ������� foo
--  ->  Seq Scan on foo  (cost=0.00..18334.10 rows=1000010 width=37) (actual time=0.009..55.650 rows=1000010 loops=1)
--  1) ��� ������ ������ ����������� hash
--  ->  Hash  (cost=7213.00..7213.00 rows=500000 width=5) (actual time=98.218..98.219 rows=500000 loops=1)
--        Buckets: 524288  Batches: 1  Memory Usage: 22163kB -- ������������ ������ ��� ���������� ��� ������� 
--        0) ����������� ��� ������� bar
--        ->  Seq Scan on bar  (cost=0.00..7213.00 rows=500000 width=5) (actual time=0.026..25.083 rows=500000 loops=1)
--Planning Time: 0.792 ms
--Execution Time: 480.325 ms
```
������� ������
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
Merge Join � Index Scan �� �������� ����� ������ ���� ������������ ������� ������������������.
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
Seq Scan? ���������, ����� ���������� �����, ���� ��������� Seq Scan.
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
�� ������ ������������, ������������� �������� ���������, ��� ������������� �����. 
����� �������� ��� ���������� ������� ������ ���������� ������. �������, �� ����������� work_mem?
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
��� �����, ���� ��������� Index Scan
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
* cost ���� ���������� � ������� � Batches: 2.
  ���� ��� �� ���������� � ������, ��� �������� ������� �� 2 ������.
* 

### ������
* https://www.postgresql.org/docs/current/sql-explain.html
* https://habr.com/ru/articles/203320/