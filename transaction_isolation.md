### Мои действия:
Я создала инстанс ВМ в Яндекс облако\
Зашла удаленным ssh (первая сессия) и установила PostgreSQL\
Зашла удаленным ssh второй сессией\
Выключила auto commit:\
```
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF
```
В первой сессии создала новую таблицу:\
```
create table persons(id serial, first_name text, second_name text);
```
И наполнила ее данными:\
```
postgres=# insert into persons(first_name,second_name) values ('ivan','ivanov');
INSERT 0 1
postgres=*# insert into persons(first_name,second_name) values ('petr','petrov');
INSERT 0 1
postgres=*# commit;
COMMIT
```
Посмотрела текущий уровень изоляции: 
```
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
И не меняя уровень изоляции в первой сессии добавила новую запись:
```
postgres=*# insert into persons(first_name,second_name) values ('serg','sergeev');
INSERT 0 1
```
Во второй сессии просмотрела содержание таблицы persons:
```
postgres=# select * from persons;
| id | first_name | second_name|
|-|--------|---|
| 8 | ivan       | ivanov|
| 9 | petr       | petrov|
(2 rows)
```
Я не вижу новую запись, т.к. пи уровне изоляции Read Committed видно только те данные, которые были записаны до начала запроса.



