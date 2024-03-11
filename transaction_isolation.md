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
В первой сессии создала новую таблицу\
```
create table persons(id serial, first_name text, second_name text);
```
И наполнила ее данными\
```
postgres=# insert into persons(first_name,second_name) values ('ivan','ivanov');
INSERT 0 1
postgres=*# insert into persons(first_name,second_name) values ('petr','petrov');
INSERT 0 1
postgres=*# commit;
COMMIT
```
