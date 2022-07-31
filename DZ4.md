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
16. Не получилось. (делаю по шпаргалке, чтобы разобраться в отличияx сxем с MS SQL)
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
31 сделайте select * from testnm.t1;
32 получилось?
33 есть идеи почему? если нет - смотрите шпаргалку
31 сделайте select * from testnm.t1;
32 получилось?
33 ура!
34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
36 есть идеи как убрать эти права? если нет - смотрите шпаргалку
37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
39 расскажите что получилось и почему 
