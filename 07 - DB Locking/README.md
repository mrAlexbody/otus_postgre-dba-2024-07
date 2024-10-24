# Домашнее задание 07
## Механизм блокировок

### Цель:
* Понимать как работает механизм блокировок объектов и строк.

### Описание/Пошаговая инструкция выполнения домашнего задания:
* Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
* Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
* Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
* Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
* _**Задание со звездочкой**\*_: Попробуйте воспроизвести такую ситуацию.
----
#### _Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения._

Подключимся к БД и введём настройки, предварительно их проверим: 
```postgresql
a_myskin@db01:~$ sudo -i -u postgres psql
psql (16.4 (Ubuntu 16.4-0ubuntu0.24.04.2))
Type "help" for help.

postgres=# SELECT * FROM pg_settings WHERE NAME LIKE 'log_min_duration_statement' \gx
-[ RECORD 1 ]---+---------------------------------------------------------------------------
name            | log_min_duration_statement
setting         | -1
unit            | ms
category        | Reporting and Logging / When to Log
short_desc      | Sets the minimum execution time above which all statements will be logged.
extra_desc      | Zero prints all queries. -1 turns this feature off.
context         | superuser
vartype         | integer
source          | default
min_val         | -1
max_val         | 2147483647
enumvals        |
boot_val        | -1
reset_val       | -1
sourcefile      |
sourceline      |
pending_restart | f

postgres=# SELECT * FROM pg_settings WHERE NAME LIKE 'log_lock_waits' \gx
-[ RECORD 1 ]---+------------------------------------
name            | log_lock_waits
setting         | off
unit            |
category        | Reporting and Logging / What to Log
short_desc      | Logs long lock waits.
extra_desc      |
context         | superuser
vartype         | bool
source          | default
min_val         |
max_val         |
enumvals        |
boot_val        | off
reset_val       | off
sourcefile      |
sourceline      |
pending_restart | f

postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)

 postgres=# SELECT * FROM pg_settings WHERE NAME LIKE 'log_statement' \gx
-[ RECORD 1 ]---+------------------------------------
name            | log_statement
setting         | none
unit            |
category        | Reporting and Logging / What to Log
short_desc      | Sets the type of statements logged.
extra_desc      |
context         | superuser
vartype         | enum
source          | default
min_val         |
max_val         |
enumvals        | {none,ddl,mod,all}
boot_val        | none
reset_val       | none
sourcefile      |
sourceline      |postgres=# SELECT * FROM pg_settings WHERE NAME LIKE 'logging_collector' \gx
-[ RECORD 1 ]---+---------------------------------------------------------------------------
name            | logging_collector
setting         | off
unit            |
category        | Reporting and Logging / Where to Log
short_desc      | Start a subprocess to capture stderr output and/or csvlogs into log files.
extra_desc      |
context         | postmaster
vartype         | bool
source          | default
min_val         |
max_val         |
enumvals        |
boot_val        | off
reset_val       | off
sourcefile      |
sourceline      |
pending_restart | f

postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# ALTER DATABASE postgres SET log_statement = 'all';
ALTER DATABASE
postgres=# ALTER SYSTEM SET logging_collector = on;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET deadlock_timeout to 200;
ALTER SYSTEM
```
Проверим применение настроек:
```postgresql
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SHOW  log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

postgres=# SHOW log_statement;
 log_statement
---------------
 none
(1 row)
postgres=# \q
```
```shell

a_myskin@db01:~$ sudo systemctl restart postgresql
[sudo] password for a_myskin:
a_myskin@db01:~$ sudo -i -u postgres psql
psql (16.4 (Ubuntu 16.4-0ubuntu0.24.04.2))
Type "help" for help.
```
```postgresql
postgres=# SHOW log_statement;
 log_statement
---------------
 all
(1 row)

postgres=# SHOW logging_collector;
 logging_collector
-------------------
 on
(1 row)

postgres=#
postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)

```
Создадим таблицу с данными и подключимся из 2-х сессий к базе, потом смоделируем ситуацию блокировки и посмотрим что в логах: 
```postgresql
postgres=# CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1100.00), (2,2000.00), (3,3000.00), (5,5000.00) ,(6,6000.00);
CREATE TABLE
INSERT 0 5
```
**Сессия: 1**
```postgresql

postgres=# SELECT * FROM accounts;
 acc_no | amount
--------+---------
      2 | 2000.00
      3 | 3000.00
      5 | 5000.00
      6 | 6000.00
      1 | 1100.00
(5 rows)

postgres=# BEGIN;
SELECT * FROM accounts;
BEGIN
 acc_no | amount
--------+---------
      2 | 2000.00
      3 | 3000.00
      5 | 5000.00
      6 | 6000.00
      1 | 1100.00
(5 rows)

postgres=*# SELECT pg_backend_pid();
 pg_backend_pid
----------------
           3173
(1 row)

postgres=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'accounts'::regclass;
 locktype |      mode       | granted | pid  | wait_for
----------+-----------------+---------+------+----------
 relation | AccessShareLock | t       | 3173 | {}
(1 row)

postgres=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
FROM pg_locks WHERE relation = 'accounts'::regclass;
 locktype |        mode         | granted | pid  | wait_for
----------+---------------------+---------+------+----------
 relation | AccessShareLock     | t       | 3173 | {}
 relation | AccessExclusiveLock | f       | 3424 | {3173}
(2 rows)
```
**Сессия: 2**
```postgresql
postgres=# BEGIN;
SELECT pg_backend_pid();
BEGIN
 pg_backend_pid
----------------
           3424
(1 row)

postgres=*# ^C
postgres=*# COMMIT;
COMMIT
postgres=# BEGIN;
BEGIN
postgres=*# LOCK TABLE accounts;

```
>> Смотрим лог:
````shell
.....
2024-10-23 10:36:18.718 UTC [3424] postgres@postgres LOG:  statement: BEGIN;
2024-10-23 10:36:24.266 UTC [3424] postgres@postgres LOG:  statement: LOCK TABLE accounts;
2024-10-23 10:36:24.467 UTC [3424] postgres@postgres LOG:  process 3424 still waiting for AccessExclusiveLock on relation 16468 of database 5 after 200.376 ms
2024-10-23 10:36:24.467 UTC [3424] postgres@postgres DETAIL:  Process holding the lock: 3173. Wait queue: 3424.
2024-10-23 10:36:24.467 UTC [3424] postgres@postgres STATEMENT:  LOCK TABLE accounts;
2024-10-23 10:36:36.636 UTC [3173] postgres@postgres LOG:  statement: COMMIT;
2024-10-23 10:36:36.637 UTC [3424] postgres@postgres LOG:  process 3424 acquired AccessExclusiveLock on relation 16468 of database 5 after 12370.236 ms
2024-10-23 10:36:36.637 UTC [3424] postgres@postgres STATEMENT:  LOCK TABLE accounts;
......
````
#### Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
Сделаем UPDATE в сессии 1:
```postgresql
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = amount + 100 WHERE acc_no = 5;
UPDATE 1
```
Сделаем UPDATE в сессии 2:
```postgresql
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = amount + 200 WHERE acc_no = 5;
```
Сделаем UPDATE в сессии 3:
```postgresql
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = amount + 300 WHERE acc_no = 5;
```
Определимся с некоторыми параметрами, сделаем это в сессии 1:
```postgresql
postgres=# SELECT pg_relation_filepath('accounts');
 pg_relation_filepath
----------------------
 base/5/16388
(1 row)

postgres=*# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          21334
(1 row)
```
Смотрим в логах, что: 
```shell
2024-10-23 13:31:51.987 UTC [21334] postgres@postgres LOG:  statement: SHOW  log_lock_waits;
2024-10-23 13:31:51.988 UTC [21334] postgres@postgres LOG:  duration: 1.066 ms
2024-10-23 13:32:01.370 UTC [21334] postgres@postgres LOG:  statement: SHOW log_statement;
2024-10-23 13:32:01.370 UTC [21334] postgres@postgres LOG:  duration: 0.112 ms
2024-10-23 13:32:41.442 UTC [21334] postgres@postgres LOG:  statement: SHOW deadlock_timeout;
2024-10-23 13:32:41.442 UTC [21334] postgres@postgres LOG:  duration: 0.118 ms
2024-10-23 14:13:52.050 UTC [21334] postgres@postgres LOG:  statement: SELECT pg_relation_filepath('accounts');
2024-10-23 14:13:52.050 UTC [21334] postgres@postgres LOG:  duration: 0.449 ms
2024-10-23 15:09:13.240 UTC [21334] postgres@postgres LOG:  statement: BEGIN;
2024-10-23 15:09:13.240 UTC [21334] postgres@postgres LOG:  duration: 0.200 ms
2024-10-23 15:09:51.800 UTC [21334] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 5;
2024-10-23 15:09:51.805 UTC [21334] postgres@postgres LOG:  duration: 4.955 ms
2024-10-23 15:10:05.198 UTC [21326] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 200 WHERE acc_no = 5;
2024-10-23 15:10:05.414 UTC [21326] postgres@postgres LOG:  process 21326 still waiting for ShareLock on transaction 744 after 200.380 ms
2024-10-23 15:10:05.414 UTC [21326] postgres@postgres DETAIL:  Process holding the lock: 21334. Wait queue: 21326.
2024-10-23 15:10:05.414 UTC [21326] postgres@postgres CONTEXT:  while updating tuple (0,4) in relation "accounts"
2024-10-23 15:10:05.414 UTC [21326] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 200 WHERE acc_no = 5;
2024-10-23 15:10:12.616 UTC [21451] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 300 WHERE acc_no = 5;
2024-10-23 15:10:12.817 UTC [21451] postgres@postgres LOG:  process 21451 still waiting for ExclusiveLock on tuple (0,4) of relation 16388 of database 5 after 200.111 ms
2024-10-23 15:10:12.817 UTC [21451] postgres@postgres DETAIL:  Process holding the lock: 21326. Wait queue: 21451.
2024-10-23 15:10:12.817 UTC [21451] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 300 WHERE acc_no = 5;
2024-10-23 15:10:29.156 UTC [21130] LOG:  checkpoint starting: time
```
Выясним каким процессом кто блокируется:
```postgresql
postgres=*# select pg_blocking_pids(21326);
 pg_blocking_pids
------------------
 {21334}
(1 row)

postgres=*# select pg_blocking_pids(21451);
 pg_blocking_pids
------------------
 {21326}
(1 row)
```
Изучим блокировки в представлении pg_locks: 
```postgresql
postgres=*# SELECT locktype, mode, granted,fastpath, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks WHERE relation = 'accounts'::regclass ORDER BY locktype;
 locktype |       mode       | granted | fastpath |  pid  | wait_for
----------+------------------+---------+----------+-------+----------
 relation | RowExclusiveLock | t       | t        | 21451 | {21326}
 relation | RowExclusiveLock | t       | t        | 21334 | {}
 relation | RowExclusiveLock | t       | t        | 21326 | {21334}
 tuple    | ExclusiveLock    | f       | f        | 21451 | {21326}
 tuple    | ExclusiveLock    | t       | f        | 21326 | {21334}
(5 rows)
```
>> Из этого следует. В первой транзакции получилась эксклюзивная блокировка при обновлении данных tuple в таблице "accounts", а остальные эксклюзивные блокировки ждущие завершения сессий.

####  Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
Запустил в сессии 1: 
```postgresql
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
UPDATE 1
```
Запустил в сессии 2:
```postgresql
postgres=# BEGIN;
BEGIN
postgres=*#  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 2;
UPDATE 1
```
Запустил в сессии 3:
```postgresql
postgres=# BEGIN;
BEGIN
postgres=*#  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
UPDATE 1
```
Потом ещё разок UPDATE:
Запустил в сессии 3:
```postgresql
postgres=*#  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
UPDATE 1
postgres=*# COMMIT;
COMMIT
```
Запустил в сессии 2 (блокировка, ждём):
```postgresql
postgres=*#  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
postgres=*# COMMIT;
COMMIT
```
Запустил в сессии 1 (ошибка):
```postgresql
postgres=*# UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
ERROR:  deadlock detected
DETAIL:  Process 21334 waits for ShareLock on transaction 760; blocked by process 21451.
Process 21451 waits for ShareLock on transaction 758; blocked by process 21334.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,22) in relation "accounts"
postgres=!# COMMIT;
ROLLBACK
```
>> В логах сложно понять какая транзакция была в каждой сессии, но видно что deadlock зафиксирован.
```shell
2024-10-23 16:28:19.065 UTC [21451] postgres@postgres CONTEXT:  while updating tuple (0,15) in relation "accounts"
2024-10-23 16:28:19.065 UTC [21451] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
2024-10-23 16:28:19.065 UTC [21451] postgres@postgres LOG:  duration: 73494.442 ms
2024-10-23 16:28:22.035 UTC [21451] postgres@postgres LOG:  statement: COMMIT;
2024-10-23 16:28:22.035 UTC [21451] postgres@postgres WARNING:  there is no transaction in progress
2024-10-23 16:28:22.035 UTC [21451] postgres@postgres LOG:  duration: 0.512 ms
2024-10-23 16:28:48.599 UTC [21334] postgres@postgres LOG:  statement: BEGIN;
2024-10-23 16:28:48.600 UTC [21334] postgres@postgres LOG:  duration: 0.095 ms
2024-10-23 16:28:56.521 UTC [21334] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
2024-10-23 16:28:56.522 UTC [21334] postgres@postgres LOG:  duration: 0.312 ms
2024-10-23 16:29:11.154 UTC [21326] postgres@postgres LOG:  statement: BEGIN;
2024-10-23 16:29:11.155 UTC [21326] postgres@postgres LOG:  duration: 0.117 ms
2024-10-23 16:29:15.190 UTC [21326] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 2;
2024-10-23 16:29:15.190 UTC [21326] postgres@postgres LOG:  duration: 0.342 ms
2024-10-23 16:29:21.716 UTC [21451] postgres@postgres LOG:  statement: BEGIN;
2024-10-23 16:29:21.716 UTC [21451] postgres@postgres LOG:  duration: 0.091 ms
2024-10-23 16:29:27.258 UTC [21451] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
2024-10-23 16:29:27.258 UTC [21451] postgres@postgres LOG:  duration: 0.596 ms
2024-10-23 16:29:35.903 UTC [21451] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
2024-10-23 16:29:36.103 UTC [21451] postgres@postgres LOG:  process 21451 still waiting for ShareLock on transaction 758 after 200.213 ms
2024-10-23 16:29:36.103 UTC [21451] postgres@postgres DETAIL:  Process holding the lock: 21334. Wait queue: 21451.
2024-10-23 16:29:36.103 UTC [21451] postgres@postgres CONTEXT:  while updating tuple (0,24) in relation "accounts"
2024-10-23 16:29:36.103 UTC [21451] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
2024-10-23 16:29:48.414 UTC [21334] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
2024-10-23 16:29:48.614 UTC [21334] postgres@postgres LOG:  process 21334 detected deadlock while waiting for ShareLock on transaction 760 after 200.134 ms
2024-10-23 16:29:48.614 UTC [21334] postgres@postgres DETAIL:  Process holding the lock: 21451. Wait queue: .
2024-10-23 16:29:48.614 UTC [21334] postgres@postgres CONTEXT:  while updating tuple (0,22) in relation "accounts"
2024-10-23 16:29:48.614 UTC [21334] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
2024-10-23 16:29:48.615 UTC [21334] postgres@postgres ERROR:  deadlock detected
2024-10-23 16:29:48.615 UTC [21334] postgres@postgres DETAIL:  Process 21334 waits for ShareLock on transaction 760; blocked by process 21451.
        Process 21451 waits for ShareLock on transaction 758; blocked by process 21334.
        Process 21334: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
        Process 21451: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
2024-10-23 16:29:48.615 UTC [21334] postgres@postgres HINT:  See server log for query details.
2024-10-23 16:29:48.615 UTC [21334] postgres@postgres CONTEXT:  while updating tuple (0,22) in relation "accounts"
2024-10-23 16:29:48.615 UTC [21334] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
2024-10-23 16:29:48.615 UTC [21451] postgres@postgres LOG:  process 21451 acquired ShareLock on transaction 758 after 12711.549 ms
2024-10-23 16:29:48.615 UTC [21451] postgres@postgres CONTEXT:  while updating tuple (0,24) in relation "accounts"
2024-10-23 16:29:48.615 UTC [21451] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
2024-10-23 16:29:48.615 UTC [21451] postgres@postgres LOG:  duration: 12711.808 ms
2024-10-23 16:30:13.865 UTC [21326] postgres@postgres LOG:  statement: UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
2024-10-23 16:30:14.066 UTC [21326] postgres@postgres LOG:  process 21326 still waiting for ShareLock on transaction 760 after 200.806 ms
2024-10-23 16:30:14.066 UTC [21326] postgres@postgres DETAIL:  Process holding the lock: 21451. Wait queue: 21326.
2024-10-23 16:30:14.066 UTC [21326] postgres@postgres CONTEXT:  while updating tuple (0,22) in relation "accounts"
2024-10-23 16:30:14.066 UTC [21326] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
2024-10-23 16:30:30.642 UTC [21130] LOG:  checkpoint starting: time
2024-10-23 16:30:30.782 UTC [21130] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.102 s, sync=0.004 s, total=0.141 s; sync files=2, longest=0.003 s, average=0.002 s; distance=2 kB, estimate=95 kB; lsn=0/155AD08, redo lsn=0/155ACC8
2024-10-23 16:35:51.126 UTC [21451] postgres@postgres LOG:  statement: COMMIT;
2024-10-23 16:35:51.127 UTC [21451] postgres@postgres LOG:  duration: 1.130 ms
2024-10-23 16:35:51.127 UTC [21326] postgres@postgres LOG:  process 21326 acquired ShareLock on transaction 760 after 337262.261 ms
2024-10-23 16:35:51.127 UTC [21326] postgres@postgres CONTEXT:  while updating tuple (0,22) in relation "accounts"
2024-10-23 16:35:51.127 UTC [21326] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100 WHERE acc_no = 3;
2024-10-23 16:35:51.127 UTC [21326] postgres@postgres LOG:  duration: 337262.692 ms
2024-10-23 16:35:57.913 UTC [21326] postgres@postgres LOG:  statement: COMMIT;
2024-10-23 16:35:57.914 UTC [21326] postgres@postgres LOG:  duration: 0.613 ms
2024-10-23 16:36:07.650 UTC [21334] postgres@postgres LOG:  statement: COMMIT;
2024-10-23 16:36:07.650 UTC [21334] postgres@postgres LOG:  duration: 0.107 ms
```

####  Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
>> Нельзя так сделать , будет **_deadlock detected_**.
