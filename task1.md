## 1) Выполнить pgbench -c8 -P 6 -T 60 -U postgres postgres.
### input
* pgbench -i postgres
* pgbench -c8 -P 6 -T 60 -U postgres postgres

### Output
```
progress: 6.0 s, 2713.0 tps, lat 2.768 ms stddev 1.959, 0 failed
progress: 12.0 s, 2953.5 tps, lat 2.702 ms stddev 1.773, 0 failed
progress: 18.0 s, 2992.8 tps, lat 2.666 ms stddev 1.345, 0 failed
progress: 24.0 s, 2893.8 tps, lat 2.758 ms stddev 1.859, 0 failed
progress: 30.0 s, 2918.8 tps, lat 2.734 ms stddev 2.017, 0 failed
progress: 36.0 s, 3016.4 tps, lat 2.645 ms stddev 1.368, 0 failed
progress: 42.0 s, 2932.6 tps, lat 2.721 ms stddev 1.856, 0 failed
progress: 48.0 s, 2868.2 tps, lat 2.782 ms stddev 1.441, 0 failed
progress: 54.0 s, 2489.5 tps, lat 3.191 ms stddev 1.590, 0 failed
progress: 60.0 s, 2524.7 tps, lat 3.172 ms stddev 2.209, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 169829
number of failed transactions: 0 (0.000%)
latency average = 2.803 ms
latency stddev = 1.765 ms
initial connection time = 352.400 ms
tps = 2846.875071 (without initial connection time)
```
---

## 2) Настроить параметры vacuum/autovacuum
### postgresql.conf
```
autovacuum = on
log_autovacuum_min_duration = 0
autovacuum_max_workers = 10
autovacuum_naptime = 15s
autovacuum_vacuum_threshold = 25
autovacuum_vacuum_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 10
autovacuum_vacuum_cost_limit = 1000
```
---

## 3) Заново протестировать: pgbench -c8 -P 6 -T 60 -U postgres postgres и сравнить результаты
### input
* SELECT pg_reload_conf();
* pgbench -c8 -P 6 -T 60 -U postgres postgres
### Output
```
progress: 6.0 s, 2635.3 tps, lat 2.854 ms stddev 2.348, 0 failed
progress: 12.0 s, 3043.3 tps, lat 2.622 ms stddev 1.445, 0 failed
progress: 18.0 s, 3082.2 tps, lat 2.589 ms stddev 1.255, 0 failed
progress: 24.0 s, 3048.5 tps, lat 2.617 ms stddev 1.441, 0 failed
progress: 30.0 s, 3053.7 tps, lat 2.613 ms stddev 1.314, 0 failed
progress: 36.0 s, 3028.5 tps, lat 2.634 ms stddev 1.453, 0 failed
progress: 42.0 s, 3022.0 tps, lat 2.640 ms stddev 1.290, 0 failed
progress: 48.0 s, 2533.3 tps, lat 3.151 ms stddev 1.906, 0 failed
progress: 54.0 s, 2787.0 tps, lat 2.864 ms stddev 1.735, 0 failed
progress: 60.0 s, 2818.5 tps, lat 2.831 ms stddev 2.027, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 174348
number of failed transactions: 0 (0.000%)
latency average = 2.731 ms
latency stddev = 1.646 ms
initial connection time = 344.109 ms
tps = 2921.853687 (without initial connection time)
```
---

## 4) Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данными в размере 1 млн строк
```
CREATE TABLE test ( 
    id serial,
    fio char(100)
);
ALTER TABLE test SET (autovacuum_enabled = on);
INSERT INTO test(fio) select 'fio ' || (random()::text) FROM generate_series(1, 1000000);;
```
---

## 5) Посмотреть размер файла с таблицей
### input
* SELECT pg_size_pretty(pg_total_relation_size('test'));
### Output
* 135 MB
---

## 6) 5 раз обновить все строчки и добавить к каждой строчке любой символ
### input
```
UPDATE test SET fio = 'v1' || fio;
UPDATE test SET fio = 'v2' || fio;
UPDATE test SET fio = 'v3' || fio;
UPDATE test SET fio = 'v4' || fio;
UPDATE test SET fio = 'v5' || fio;
```
---

## 7) Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
### input
* SELECT n_dead_tup, last_autovacuum FROM pg_stat_all_tables WHERE relname = 'test';
### Output
* "n_dead_tup"	"last_autovacuum"
* 0	"2025-07-09 18:03:36.832537+03"
---

## 8) Отключить Автовакуум на таблице и опять 5 раз обновить все строки
### input
```
ALTER TABLE test SET (autovacuum_enabled = off);
UPDATE test SET fio = 'v6' || fio;
UPDATE test SET fio = 'v7' || fio;
UPDATE test SET fio = 'v8' || fio;
UPDATE test SET fio = 'v9' || fio;
UPDATE test SET fio = 'v10' || fio;
```
---

## 9. Объяснить полученные результаты
### Input
* SELECT pg_size_pretty(pg_total_relation_size('test'));
* SELECT n_dead_tup, last_autovacuum FROM pg_stat_all_tables WHERE relname = 'test';

### Output
* "pg_size_pretty"
* "808 MB"
* "n_dead_tup"	"last_autovacuum"
* 4999958	"2025-07-09 18:03:36.832537+03"

### Input
* vacuum full test;
### Output
* "pg_size_pretty" 
* "135 MB" 

### Result
* Используемый postgres механизм MVCC основан на создании новых версий записей при изменении данных.
* Autovacuum автоматически удаляет "мертвые" версии записей, предотвращая разрастание таблиц и особождая место.
* Отключив autovacuum и выполнив 5кратный update кол-во записей в таблице увеличилось на ~500%
---