# Journals

Виртуалка: [hw6](https://console.cloud.google.com/compute/instancesDetail/zones/europe-west3-c/instances/hw6?project=pgotus&rif_reserved)

**Настройте выполнение контрольной точки раз в 30 секунд.**

```bash
echo "checkpoint_timeout = 30s" | sudo tee -a /etc/postgresql/12/main/postgresql.conf
sudo systemctl restart postgresql
```

**10 минут c помощью утилиты pgbench подавайте нагрузку.**

```bash
sudo -u postgres psql -c "select pg_stat_reset()"
sudo -u postgres psql -c "select pg_stat_reset_shared('bgwriter')"

sudo du -h /var/lib/postgresql/12/main/pg_wal # 17mb
sudo -u postgres pgbench -i postgres
sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
```

**Измерьте, какой объем журнальных файлов был сгенерирован за это время.**

```bash
sudo du -h /var/lib/postgresql/12/main/pg_wal # 161mb
```

**Оцените, какой объем приходится в среднем на одну контрольную точку.**

Отталкиваясь от [этого](https://dataegret.com/2017/03/deep-dive-into-postgres-stats-pg_stat_bgwriter-reports/) отчета - 12mb

> Примечание: для чистоты эксперимента, пришлось сбросить статистику и прогнать эксперимент еще раз

**Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?**

При выполнении нагрузочного тестирования в постгресе ничего не будет происходить в положенном порядке

Судя по статистике среднее время выполнения контролькой точке 120 секунд

Поскольку мы заказали делать точки каждые 25 секунде - они просто напросто накладывались друг на дргуа

При такой нагрузке нет смысла так часто делать контрольные точки

**Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.**

```bash
sudo -u postgres pgbench -P 1 -T 10 postgres # 430
echo "synchronous_commit = off" | sudo tee -a /etc/postgresql/12/main/postgresql.conf
sudo systemctl restart postgresql
sudo -u postgres pgbench -P 1 -T 10 postgres # 1110
```

Разница почти в 3 раза.

Причина в более эффективном сбрасывании на диск, теперь за вместо того что бы писать на диск на каждый чих, мы делаем это отдельным  процессом, по расписанию и пачками.

**Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы.**

```bash
sudo pg_createcluster 12 demo -p 5433 -- --data-checksums
sudo pg_ctlcluster 12 demo start
sudo -u postgres psql -p 5433 -c "create table messages(message text)"
sudo -u postgres psql -p 5433 -c "insert into messages (message) values ('hello')"
sudo -u postgres psql -p 5433 -c "insert into messages (message) values ('world')"
sudo -u postgres psql -p 5433 -c "SELECT pg_relation_filepath('messages');" # base/13427/16384
sudo pg_ctlcluster 12 demo stop
sudo dd if=/dev/zero of=/var/lib/postgresql/12/demo/base/13427/16384 oflag=dsync conv=notrunc bs=1 count=8
sudo pg_ctlcluster 12 demo start
sudo -u postgres psql -p 5433 -c "select * from messages"
```

**Что и почему произошло?**

`data checksums` гарантирует целостность на уровне байтиков в файлах и ругается о том что файл был поврежден

```
WARNING:  page verification failed, calculated checksum 40176 but expected 64197
ERROR:  invalid page in block 0 of relation base/13427/16384
```

**Как проигнорировать ошибку и продолжить работу?**

Есть сразу несколько вариантов:

- в самом сеансе можно попросить postgres игнорировать ошибку и выдывать что есть
- можно найти поврежденные строки и удалить их
- можно выставить настройку зануляющую поврежденные строки и выполнить полный вакуум

```sql
SET zero_damaged_pages = on;
vacuum full messages;
select * from messages;
```

> Примечание: Во всех сценариях, если конечно мы не полезем в дампы и хэксы файла, поврежденная строка - потеряна
