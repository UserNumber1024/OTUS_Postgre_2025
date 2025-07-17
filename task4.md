## 1) Создать бекапы с помощью pg_dump, pg_dumpall и pg_basebackup сравнить скорость создания и возможности.
### init
```
create database testdb;
connect testdb
CREATE TABLE test ( 
    id serial,
    fio char(100)
);
INSERT INTO test(fio) select 'fio ' || (random()::text) FROM generate_series(1, 1000000);
```
### pg_dump -d testdb -U postgres -h localhost > C:\b\testdb1.sql
```
-- PostgreSQL database dump
-- Dumped from database version 17.5
-- Dumped by pg_dump version 17.5
...
CREATE TABLE public.test (
    id integer NOT NULL,
    fio character(100)
);
...
COPY public.test (id, fio) FROM stdin;
1	fio 0.058299980840293975                                                                            
2	fio 0.4290695725164517  
...
```
### pg_dumpall -U postgres -h localhost > C:\b\all.sql
```
-- PostgreSQL database cluster dump
...
-- Roles
...
-- Databases
...
```
### pg_basebackup -U postgres -h localhost -v -D C:\b\basebackup.sql
```
pg_basebackup: начинается базовое резервное копирование, ожидается завершение контрольной точки
pg_basebackup: контрольная точка завершена
pg_basebackup: стартовая точка в журнале предзаписи: 1/31000028 на линии времени 1
pg_basebackup: запуск фонового процесса считывания WAL
pg_basebackup: создан временный слот репликации "pg_basebackup_13520"
pg_basebackup: конечная точка в журнале предзаписи: 1/31000120
pg_basebackup: ожидание завершения потоковой передачи фоновым процессом...
pg_basebackup: сохранение данных на диске...
pg_basebackup: переименование backup_manifest.tmp в backup_manifest
pg_basebackup: базовое резервное копирование завершено
```
```
Содержимое папки C:\b\basebackup.sql
17.07.2025  23:55    <DIR>          .
17.07.2025  23:55    <DIR>          ..
17.07.2025  23:55               227 backup_label
...
17.07.2025  23:55    <DIR>          pg_xact
17.07.2025  23:55                90 postgresql.auto.conf
17.07.2025  23:55            31 668 postgresql.conf
               8 файлов        277 046 байт
              20 папок  156 293 636 096 байт свободно
```
### Summary
#### pg_dump
* резевирование вплоть до отдельной таблицы
* логическое копирование (т.е. можно восстановить на другой архитектуре)
* параллельное выполнение
#### pg_dumpall
* только кластер целиком
* логическое копирование
#### pg_basebackup
* скорость (на реальных базах) т.к. физическое копирование
* возможно восстановление на момент времени
* только кластер целиком
* восстановление строго на той же архитектуре
---

## 2) Настроить копирование WAL файлов.
### postgresql.conf 
```
wal_level = replica
archive_mode = on
archive_command = 'copy "%p" "C:\\b\\wal\\%f"'
```
### restart
```
net stop postgresql-x64-17
net start postgresql-x64-17
SELECT name, setting FROM pg_settings WHERE name IN ('archive_mode','archive_command','archive_timeout');
```

### SELECT pg_switch_wal();
```
 Содержимое папки C:\b\wal

18.07.2025  00:35    <DIR>          .
18.07.2025  00:32    <DIR>          ..
18.07.2025  00:34        16 777 216 00000001000000010000003C
18.07.2025  00:35        16 777 216 00000001000000010000003D
               2 файлов     33 554 432 байт
               2 папок  155 687 694 336 байт свободно
```
---

## 3) Восстановить базу на другой машине PostgreSQL на заданное время, используя ранее созданные бекапы и WAL файлы.
* (выполняется на одной машине)
### pg_basebackup -U postgres -h localhost -v -D C:\b\fullb\basebackup.sql
### update test set fio = 'check restore' 
### select now();
```
2025-07-18 00:51:23.923732+03

надо было сделать до изменения данных, но +- понятно куда потом откатывать )
```
### net stop postgresql-x64-17
### удалить "PostgreSQL\17\data\*.*"
### копировать "C:\b\fullb\basebackup.sql\*.*" -> "PostgreSQL\17\data\*.*"
### создать "PostgreSQL\17\data\recovery.signal"
### postgresql.conf 
```
restore_command = 'copy "C:\\b\\wal\\%f" "%p"'
recovery_target_time = '2025-07-18 00:40:00'
```
### net start postgresql-x64-17
### log
```
2025-07-18 01:11:25 MSK ВАЖНО:  система баз данных запускается
2025-07-18 01:11:31 MSK СООБЩЕНИЕ:  начинается восстановление копии с LSN redo 1/40000028, LSN контрольной точки 1/40000080, на линии времени 1
2025-07-18 01:11:31 MSK СООБЩЕНИЕ:  файл журнала "000000010000000100000040" восстановлен из архива
2025-07-18 01:11:31 MSK СООБЩЕНИЕ:  начинается восстановление точки во времени до 2025-07-18 00:40:00+03
2025-07-18 01:11:31 MSK СООБЩЕНИЕ:  запись REDO начинается со смещения 1/40000028
2025-07-18 01:11:31 MSK СООБЩЕНИЕ:  завершено восстановление копии с LSN redo 1/40000028 и конечным LSN 1/40000158
2025-07-18 01:11:31 MSK СООБЩЕНИЕ:  согласованное состояние восстановления достигнуто в позиции 1/40000158
2025-07-18 01:11:31 MSK СООБЩЕНИЕ:  файл журнала "000000010000000100000041" восстановлен из архива
...
2025-07-18 01:11:48 MSK СООБЩЕНИЕ:  восстановление останавливается перед фиксированием транзакции 345083, время 2025-07-18 00:50:26.826632+03
2025-07-18 01:11:48 MSK СООБЩЕНИЕ:  остановка в конце восстановления
2025-07-18 01:11:48 MSK ПОДСКАЗКА:  Выполните pg_wal_replay_resume() для повышения.
```
### select * from test
```
1	"fio 0.058299980840293975                                                                            "
2	"fio 0.4290695725164517                                                                              "
3	"fio 0.5309367275374992                                                                              "
4	"fio 0.6677636171209322                                                                              "
```
---