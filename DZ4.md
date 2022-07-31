1. Создал новый кластер PostgresSQL 14
- postgres@postgres:~$ pg_lsclusters
- Ver Cluster Port Status Owner    Data directory              Log file
- 14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
2. Зашел в созданный кластер под пользователем postgres
- postgres@postgres:~$ sudo -u postgres psql
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- Type "help" for help.
- 
- postgres=#
3. Создал новую базу данных testdb
- postgres=# CREATE DATABASE testdb;
- CREATE DATABASE
4. Зашел в созданную базу данных под пользователем postgres
- postgres=# \c testdb
- You are now connected to database "testdb" as user "postgres".
- testdb=#
5. Создал новую схему testnm
- testdb=# CREATE SCHEMA testnm;
- CREATE SCHEMA
6. Создал новую таблицу t1 с одной колонкой c1 типа integer
-testdb=# CREATE TABLE t1(c1 integer);
- CREATE TABLE
7. Вставил строку со значением c1=1
- testdb=# INSERT INTO t1 values(1);
- INSERT 0 1
8. Создал новую роль readonly
- testdb=# CREATE role readonly;
-CREATE ROLE
9. Дал новой роли право на подключение к базе данных testdb
- testdb=# grant connect on DATABASE testdb TO readonly;
- GRANT
10. Дал новой роли право на использование схемы testnm
- testdb=# grant usage on SCHEMA testnm to readonly;
- GRANT
11. Дал новой роли право на select для всех таблиц схемы testnm
- grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
- GRANT
12. Создал пользователя testread с паролем test123
- testdb=# CREATE USER testread with password 'test123';
- CREATE ROLE
13. Дал роль readonly пользователю testread
- testdb=# grant readonly TO testread;
- GRANT ROLE
14. Зашел под пользователем testread в базу данных testdb
- postgres@postgres:~$ psql -h 127.0.0.1 -U testread -d testdb -W
- Password:
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
- Type "help" for help.
-
- testdb=>
15. Сделал select * from t1;
- testdb=> select * from t1;
- ERROR:  permission denied for table t1
16. Не получилось. 
17. Думаю, что таблица создана в схеме по умолчанию public (т.к. явно ее не указывали) и прав на public для роли readonly не давали
18. Посмотрел на список таблиц и увидел что для testdb - {=Tc/postgres,postgres=CTc/postgres,readonly=c/postgres}
19. Вернулся в базу данных testdb под пользователем postgres
- postgres=# \c testdb postgres
- You are now connected to database "testdb" as user "postgres".
- testdb=#
20. Удалил таблицу t1
- testdb=# drop TABLE t1;
- DROP TABLE
21. Создал ее заново но уже с явным указанием имени схемы testnm
- testdb=# CREATE TABLE testnm.t1(c1 integer);
- CREATE TABLE
22. Вставил строку со значением c1=1
- testdb=# INSERT INTO testnm.t1 values(1);
- INSERT 0 1
23. Зашел под пользователем testread в базу данных testdb
-postgres@postgres:~$ psql -h 127.0.0.1 -U testread -d testdb -W
- Password:
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
- Type "help" for help.
-
- testdb=>
24. Сделал select * from testnm.t1;
- testdb=> select * from testnm.t1;
- ERROR:  permission denied for table t1
25. Не получилось
26. Потому что grant SELECT on all TABLEs in SCHEMA testnm TO readonly дал доступ только для существующих на тот момент времени таблиц а t1 пересоздавалась
27. Вернулся в базу данных testdb под пользователем postgres
- postgres=# \c testdb postgres
- You are now connected to database "testdb" as user "postgres".
- testdb=#
28. Выдал права на выборку сxеме testnm - ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly;
29. Зашел под пользователем testread в базу данных testdb
-postgres@postgres:~$ psql -h 127.0.0.1 -U testread -d testdb -W
- Password:
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
- Type "help" for help.
-
- testdb=>
30. Сделал select * from testnm.t1;
- Делаю select * from testnm.t1;
- testdb=> select * from testnm.t1;
- c1
----
-  1
- (1 row)
31. Потому что ALTER default будет действовать для новых таблиц а grant SELECT on all TABLEs in SCHEMA testnm TO readonly отработал только для существующих на тот момент времени. Надо сделать снова или grant SELECT или пересоздать таблицу
32. Пробую выполнить команду create table t2(c1 integer); insert into t2 values (2);
- testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
33. Получилось это все потому что search_path указывает в первую очередь на схему public. А схема public создается в каждой базе данных по умолчанию. И grant на все действия в этой схеме дается роли public. А роль public добавляется всем новым пользователям. Соответсвенно каждый пользователь может по умолчанию создавать объекты в схеме public любой базы данных, ес-но если у него есть право на подключение к этой базе данных.
34. Наверно нужно забрать права на создание и редактирование из сxемы public для базы testdb. (смотрю шпаргалку)
35. Зашел в базу под postgres 
- postgres@postgres:~$ psql -U postgres
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- Type "help" for help.

- postgres=# \c testdb
- testdb=#
36. Убрал права на создание 
- testdb=# revoke CREATE on SCHEMA public FROM public;
- REVOKE
37. Убрал все предоставленные права в testdb из cxемы public
- testdb=# revoke all on DATABASE testdb FROM public;
- REVOKE
38. Зашел в testdb под testread 
- psql -h 127.0.0.1 -U testread -d testdb -W
- Password:
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
- Type "help" for help.
-
- testdb=>
39. Пробую выполнить команду create table t3(c1 integer); insert into t2 values (2);
- testdb=> create table t3(c1 integer); insert into t2 values (2);
- ERROR:  permission denied for schema public
- LINE 1: create table t3(c1 integer);
- INSERT 0 1
40. Таблица не была создана (права на создание забрали), а запись в таблицу t2 была добавлена (права на вставку не забирали)
- testdb=> select * from t2;
- c1
----
-  2
-  2
(2 rows)
