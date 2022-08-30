1. Создал в YC инстанс на Ubuntu c 2 CPU 4 Gb Ram и standard disk 15GB
2. Установил на него PostgreSQL 14 с дефолтными настройками
3. Настроил сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. 
```bash
postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET deadlock_timeout TO 200;
ALTER SYSTEM
postgres=# select pg_reload_conf();
```
|pg_reload_conf  |
|----------------|
| t              |
| (1 row)        |
```bash
postgres=# show deadlock_timeout;
```
| deadlock_timeout |
|------------------|
| 200ms            |
| (1 row)          |
4. Воспроизвожу ситуацию, при которой в журнале появятся такие сообщения.

4.1. Создал таблицу
```bash
postgres=# create table messages(id int primary key,message text);
CREATE TABLE
postgres=# insert into messages values (1, 'hello world');
INSERT 0 1
postgres=# insert into messages values (2, 'мама мыла раму');
INSERT 0 1
```
4.2. подключение 1
```sql
sudo -u postgres psql << EOF
BEGIN;
SELECT message FROM messages WHERE id = 1 FOR UPDATE;
SELECT pg_sleep(10);
UPDATE messages SET message = 'message from session 1' WHERE id = 2;
COMMIT;
EOF
```
4.3. подключение 2
```sql
sudo -u postgres psql << EOF
BEGIN;
SELECT message FROM messages WHERE id = 2 FOR UPDATE;
UPDATE messages SET message = 'message from session 2' WHERE id = 1;
COMMIT;
EOF
```
4.4. Результат
```bash
sudo -u postgres psql -c "select * from messages"
postgres=# select * from messages;
```
| id |        message         |
| ---|------------------------|
|  2 | hello world            |
|  1 | message from session 2 |
| (2 rows)                    |

4.5. Первая сессия оборвалась с ошибкой
```log 
ERROR:  deadlock detected
DETAIL:  Process 6441 waits for ShareLock on transaction 745; blocked by process 6445.
Process 6445 waits for ShareLock on transaction 744; blocked by process 6441.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,9) in relation "messages"
ROLLBACK
```
4.6. Вторая при этом успешно завершилась

4.7. В логе при этом
```log
cpostgres@postgres:~$ cat /var/log/postgresql/postgresql-14-main.log | grep "deadlock detected" -A 10
2022-08-30 05:07:49.340 UTC [6441] postgres@postgres ERROR:  deadlock detected
2022-08-30 05:07:49.340 UTC [6441] postgres@postgres DETAIL:  Process 6441 waits for ShareLock on transaction 745; blocked by process 6445.
         Process 6445 waits for ShareLock on transaction 744; blocked by process 6441.
         Process 6441: UPDATE messages SET message = 'message from session 1' WHERE id = 2;
         Process 6445: UPDATE messages SET message = 'message from session 2' WHERE id = 1;
2022-08-30 05:07:49.340 UTC [6441] postgres@postgres HINT:  See server log for query details.
2022-08-30 05:07:49.340 UTC [6441] postgres@postgres CONTEXT:  while updating tuple (0,9) in relation "messages"
2022-08-30 05:07:49.340 UTC [6441] postgres@postgres STATEMENT:  UPDATE messages SET message = 'message from session 1' WHERE id = 2;
2022-08-30 05:07:49.340 UTC [6445] postgres@postgres LOG:  process 6445 acquired ShareLock on transaction 744 after 8730.794 ms
2022-08-30 05:07:49.340 UTC [6445] postgres@postgres CONTEXT:  while updating tuple (0,10) in relation "messages"
2022-08-30 05:07:49.340 UTC [6445] postgres@postgres STATEMENT:  UPDATE messages SET message = 'message from session 2' WHERE id = 1;
```
5. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

5.1. Удаляю старую таблицу
```bash
postgres@postgres:~$ sudo -u postgres psql -c "drop table messages"
DROP TABLE
```
5.2. Создаю новую таблицу с главным ключем
```bash
postgres@postgres:~$ sudo -u postgres psql -c "create table messages(id int primary key,message text)"
CREATE TABLE
```
5.3. Вставляю строку, которую буду менять
```bash
postgres@postgres:~$ sudo -u postgres psql -c "insert into messages values (1, 'hello')"
- INSERT 0 1
```
5.4. подключение 1
```sql
BEGIN;
SELECT pg_backend_pid() as pid, txid_current() as tid;
UPDATE messages SET message = 'message from session 1' WHERE id = 1;
```
5.5. подключение 1
```sql
BEGIN;
SELECT pg_backend_pid() as pid, txid_current() as tid;
UPDATE messages SET message = 'message from session 2' WHERE id = 1;
```
5.6. подключение 3
```sql
BEGIN;
SELECT pg_backend_pid() as pid, txid_current() as tid;
UPDATE messages SET message = 'message from session 3' WHERE id = 1;
```
5.7. Сочетание заблокированной и блокирующей активности
| сеанс | pid  | tid |
| ----- | ---- | --- |
|     1 | 6531 | 757 |
|     2 | 6707 | 758 |
|     3 | 6711 | 759 |

**[блокировки] (https://wiki.postgresql.org/wiki/Lock_Monitoring)**

```sql

SELECT blocked_locks.pid     AS blocked_pid,
         blocked_activity.usename  AS blocked_user,
         blocking_locks.pid     AS blocking_pid,
         blocking_activity.usename AS blocking_user,
         blocked_activity.query    AS blocked_statement,
         blocking_activity.query   AS current_statement_in_blocking_process,
         blocked_activity.application_name AS blocked_application,
         blocking_activity.application_name AS blocking_application
   FROM  pg_catalog.pg_locks         blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks         blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid
 
    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
   WHERE NOT blocked_locks.GRANTED;
```

| blocked_pid | blocked_user | blocking_pid | blocking_user |                          blocked_statement                           |                current_statement_in_blocking_process                 | blocked_application | blocking_application |
| ----------- | ------------ | ------------ | ------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------- | -------------------- |
|        6707 | postgres     |         6531 | postgres      | UPDATE messages SET message = 'message from session 2' WHERE id = 1; | UPDATE messages SET message = 'message from session 1' WHERE id = 1; | psql                | psql                 |
|        6711 | postgres     |         6707 | postgres      | UPDATE messages SET message = 'message from session 3' WHERE id = 1; | UPDATE messages SET message = 'message from session 2' WHERE id = 1; | psql                | psql                 |

из таблицы понятно, что `6531` блокирует `6707`, а та в свою очередь - `6711`

**Альтернативный вид тех же данных, который включает имя_приложения

```sql
SELECT 
row_number() over(ORDER BY pid, virtualxid, transactionid::text::bigint) as n,
CASE
WHEN locktype = 'relation' THEN 'отношение'
WHEN locktype = 'extend' THEN 'расширение отношения'
WHEN locktype = 'frozenid' THEN 'замороженный идентификатор'
WHEN locktype = 'page' THEN 'страница'
WHEN locktype = 'tuple' THEN 'кортеж'
WHEN locktype = 'transactionid' THEN 'идентификатор транзакции'
WHEN locktype = 'virtualxid' THEN 'виртуальный идентификатор'
WHEN locktype = 'object' THEN 'объект'
WHEN locktype = 'userlock' THEN 'пользовательская блокировка'
WHEN locktype = 'advisory' THEN 'рекомендательная'
END AS locktype,

relation::regclass,
-- CASE WHEN relation IS NULL THEN 'цель блокировки — не отношение или часть отношения' ELSE CAST(relation::regclass AS TEXT) END AS relation,

CASE WHEN page IS NOT NULL AND tuple IS NOT NULL THEN (select message from messages m where m.ctid::text = '(' || page || ',' || tuple || ')' limit 1) ELSE NULL END AS row, -- page, tuple, 

virtualxid, transactionid, virtualtransaction, 

pid, 
CASE WHEN pid = 6531 THEN 'session1' WHEN pid = 6707 THEN 'session2' WHEN pid = 6711 THEN 'session3' END AS session,

mode, 

CASE WHEN granted = true THEN 'блокировка получена' ELSE 'блокировка ожидается' END AS granted,
CASE WHEN fastpath = true THEN 'блокировка получена по короткому пути' ELSE 'блокировка получена через основную таблицу блокировок' END AS fastpath 
FROM pg_locks WHERE pid in (6531, 6707, 6711) 
ORDER BY pid, virtualxid, transactionid::text::bigint;
```

| n  |         locktype          |   relation    |          row           | virtualxid | transactionid | virtualtransaction | pid  | session  |       mode       |       granted        |                       fastpath                        |
|----|---------------------------|---------------|------------------------|------------|---------------|--------------------|------|----------|------------------|----------------------|-------------------------------------------------------|
|  1 | виртуальный идентификатор |               |                        | 3/97       |               | 3/97               | 6531 | session1 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
|  2 | идентификатор транзакции  |               |                        |            |           757 | 3/97               | 6531 | session1 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
|  3 | отношение                 | messages      |                        |            |               | 3/97               | 6531 | session1 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
|  4 | отношение                 | messages_pkey |                        |            |               | 3/97               | 6531 | session1 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
|  5 | виртуальный идентификатор |               |                        | 4/376      |               | 4/376              | 6707 | session2 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
|  6 | идентификатор транзакции  |               |                        |            |           757 | 4/376              | 6707 | session2 | ShareLock        | блокировка ожидается | блокировка получена через основную таблицу блокировок |
|  7 | идентификатор транзакции  |               |                        |            |           758 | 4/376              | 6707 | session2 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
|  8 | отношение                 | messages      |                        |            |               | 4/376              | 6707 | session2 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
|  9 | отношение                 | messages_pkey |                        |            |               | 4/376              | 6707 | session2 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 10 | кортеж                    | messages      | message from session 1 |            |               | 4/376              | 6707 | session2 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 11 | виртуальный идентификатор |               |                        | 5/5        |               | 5/5                | 6711 | session3 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
| 12 | идентификатор транзакции  |               |                        |            |           759 | 5/5                | 6711 | session3 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 13 | кортеж                    | messages      | message from session 1 |            |               | 5/5                | 6711 | session3 | ExclusiveLock    | блокировка ожидается | блокировка получена через основную таблицу блокировок |
| 14 | отношение                 | messages      |                        |            |               | 5/5                | 6711 | session3 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 15 | отношение                 | messages_pkey |                        |            |               | 5/5                | 6711 | session3 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |

Примечание из этой таблицы:
- каждый сеанс держит эксклюзивные (exclusive lock) блокировки на номера своих транзакций (transactionid - 2, 7, 12 строки) и виртуальной транзакции (virtualxid - 1, 5, 11 - строки)
- первый сеанс захватил эксклюзивную блокировку строки для ключа и самой строки, строки 3, 4
- оставшиеся два запроса хоть и ожидают блокировки так же повесили row exclusive lock на ключ и строку, строки - 9, 10 и 14, 15
- так же оставшиеся два сеанса повесили экслоюзивную блокировку на сам кортеж, т.к. хотят обновить именно его, а он уже обновлен в первом сеансе, строки 8 и 13
- оставшаяся блокировка share lock в 6 строке вызванна тем что мы пытаемся обновить ту же строку что и в первом сеансе у которого уже захвачен row exclusive lock


6. Воспроизвожу взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
6.1. Удаляю предидущую таблицу
```bash
sudo -u postgres psql -c "drop table messages"
DROP TABLE
```
6.2. Создаю новую таблицу
```bash
sudo -u postgres psql -c "create table messages(id int primary key,message text)"
CREATE TABLECREATE TABLE
sudo -u postgres psql -c "insert into messages values (1, 'one')"
INSERT 0 1
sudo -u postgres psql -c "insert into messages values (2, 'two')"
INSERT 0 1
sudo -u postgres psql -c "insert into messages values (3, 'three')"
INSERT 0 1
```
6.3. Таблица с шагами эксперимента

| step | session 1 (pid: 6531, tid: 769)                                        | session 2 (pid: 6707, tid: 770)                                        | session 3 (pid: 6711, tid: 771)                                        |
| ---- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 1    | `begin;`                                                               | `begin;`                                                               | `begin;`                                                               |
| 2    | `SELECT pg_backend_pid() as pid, txid_current() as tid;`               | `SELECT pg_backend_pid() as pid, txid_current() as tid;`               | `SELECT pg_backend_pid() as pid, txid_current() as tid;`               |
| 3    | `SELECT message FROM messages WHERE id = 1 FOR UPDATE;`                |                                                                        |                                                                        |
| 4    |                                                                        | `SELECT message FROM messages WHERE id = 2 FOR UPDATE;`                |                                                                        |
| 5    |                                                                        |                                                                        | `SELECT message FROM messages WHERE id = 3 FOR UPDATE;`                |
| 6    | `UPDATE messages SET message = 'message from session 1' WHERE id = 2;` |                                                                        |                                                                        |
| 7    |                                                                        | `UPDATE messages SET message = 'message from session 2' WHERE id = 3;` |                                                                        |
| 8    |                                                                        |                                                                        | `UPDATE messages SET message = 'message from session 3' WHERE id = 1;` |

6.4. Результаты эксперимента
- первый сеанс висит на апдейте
- второй сеанс обновил строку
- третий сеанс вылетел с ошибкой
```log
ERROR:  deadlock detected
        DETAIL:  Process 6711 waits for ShareLock on transaction 769; blocked by process 7361.
        Process 7361 waits for ShareLock on transaction 770; blocked by process 6707.
        Process 6707 waits for ShareLock on transaction 771; blocked by process 6711.
        HINT:  See server log for query details.
        CONTEXT:  while updating tuple (0,5) in relation "messages"
```
6.5. Смотрим лог
```log
2022-08-30 07:54:45.767 UTC [6711] postgres@postgres ERROR:  deadlock detected
2022-08-30 07:54:45.767 UTC [6711] postgres@postgres DETAIL:  Process 6711 waits for ShareLock on transaction 769; blocked by process 7361.
        Process 7361 waits for ShareLock on transaction 770; blocked by process 6707.
        Process 6707 waits for ShareLock on transaction 771; blocked by process 6711.
        Process 6711: UPDATE messages SET message = 'message from session 3' WHERE id = 1;
        Process 7361: UPDATE messages SET message = 'message from session 1' WHERE id = 2;
        Process 6707: UPDATE messages SET message = 'message from session 2' WHERE id = 3;
2022-08-30 07:54:45.767 UTC [6711] postgres@postgres HINT:  See server log for query details.
2022-08-30 07:54:45.767 UTC [6711] postgres@postgres CONTEXT:  while updating tuple (0,5) in relation "messages"
2022-08-30 07:54:45.767 UTC [6711] postgres@postgres STATEMENT:  UPDATE messages SET message = 'message from session 3' WHERE id = 1;
2022-08-30 07:54:45.767 UTC [6707] postgres@postgres LOG:  process 6707 acquired ShareLock on transaction 771 after 14097.980 ms
```
- В логе мы видим что процесс № 6711 (третий сеанс) поймал deadlock
Детали нам говорят о том что:
- третий сенас ждал первого и второго, сеанс два при этом ждал третьего (кольцо)
- далее приведены запросы, но из-за того что блокировку мы вызвали ранее по ним можно только сказать что мы пытались обновить, но не причину блокировки

7. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
7.1. Удаляю старую таблицу
```bash
sudo -u postgres psql -c "drop table messages"
```
7.2. Создаю таблицу для эксперимента
```bash
sudo -u postgres psql -c "create table test(id integer primary key generated always as identity, n float)"
CREATE TABLE
sudo -u postgres psql -c "insert into test(n) select random() from generate_series(1,1000000)"
INSERT 0 1000000
```
7.3. Подключение 1
```sql
sudo -u postgres psql << EOF
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE test SET n = (select id from test order by id asc limit 1 for update);
COMMIT;
EOF
```
7.4. Подключение 2
```sql
sudo -u postgres psql << EOF
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE test SET n = (select id from test order by id desc limit 1 for update);
COMMIT;
EOF
```
7.5. В моем случае первый сеанс отвалился с ошибкой:
```log
ERROR:  deadlock detected
DETAIL:  Process 7533 waits for ShareLock on transaction 777; blocked by process 7537.
Process 7537 waits for ShareLock on transaction 776; blocked by process 7533.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (5405,75) in relation "test"
ROLLBACK
```
7.6. Примечание: мы забираем id for update в первом сеансе отсортированные во возрастанию, а во втором по убыванию из-за чего по началу вроде как все ок, но когда два запроса "пересекаются" начинаются проблемы
