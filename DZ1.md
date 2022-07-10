1. Создал проект на Yandex Cloud
2. Создал ВМ c добавлением ssh ключа
3. Зашел удаленным ssh (первая сессия)
4. Поставил сразу PostgresSQL 14
5. Зашел ssh (вторая сессия)
6. Запустил pgsql под пользователем postgres (первая и вторая сессия)
7. Выключил auto commit
8. Сделал в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
9. Посмотрел текущий уровень изоляции: show transaction isolation level - read commited
10. Начал новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
11. В первой сессии добавил новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
12. Сделал select * from persons во второй сессии - новой записи нет
13. Завершил первую транзакцию - commit;
14. Сделал select * from persons во второй сессии - появилась новая запись, т.к. Уровень изоляции транзакции позволяет, а в первой сессии завершена транзакция и данные записаны СУБД.  
15. Завершил транзакцию во второй сессии
16. Начал новые, но уже repeatable read транзации - set transaction isolation level repeatable read;
17. В первой сессии добавил новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
18. Сделал select * from persons во второй сессии новой записи - нет
19. Завершил первую транзакцию - commit;
20. Сделал select * from persons во второй сессии - новую запись не вижу.
21. Завершил вторую транзакцию
22. Сделал select * from persons во второй сессии - появилась новая запись, т.к. Уровень изоляции транзакции repeatable read, в первой сессии завершена транзакция и данные записаны СУБД, а также завершена предидущая транзакция select.
