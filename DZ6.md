1. Создал в YC инстанс на Ubuntu c 2 CPU 4 Gb Ram и standard disk 15GB
2. Установил на него PostgreSQL 14 с дефолтными настройками
3. Настроил сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. 
- postgres=# ALTER SYSTEM SET log_lock_waits = on;
- ALTER SYSTEM
- postgres=# ALTER SYSTEM SET deadlock_timeout TO 200;
- ALTER SYSTEM
- postgres=# select pg_reload_conf();
- pg_reload_conf
----------------
- t
- (1 row)
- postgres=# show deadlock_timeout;
- deadlock_timeout
------------------
- 200ms
- (1 row)
4. Воспроизвожу ситуацию, при которой в журнале появятся такие сообщения.

4.1. Создал таблицу
- postgres=# create table messages(id int primary key,message text);
- CREATE TABLE
- postgres=# insert into messages values (1, 'hello world');
- INSERT 0 1
- postgres=# insert into messages values (2, 'мама мыла раму');
- INSERT 0 1

4.2. 1 подключение
- sudo -u postgres psql << EOF
- BEGIN;
- SELECT message FROM messages WHERE id = 1 FOR UPDATE;
- SELECT pg_sleep(10);
- UPDATE messages SET message = 'message from session 1' WHERE id = 2;
- COMMIT;
- EOF

4.3. 2 подключение
- sudo -u postgres psql << EOF
- BEGIN;
- SELECT message FROM messages WHERE id = 2 FOR UPDATE;
- UPDATE messages SET message = 'message from session 2' WHERE id = 1;
- COMMIT;
- EOF

4.4. Результат
- sudo -u postgres psql -c "select * from messages"
- postgres=# select * from messages;
- id |        message
- ---+------------------------
-  2 | hello world
-  1 | message from session 2
-(2 rows)

4.5. Первая сессия оборвалась с ошибкой
- ERROR:  deadlock detected
- DETAIL:  Process 6441 waits for ShareLock on transaction 745; blocked by process 6445.
- Process 6445 waits for ShareLock on transaction 744; blocked by process 6441.
- HINT:  See server log for query details.
- CONTEXT:  while updating tuple (0,9) in relation "messages"
- ROLLBACK

4.6. Вторая при этом успешно завершилась

4.7.В логе при этом
- cpostgres@postgres:~$ cat /var/log/postgresql/postgresql-14-main.log | grep "deadlock detected" -A 10
- 2022-08-30 05:07:49.340 UTC [6441] postgres@postgres ERROR:  deadlock detected
- 2022-08-30 05:07:49.340 UTC [6441] postgres@postgres DETAIL:  Process 6441 waits for ShareLock on transaction 745; blocked by process 6445.
-         Process 6445 waits for ShareLock on transaction 744; blocked by process 6441.
-         Process 6441: UPDATE messages SET message = 'message from session 1' WHERE id = 2;
-         Process 6445: UPDATE messages SET message = 'message from session 2' WHERE id = 1;
- 2022-08-30 05:07:49.340 UTC [6441] postgres@postgres HINT:  See server log for query details.
- 2022-08-30 05:07:49.340 UTC [6441] postgres@postgres CONTEXT:  while updating tuple (0,9) in relation "messages"
- 2022-08-30 05:07:49.340 UTC [6441] postgres@postgres STATEMENT:  UPDATE messages SET message = 'message from session 1' WHERE id = 2;
- 2022-08-30 05:07:49.340 UTC [6445] postgres@postgres LOG:  process 6445 acquired ShareLock on transaction 744 after 8730.794 ms
- 2022-08-30 05:07:49.340 UTC [6445] postgres@postgres CONTEXT:  while updating tuple (0,10) in relation "messages"
- 2022-08-30 05:07:49.340 UTC [6445] postgres@postgres STATEMENT:  UPDATE messages SET message = 'message from session 2' WHERE id = 1;

5. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
подготовка

sudo -u postgres psql -c "drop table messages"
sudo -u postgres psql -c "create table messages(id int primary key,message text)"
sudo -u postgres psql -c "insert into messages values (1, 'hello')"
эксперимент

в каждом терминале:

BEGIN;
SELECT pg_backend_pid() as pid, txid_current() as tid;
UPDATE messages SET message = 'message from session 1' WHERE id = 1;
примечание: каждая сессия вставляет свое сообщение

сеанс	pid	tid
1	6261	530
2	6274	531
3	6284	532
блокировки

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
blocked_pid	blocked_user	blocking_pid	blocking_user	blocked_statement	current_statement_in_blocking_process	blocked_application	blocking_application
6274	postgres	6261	postgres	UPDATE messages SET message = 'message from session 2' WHERE id = 1;	UPDATE messages SET message = 'message from session 1' WHERE id = 1;	psql	psql
6284	postgres	6274	postgres	UPDATE messages SET message = 'message from session 3' WHERE id = 1;	UPDATE messages SET message = 'message from session 2' WHERE id = 1;	psql	psql
6261 блокирует 6274, а та в свою очередь - 6284

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
CASE WHEN pid = 6261 THEN 'session1' WHEN pid = 6274 THEN 'session2' WHEN pid = 6284 THEN 'session3' END AS session,

mode, 

CASE WHEN granted = true THEN 'блокировка получена' ELSE 'блокировка ожидается' END AS granted,
CASE WHEN fastpath = true THEN 'блокировка получена по короткому пути' ELSE 'блокировка получена через основную таблицу блокировок' END AS fastpath 
FROM pg_locks WHERE pid in (6261, 6274,6284) 
ORDER BY pid, virtualxid, transactionid::text::bigint;
n	locktype	relation	row	virtualxid	transactionid	virtualtransaction	pid	session	mode	granted	fastpath
1	виртуальный идентификатор			4/202		4/202	6261	session1	ExclusiveLock	блокировка получена	блокировка получена по короткому пути
2	идентификатор транзакции				530	4/202	6261	session1	ExclusiveLock	блокировка получена	блокировка получена через основную таблицу блокировок
3	отношение	messages_pkey				4/202	6261	session1	RowExclusiveLock	блокировка получена	блокировка получена по короткому пути
4	отношение	messages				4/202	6261	session1	RowExclusiveLock	блокировка получена	блокировка получена по короткому пути
5	виртуальный идентификатор			5/22		5/22	6274	session2	ExclusiveLock	блокировка получена	блокировка получена по короткому пути
6	идентификатор транзакции				530	5/22	6274	session2	ShareLock	блокировка ожидается	блокировка получена через основную таблицу блокировок
7	идентификатор транзакции				531	5/22	6274	session2	ExclusiveLock	блокировка получена	блокировка получена через основную таблицу блокировок
8	кортеж	messages	hello			5/22	6274	session2	ExclusiveLock	блокировка получена	блокировка получена через основную таблицу блокировок
9	отношение	messages_pkey				5/22	6274	session2	RowExclusiveLock	блокировка получена	блокировка получена по короткому пути
10	отношение	messages				5/22	6274	session2	RowExclusiveLock	блокировка получена	блокировка получена по короткому пути
11	виртуальный идентификатор			6/6		6/6	6284	session3	ExclusiveLock	блокировка получена	блокировка получена по короткому пути
12	идентификатор транзакции				532	6/6	6284	session3	ExclusiveLock	блокировка получена	блокировка получена через основную таблицу блокировок
13	кортеж	messages	hello			6/6	6284	session3	ExclusiveLock	блокировка ожидается	блокировка получена через основную таблицу блокировок
14	отношение	messages				6/6	6284	session3	RowExclusiveLock	блокировка получена	блокировка получена по короткому пути
15	отношение	messages_pkey				6/6	6284	session3	RowExclusiveLock	блокировка получена	блокировка получена по короткому пути
Примечание:

каждый сеанс держит эксклюзивные (exclusive lock) блокировки на номера своих транзакций (transactionid - 2, 7, 12 строки) и виртуальной транзакции (virtualxid - 1, 5, 11 - строки)
первый сеанс захватил эксклюзивную блокировку строки для ключа и самой строки, строки 3, 4
оставшиеся два запроса хоть и ожидают блокировки так же повесили row exclusive lock на ключ и строку, строки - 9, 10 и 14, 15
так же оставшиеся два сеанса повесили экслоюзивную блокировку на сам кортеж, т.к. хотят обновить именно его, а он уже обновлен в первом сеансе, строки 8 и 13
оставшаяся блокировка share lock в 6 строке вызванна тем что мы пытаемся обновить ту же строку что и в первом сеансе у которого уже захвачен row exclusive lock
Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
подготовка

sudo -u postgres psql -c "drop table messages"
sudo -u postgres psql -c "create table messages(id int primary key,message text)"
sudo -u postgres psql -c "insert into messages values (1, 'one')"
sudo -u postgres psql -c "insert into messages values (2, 'two')"
sudo -u postgres psql -c "insert into messages values (3, 'three')"
эксперимент

step	session 1 (pid: 6946, tid: 538)	session 2 (pid: 6956, tid: 539)	session 3 (pid: 6966, tid: 540)
1	begin;	begin;	begin;
2	SELECT pg_backend_pid() as pid, txid_current() as tid;	SELECT pg_backend_pid() as pid, txid_current() as tid;	SELECT pg_backend_pid() as pid, txid_current() as tid;
3	SELECT message FROM messages WHERE id = 1 FOR UPDATE;		
4		SELECT message FROM messages WHERE id = 2 FOR UPDATE;	
5			SELECT message FROM messages WHERE id = 3 FOR UPDATE;
6	UPDATE messages SET message = 'message from session 1' WHERE id = 2;		
7		UPDATE messages SET message = 'message from session 2' WHERE id = 3;	
8			UPDATE messages SET message = 'message from session 3' WHERE id = 1;
результаты

первый сеанс висит на апдейте
второй сеанс обновил строку
третий сеанс вылетел с ошибкой
ERROR:  deadlock detected
DETAIL:  Process 6966 waits for ShareLock on transaction 538; blocked by process 6946.
Process 6946 waits for ShareLock on transaction 539; blocked by process 6956.
Process 6956 waits for ShareLock on transaction 540; blocked by process 6966.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "messages"
лог

2021-05-30 08:56:27.777 UTC [6966] postgres@postgres ERROR:  deadlock detected
2021-05-30 08:56:27.777 UTC [6966] postgres@postgres DETAIL:  
        Process 6966 waits for ShareLock on transaction 538; blocked by process 6946.
        Process 6946 waits for ShareLock on transaction 539; blocked by process 6956.
        Process 6956 waits for ShareLock on transaction 540; blocked by process 6966.
        Process 6966: UPDATE messages SET message = 'message from session 3' WHERE id = 1;
        Process 6946: UPDATE messages SET message = 'message from session 1' WHERE id = 2;
        Process 6956: UPDATE messages SET message = 'message from session 2' WHERE id = 3;
2021-05-30 08:56:27.777 UTC [6966] postgres@postgres HINT:  See server log for query details.
2021-05-30 08:56:27.777 UTC [6966] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "messages"
2021-05-30 08:56:27.777 UTC [6966] postgres@postgres STATEMENT:  UPDATE messages SET message = 'message from session 3' WHERE id = 1;
В логе мы видим что процесс № 6966 (третий сеанс) споймал deadlock

Детали нам говорят о том что:

третий сенас ждал первого и второго, сеанс два при этом ждал третьего (кольцо)
далее приведены запросы, но из-за того что блокировку мы вызвали ранее по ним можно только сказать что мы пытались обновить, но не причину блокировки
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
подготовка

sudo -u postgres psql -c "drop table test"
sudo -u postgres psql -c "create table test(id integer primary key generated always as identity, n float)"
sudo -u postgres psql -c "insert into test(n) select random() from generate_series(1,1000000)"
session 1

sudo -u postgres psql << EOF
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE test SET n = (select id from test order by id asc limit 1 for update);
COMMIT;
EOF
session 2

sudo -u postgres psql << EOF
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE test SET n = (select id from test order by id desc limit 1 for update);
COMMIT;
EOF
В моем случае первый сеанс отвалился с ошибкой:

ERROR:  deadlock detected
DETAIL:  Process 8056 waits for ShareLock on transaction 608; blocked by process 8066.
Process 8066 waits for ShareLock on transaction 607; blocked by process 8056.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (15554,62) in relation "test"
ROLLBACK
Примечание: мы забираем id for update в первом сеансе отсортированные во возрастанию, а во втором по убыванию из-за чего по началу вроде как все ок, но когда два запроса "пересекаются" начинаются проблемы
