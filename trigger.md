-- Создать триггер (на таблице sales) для поддержки.
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

-- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
-- Подсказка: В реальной жизни возможны изменения цен.
Смотрю что в таблице
```
select * from good_sum_mart;
```
Пусто же, надо заполнить
```
INSERT INTO good_sum_mart 
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

select * from good_sum_mart;
```
Создаю функцию для инсерта и апдейта
```
CREATE or replace function cm_fn_insert_into_good_sum_mart()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
new_price numeric(20,2);
new_name varchar(63);
BEGIN

SELECT G.good_name, sum(G.good_price * S.sales_qty)
INTO new_name, new_price 
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id and G.goods_id = NEW.good_id
GROUP BY G.good_name; 

IF EXISTS(select from good_sum_mart T where T.good_name = new_name)
	THEN 
		UPDATE good_sum_mart T SET sum_sale = new_price where T.good_name = new_name;
	ELSE 
		INSERT INTO good_sum_mart (good_name, sum_sale) values(new_name, new_price);
END IF;
RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql
VOLATILE
SET search_path = pract_functions, public
COST 50;
```
А теперь для удаления
```
CREATE or replace function cm_fn_insert_into_good_sum_mart_delete()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
new_price numeric(20,2);
new_name varchar(63);
BEGIN

SELECT G.good_name, G.good_price*OLD.sales_qty 
INTO new_name, new_price 
FROM goods G where G.goods_id = OLD.good_id;

IF EXISTS(select 1 from good_sum_mart T where T.good_name = new_name)
	THEN 
		UPDATE good_sum_mart T SET sum_sale = sum_sale - new_price where T.good_name = new_name;
		DELETE FROM good_sum_mart T where T.good_name = new_name and (sum_sale <= 0);
END IF;
RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql
VOLATILE
SET search_path = pract_functions, public
COST 50;
```
Создаю триггеры
```
CREATE TRIGGER tr_insert_sales
AFTER INSERT
ON sales
FOR EACH ROW
EXECUTE PROCEDURE cm_fn_insert_into_good_sum_mart();

CREATE TRIGGER tr_update_sales
AFTER UPDATE
ON sales
FOR EACH ROW
EXECUTE PROCEDURE cm_fn_insert_into_good_sum_mart();

CREATE TRIGGER tr_delete_sales
AFTER DELETE
ON sales
FOR EACH ROW
EXECUTE PROCEDURE cm_fn_insert_into_good_sum_mart_delete();
```
Далее играюсь с данными что бы проверить:
```
INSERT INTO sales (good_id, sales_qty) VALUES (1, 5), (1, 1), (1, 12), (2, 1);

select * from --truncate table 
good_sum_mart;

select * from --truncate table --1682
sales where good_id = 2;

DELETE 
FROM sales 
WHERE good_id = 2;

UPDATE sales 
SET sales_qty = 10
where good_id = 2;
```
Все работает правильно


