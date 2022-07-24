1. Cоздал виртуальную машину c Ubuntu 20.04 LTS (bionic) в YC
2. Поставил на нее PostgreSQL 14 через sudo apt
3. Проверил что кластер запущен через sudo -u postgres pg_lsclusters
- postgres@postgres:~$ sudo -u postgres pg_lsclusters
- Ver Cluster Port Status Owner    Data directory              Log file
- 14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
4. Зайшел из под пользователя postgres в psql и сделал произвольную таблицу с произвольным содержимым 
- postgres@postgres:~$ sudo -u postgres psql -p 5432
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- Type "help" for help.
- postgres=# create table test(c1 text); insert into test values('1'); \q
- CREATE TABLE
- INSERT 0 1
5. Остановил postgres через sudo -u postgres pg_ctlclusters 14 main stop
- postgres@postgres:~$ sudo -u postgres pg_ctlcluster 14 main stop
- postgres@postgres:~$ sudo -u postgres pg_lsclusters
- Ver Cluster Port Status Owner    Data directory              Log file
- 14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
6. Cоздал новый standard persistent диск pgdata размером 20GB
7. Добавил свеже-созданный диск к виртуальной машине
8. Проинициализировал диск 
- postgres@postgres:~$ df -h -x tmpfs
- Filesystem      Size  Used Avail Use% Mounted on
- udev            1.9G     0  1.9G   0% /dev
- /dev/vda2        15G  2.6G   12G  19% /
- /dev/vdb1        20G   24K   19G   1% /mnt/data
9. Сделал пользователя postgres владельцем /mnt/data -  sudo chown -R postgres:postgres /mnt/data/
10. Перенес содержимое /var/lib/postgres/14 в /mnt/data - mv /var/lib/postgresql/14 /mnt/data
11. Попытался запустить кластер - sudo -u postgres pg_ctlcluster 14 main start
- postgres@postgres:~$ sudo -u postgres pg_ctlcluster 14 main start
- Error: /var/lib/postgresql/14/main is not accessible or does not exist
12. Кластер не запустится т.к. содержимое перенесено в mnt/data
13. Найшел конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменял его
- Поменял список видимых адресов на внутреннюю подсеть для выполнения задания со *
- postgres@postgres:/etc/postgresql/14/main$ sudo nano /etc/postgresql/14/main/pg_hba.conf
- host    all             all             10.0.0.0/0            scram-sha-256
- поменял стартовую директорию кластера postgres куда переместил данные ранее
- postgres@postgres:~$ sudo nano /etc/postgresql/14/main/postgresql.conf
- data_directory = '/mnt/data/14/main'
14. Запустил кластер
-postgres@postgres:~$ sudo -u postgres pg_ctlcluster 14 main start
- Warning: the cluster will not be running as a systemd service. Consider using systemctl:
-  sudo systemctl start postgresql@14-main 
- postgres@postgres:~$ pg_lsclusters
- Ver Cluster Port Status Owner    Data directory    Log file
- 14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log
15. Зашел через через psql и проверил содержимое ранее созданной таблицы
- postgres@postgres:~$ sudo -u postgres psql -p 5432
- psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
- Type "help" for help.
- postgres=# select * from test;
- c1
- ----
-  1
- (1 row)
задание со звездочкой *: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
