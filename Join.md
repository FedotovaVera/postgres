Я создала таблицы:
```
CREATE TABLE sweets_types
(
    id SERIAL PRIMARY KEY,
    name varchar(100) NOT NULL
);

CREATE TABLE sweets
(
    id SERIAL PRIMARY KEY,
    sweets_types_id INT,
    name varchar(100) NOT NULL,
    cost varchar(100) NOT NULL,
    weight varchar(100) NOT NULL,
    manufacturer_id INT NOT NULL,
    with_sugar INT,
    requires_freezing INT,
    production_date date NOT NULL,
    expiration_date date NOT NULL
);

CREATE TABLE manufacturers_storehouses
(
    id SERIAL PRIMARY KEY,
    storehouses_id INT NOT NULL,
    manufacturers_id INT NOT NULL
);


CREATE TABLE manufacturers
(
    id SERIAL PRIMARY KEY,
    name varchar(100) NOT NULL,
	phone varchar(100),
	adress varchar(100),
	city varchar(100) NOT NULL,
	country varchar(100) NOT NULL
);

CREATE TABLE storehouses
(
    id SERIAL PRIMARY KEY,
    name varchar(100) NOT NULL,
    adress varchar(100),
    city varchar(100) NOT NULL,
    country varchar(100) NOT NULL
);
```
Заполнила данными:
```
INSERT INTO sweets_types(
	name)
	VALUES
	('вафли'),
	('конфеты'),
	('мармелад'),
	('печенье'),
	('шоколад');

INSERT INTO storehouses(
	name, adress, city, country)
	VALUES 
	('MSK-1', '109235, г. Москва, Проектируемый проезд 4386, д.8', 'Moscow', 'Russia'),
	('SPB-1', '197375, г. Санкт-Петербург, Суздальское шоссе, д. 26', 'Saint-petersburg', 'Russia'),
	('EKB-1', '620137, г. Екатеринбург, Шефская улица, д. 1А', 'Ekaterinburg', 'Russia'),
	('EKB-2', '620137, г. Екатеринбург, Шефская улица, д. 2А', 'Ekaterinburg', 'Russia');
	
INSERT INTO manufacturers(
	name, phone, adress, city, country)
	VALUES 
	('Мишаня', '', '109235, г. Москва, Проектируемый проезд, д.15', 'Moscow', 'Russia'),
	('Собакен', '78125748899', '197375, г. Санкт-Петербург, Суздальское шоссе, д. 75', 'Saint-petersburg', 'Russia'),
	('Мартыха', '74657896525', '620137, г. Екатеринбург, Шефская улица, д. 5А', 'Ekaterinburg', 'Russia'),
	('Мишаня', '', '109235, г. Казань, Проектируемая улица, д.15', 'Kazan', 'Russia');
	
INSERT INTO manufacturers_storehouses(
	storehouses_id, manufacturers_id)
	VALUES 
	(1, 1),
	(2, 2),
	(3, 3),
	(1, 2),
	(2, 1);
	
INSERT INTO sweets(
	sweets_types_id, name, cost, weight, manufacturer_id, with_sugar, requires_freezing, production_date, expiration_date)
	VALUES 
	(1, 'Мильтик', '100', '200',1, 0, 0, '2022-05-03', '2022-05-15'),
	(2, 'Микус', '150', '300', 1 , 1, 1, '2022-04-03', '2022-05-03'),
	(3, 'Миви', '110', '100', 1 , 1, 0, '2022-03-03', '2022-04-14'),
	(4, 'Ми', '120', '200', 1, 0, 1, '2022-03-04', '2022-04-04'),
	(5, 'Миса', '145', '570', 1, 1, 0, '2021-03-03', '2021-12-03'),
	(1, 'Сольтик', '115', '200', 2 , 0, 0, '2022-05-03', '2022-05-15'),
	(2, 'Сокус', '155', '300', 2 , 1, 1, '2022-03-03', '2022-05-03'),
	(3, 'Сови', '117', '500', 2 , 1, 0, '2022-03-03', '2022-04-14'),
	(4, 'Со', '129', '250', 2, 0, 1, '2022-03-04', '2022-04-04'),
	(5, 'Сор', '148', '500', 2, 1, 0, '2021-02-03', '2021-12-03'),
	(1, 'Мальтик', '210', '200', 3 , 0, 0, '2022-05-03', '2022-05-15'),
	(2, 'Макус', '350', '300', 3 , 1, 1, '2022-01-03', '2022-05-03');
```
****Реализовать прямое соединение двух или более таблиц****
```
select --в результате выборки будут только те записи, айди сладостей которых есть и в левой и правой таблицах
	a.name,
	b.name
from
sweets_types a
join sweets b 
	on a.id = b.sweets_types_id
```
****Реализовать левостороннее (или правостороннее)
соединение двух или более таблиц****
```
select --в результате выборки будут все записи из левой таблицы, даже если нет совпадений с правой таблицей. (где нет совпадений, в колонках правой таблицы будет null)
	a.name,
	b.name
from
sweets_types a
left join sweets b 
	on a.id = b.sweets_types_id
```
****Реализовать кросс соединение двух или более таблиц****
```
select --в результате выборки всем строкам левой таблицы будет сопоставлена каждая строка из правой таблицы.
       --условно, если в левой таблице 3 записи и в правой таблице 3 записи, то в результате выборки будет 9 строк
	a.name,
	b.name
from
sweets_types a
cross join sweets b 
	on a.id = b.sweets_types_id
```
****Реализовать полное соединение двух или более таблиц****
```
select -- в результате выборки будут строки из левой таблицы и из правой таблицы. Даже если у них не выполняется условие объединения (там будет null).
      -- То есть будут выведены строки, где есть совпадение, потом строки из левой таблицы (в правой будет null), а затем наоборот.
	a.name,
	b.name
from
sweets_types a
full join sweets b 
	on a.id = b.sweets_types_id
```
****Реализовать запрос, в котором будут использованы
разные типы соединений****
```
select 
	a.name,
	b.name,
	c.*
from
sweets_types a
full join sweets b
	on a.id = b.sweets_types_id
join manufacturers c
	on b.manufacturer_id = c.id
```
