### Мои действия:
Я создала инстанс ВМ в Яндекс облако\
Зашла удаленным ssh (первая сессия) и установила PostgreSQL\
Зашла удаленным ssh второй сессией\
Выключила auto commit\
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
```
И не меняя уровень изоляции в первой сессии добавила новую запись:
```
postgres=*# insert into persons(first_name,second_name) values ('sergey','sergeev');
INSERT 0 1
```
Во второй сессии просмотрела содержание таблицы persons:
```
postgres=# select * from persons;
```
| id | first_name | second_name|
|-|--------|---|
| 8 | ivan       | ivanov|
| 9 | petr       | petrov|

:exclamation: **Я не вижу новую запись, т.к. пи уровне изоляции Read Committed видно только те данные, которые были записаны до начала запроса.**\
Далее завершила первую транзакцию\
:exclamation: **И теперь новую запись в таблице при запросе во второй сессии стало видно, т.к. транзация, которая внесла ее в таблицу была завершена**
```
postgres=*# select * from persons;
```
| id | first_name | second_name|
|-|--------|---|
| 8 | ivan       | ivanov|
| 9 | petr       | petrov|
| 10 | sergey      | sergeev|

После чего завершила транзацию во второй сессии\
Далее был изменен уровень транзации на Repeatable Read:
```
postgres=# set transaction isolation level repeatable read;
SET
```
В первой сессии была добавлена новая запись:\
```
postgres=*# insert into persons(first_name, second_name) values ('sveta','svetova');
INSERT 0 1
```
А во второй сессии вывела данные из таблицы persons:
```
postgres=# select * from persons;
```
| id | first_name | second_name|
|-|--------|---|
| 8 | ivan       | ivanov|
| 9 | petr       | petrov|
| 10 | sergey      | sergeev|

:exclamation: **Я не вижу новую запись, т.к. пи уровне изоляции Repeatable Read видно только те данные, которые были записаны до начала транзакции.**\
:exclamation: **Но при этом я вижу данные, которые внесла в таблицу, в текущей транзации, хотя еще не завершила транзацию**\
:exclamation: **После того, как я завершила транзацию в первой сессии, данные в таблице появились во второй сессии**\




