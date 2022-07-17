1. Создал ВМ на YC, прописал публичный SSh ключ
2. Подключился по SSH
3. Установил Docer 
  curl -fsSL https://get.docker.com -o get-docker.sh
  sudo sh get-docker.sh
  rm get-docker.sh
  sudo usermod -aG docker $USER

4. Создал docker-сеть: 
sudo docker network create pg-net

5. Подключил созданную сеть к контейнеру сервера Postgres с созданием и монтированием каталога /var/lib/postgres
sudo docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

6. Запустил отдельный контейнер с клиентом в общей сети с БД: 
sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres

7. Создал простую таблицу:
  CREATE TABLE test (i serial, amount int);
  INSERT INTO test(amount) VALUES (100);
  INSERT INTO test(amount) VALUES (500);
 
8. Вышел из контейнера \q
 
9. Проверил, что подключился через отдельный контейнер:
sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
263b9e52a44e   postgres:14   "docker-entrypoint.s…"   15 minutes ago   Up 15 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker

10. Удалил контейнер с сервером
otus@otus-vm-db-pg-net-1:~$ sudo docker stop pg-docker
pg-docker
otus@otus-vm-db-pg-net-1:~$ sudo docker rm pg-docker
pg-docker
otus@otus-vm-db-pg-net-1:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS       PORTS      NAMES

11.Создал контейнер с сервером снова
otus@otus-vm-db-pg-net-1:~$ sudo docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
2df8a387b841b41fd09a0e95c99c8f50db57363f4ff856a49fce00ce851596dd

12. Подключился снова из контейнера с клиентом к контейнеру с сервером - данные на месте
otus@otus-vm-db-pg-net-1:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
Password for user postgres:
psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

postgres=# select * from test;
 i | amount
 ---+--------
 1 |    100
 2 |    500
(2 rows)
