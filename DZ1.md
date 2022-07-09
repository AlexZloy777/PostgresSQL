Создал проект на Yandex Cloud
Создал ВМ c добавлением ssh ключа
Зашел удаленным ssh (первая сессия)
Поставил сразу PostgresSQL 14
Зашел ssh (вторая сессия)
Запустил pgsql под пользователем postgres (первая и вторая сессия)
выключить auto commit
сделать в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
посмотрел текущий уровень изоляции: show transaction isolation level - read commited
начал новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
в первой сессии добавил новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
сделал select * from persons во второй сессии - новой записи нет
завершил первую транзакцию - commit;
сделал select * from persons во второй сессии - появилась новая запись, т.к. в первой сессии завершена транзакция и данные записаны СУБД
завершил транзакцию во второй сессии
начал новые но уже repeatable read транзации - set transaction isolation level repeatable read; commit;
в первой сессии добавил новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
сделал select * from persons во второй сессии новой записи - нет
завершил первую транзакцию - commit;
сделал select * from persons во второй сессии - новую запись не вижу.
завершил вторую транзакцию
сделал select * from persons во второй сессии - появилась новая запись, т.к. в первой сессии завершена транзакция и данные записаны СУБД, а также завершена предидущая транзакция select.