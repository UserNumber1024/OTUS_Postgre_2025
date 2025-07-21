## 1) Написать запросы поиска данных без индексов, посмотреть их план запросов.
### create test data
```
DROP TABLE test;

CREATE TABLE test ( 
    id serial,
    fio char(100),
    email text,
    date date,
    phone int
);
INSERT INTO test(fio, email, date, phone) 
SELECT 
	'fio ' || (random()::text) ,
	'email' || (random()::text)  || '@mail.su', 
	CURRENT_DATE - (random() * 1000)::int  , 
	(floor(random() * 1000000000)::int) 
FROM generate_series(1, 1000000);
```

### select pg_size_pretty(pg_indexes_size('test'));
```
"pg_size_pretty"
"0 bytes"
``` 

### explain analyze select * from test where phone = 228848862
```
"Gather  (cost=1000.00..28431.43 rows=1 width=145) (actual time=2.316..176.712 rows=1 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Parallel Seq Scan on test  (cost=0.00..27431.33 rows=1 width=145) (actual time=12.775..43.645 rows=0 loops=3)"
"        Filter: (phone = 228848862)"
"        Rows Removed by Filter: 333333"
"Planning Time: 0.055 ms"
"Execution Time: 176.733 ms"
```
### explain analyze select * from test where date >= '2025-01-01'
```
"Seq Scan on test  (cost=0.00..35756.00 rows=198783 width=152) (actual time=0.017..163.600 rows=195961 loops=1)"
"  Filter: (date >= '2025-01-01'::date)"
"  Rows Removed by Filter: 804039"
"Planning Time: 0.384 ms"
"Execution Time: 172.524 ms"
```
---

## 2) Добавить на таблицы индексы с целью оптимизации запросов поиска данных.
### input
```
create index on test (phone);
create index on test (date);
analyze test;
```
---

## 3) Сравнить новые планы запросов с предыдущими.
### explain analyze select * from test where phone = 228848862
```
"Index Scan using test_phone_idx on test  (cost=0.42..8.44 rows=1 width=145) (actual time=0.524..0.526 rows=1 loops=1)"
"  Index Cond: (phone = 228848862)"
"Planning Time: 0.085 ms"
"Execution Time: 0.542 ms"
```
### explain analyze select * from test where date >= '2025-01-01'
```
"Bitmap Heap Scan on test  (cost=2209.74..27906.56 rows=195266 width=152) (actual time=11.723..297.488 rows=195961 loops=1)"
"  Recheck Cond: (date >= '2025-01-01'::date)"
"  Heap Blocks: exact=23256"
"  ->  Bitmap Index Scan on test_date_idx  (cost=0.00..2160.92 rows=195266 width=0) (actual time=8.994..8.994 rows=195961 loops=1)"
"        Index Cond: (date >= '2025-01-01'::date)"
"Planning Time: 0.073 ms"
"Execution Time: 308.362 ms"
```

#### Профита по времени выполнения нет, но в плане видно использование индекса )
---

## 4) Сравнить применение различных типов индексов.

### create index on test (email);
* explain analyze select * from test where email ilike '%123456%'
```
"Gather  (cost=1000.00..28441.33 rows=100 width=145) (actual time=34.785..694.973 rows=13 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Parallel Seq Scan on test  (cost=0.00..27431.33 rows=42 width=145) (actual time=115.743..645.337 rows=4 loops=3)"
"        Filter: (email ~~* '%123456%'::text)"
"        Rows Removed by Filter: 333329"
"Planning Time: 0.344 ms"
"Execution Time: 694.995 ms"
```
* Обычный (btree) индекс для поиска like-конструкции не работает 

### create index email_search_idx on test using GIN (email gin_trgm_ops)
* CREATE EXTENSION IF NOT EXISTS pg_trgm;
* drop previos index on email
* explain analyze select * from test where email ilike '%123456%'
```
"Bitmap Heap Scan on test  (cost=117.26..498.38 rows=100 width=145) (actual time=1.598..1.630 rows=13 loops=1)"
"  Recheck Cond: (email ~~* '%123456%'::text)"
"  Rows Removed by Index Recheck: 2"
"  Heap Blocks: exact=15"
"  ->  Bitmap Index Scan on email_search_idx  (cost=0.00..117.23 rows=100 width=0) (actual time=1.577..1.577 rows=15 loops=1)"
"        Index Cond: (email ~~* '%123456%'::text)"
"Planning Time: 7.209 ms"
"Execution Time: 1.652 ms"
```
* Специализированный (Gin) индекс для like-конструкций по строке работает 
---