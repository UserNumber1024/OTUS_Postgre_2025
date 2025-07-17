## 1) Выполнить pgbench -c8 -P 6 -T 60 -U postgres postgres.
### init
```
отключение autovacuum в конфиге
net stop postgresql-x64-17
net start postgresql-x64-17
```
### input
* pgbench -i postgres
* pgbench -c8 -P 6 -T 60 -U postgres postgres

### Output
```
progress: 6.0 s, 2740.8 tps, lat 2.735 ms stddev 2.091, 0 failed
progress: 12.0 s, 3032.7 tps, lat 2.631 ms stddev 1.332, 0 failed
progress: 18.0 s, 2999.2 tps, lat 2.660 ms stddev 1.513, 0 failed
progress: 24.0 s, 3070.0 tps, lat 2.598 ms stddev 1.304, 0 failed
progress: 30.0 s, 2976.5 tps, lat 2.681 ms stddev 1.589, 0 failed
progress: 36.0 s, 2995.2 tps, lat 2.664 ms stddev 1.355, 0 failed
progress: 42.0 s, 2440.8 tps, lat 3.270 ms stddev 23.601, 0 failed
progress: 48.0 s, 2982.0 tps, lat 2.676 ms stddev 1.332, 0 failed
progress: 54.0 s, 2953.5 tps, lat 2.701 ms stddev 1.476, 0 failed
progress: 60.0 s, 2904.2 tps, lat 2.747 ms stddev 1.422, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 174573
number of failed transactions: 0 (0.000%)
latency average = 2.726 ms
latency stddev = 6.987 ms
initial connection time = 363.968 ms
tps = 2926.870242 (without initial connection time)
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
```
net stop postgresql-x64-17
net start postgresql-x64-17
SHOW autovacuum; --on
```
### pgbench -c8 -P 6 -T 60 -U postgres postgres
```
progress: 6.0 s, 2817.3 tps, lat 2.667 ms stddev 1.896, 0 failed
progress: 12.0 s, 3078.1 tps, lat 2.592 ms stddev 1.277, 0 failed
progress: 18.0 s, 3037.5 tps, lat 2.626 ms stddev 1.444, 0 failed
progress: 24.0 s, 3048.7 tps, lat 2.617 ms stddev 1.453, 0 failed
progress: 30.0 s, 3059.3 tps, lat 2.608 ms stddev 1.323, 0 failed
progress: 36.0 s, 2980.6 tps, lat 2.677 ms stddev 1.505, 0 failed
progress: 42.0 s, 2982.7 tps, lat 2.675 ms stddev 1.576, 0 failed
progress: 48.0 s, 2981.2 tps, lat 2.676 ms stddev 1.477, 0 failed
progress: 54.0 s, 2976.0 tps, lat 2.681 ms stddev 1.344, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 179429
number of failed transactions: 0 (0.000%)
latency average = 2.652 ms
latency stddev = 1.500 ms
initial connection time = 348.240 ms
tps = 3008.057458 (without initial connection time)
```
* существенно уменьшилось latency stddev, стандартное отклонение задержки выполнения транзакций

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