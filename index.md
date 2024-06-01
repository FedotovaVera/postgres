***Создать индекс к какой-либо из таблиц вашей БД***
***Прислать текстом результат команды explain, в которой используется данный индекс***
Создаю таблицу:
```
CREATE TABLE test (
      IntColumn INT,
      TextColumn VARCHAR(100)
   );

INSERT INTO test (IntColumn, TextColumn)
SELECT
  generate_series(1,10000000) AS IntColumn,
  format('ИмяПользователя%s', generate_series(1,10000000)::text ) AS TextColumn;
```
![image](https://github.com/FedotovaVera/postgres/assets/111522773/cb26fda8-b90d-43e1-a109-276f71c8c589)
Смотрю explain до:
```
"Seq Scan on test  (cost=0.00..1482122.40 rows=35930240 width=222)"
```
Добавляю индекс и Смотрю explain после:
```
drop index if exists ix_test_intcolumn;
create index ix_test_intcolumn on test(intcolumn);
analyze test;

EXPLAIN SELECT * FROM test;
"Seq Scan on test  (cost=0.00..1122820.00 rows=1 width=222)"
EXPLAIN SELECT * FROM test where intcolumn < 100;
"Index Scan using ix_test_intcolumn on test  (cost=0.12..8.14 rows=1 width=222)"
"  Index Cond: (intcolumn < 100)"
```
Запрос по индексированной колонке выполняется в разы быстрее, да и не по колонке тоже
***Реализовать индекс для полнотекстового поиска***
```
set default_text_search_config = russian;

create index ix_test_TextColumn on test using gin (to_tsvector('russian', TextColumn));
analyze test;
explain select TextColumn from test where to_tsvector('russian', TextColumn) @@ to_tsquery('ИмяПользователя500');
```
```
"Bitmap Heap Scan on test  (cost=8.25..12.76 rows=1 width=218)"
"  Recheck Cond: (to_tsvector('russian'::regconfig, (textcolumn)::text) @@ to_tsquery('ИмяПользователя500'::text))"
"  ->  Bitmap Index Scan on ix_test_textcolumn  (cost=0.00..8.25 rows=1 width=0)"
"        Index Cond: (to_tsvector('russian'::regconfig, (textcolumn)::text) @@ to_tsquery('ИмяПользователя500'::text))"
```
***Реализовать индекс на часть таблицы или индекс на поле с функцией***
```
drop index if exists ix_test_IntColumn;
create index ix_test_IntColumn on test(IntColumn) where IntColumn < 50;
analyze test;

explain SELECT * FROM test where IntColumn < 40;
"Index Scan using ix_test_intcolumn on test  (cost=0.12..8.14 rows=1 width=222)"
"  Index Cond: (intcolumn < 40)"
```
```
explain SELECT * FROM test where IntColumn > 55 and IntColumn < 95;
"Seq Scan on test  (cost=0.00..1122820.00 rows=1 width=222)"
"  Filter: ((intcolumn > 55) AND (intcolumn < 95))"
```
Быстро выполняется запрос по проиндексированным строкам и медленно по неиндексированным - более затратная операция
***Создать индекс на несколько полей***
Удалю существующие индексы
```
drop index if exists ix_test_intcolumn;
drop index if exists ix_test_TextColumn;
create index ix_test_intcolumn_TextColumn on test (IntColumn, IntColumn);
```
И выполним тот же самый запрос и увидим, что затраты существенно снизились
```
explain SELECT * FROM test where IntColumn > 55 and IntColumn < 95;
"Index Scan using ix_test_intcolumn_textcolumn on test  (cost=0.12..8.14 rows=1 width=222)"
"  Index Cond: ((intcolumn > 55) AND (intcolumn < 95))"
```
