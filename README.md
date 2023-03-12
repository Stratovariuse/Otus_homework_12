# DML триггер

Дано:
* ТАБЛИЦА ТОВАРОВ
> CREATE TABLE goods  
(  
    goods_id    integer PRIMARY KEY,  
    good_name   varchar(63) NOT NULL,  
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)  
);  
INSERT INTO goods (goods_id, good_name, good_price)  
VALUES 	(1, 'Спички хозайственные', .50),  
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);  
    
    
* ТАБЛИЦА ПРОДАЖ  
> CREATE TABLE sales  
(  
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    good_id     integer REFERENCES goods (goods_id),   
    sales_time  timestamp with time zone DEFAULT now(),  
    sales_qty   integer CHECK (sales_qty > 0)  
);  

> INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1); 

* ОТЧЕТ  
> SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

### С увеличением объёма данных отчет стал создаваться медленно. Принято решение денормализовать БД, создать таблицу:
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);

### Создать триггер для поддержки (INSERT/UPDATE/DELETE)
* INSERT 
~~~ CREATE OR REPLACE FUNCTION tf_sales_ins()  
RETURNS trigger  
AS  
$TRIG_FUNC$  
BEGIN  
  INSERT INTO good_sum_mart (select g.good_name, sum(G.good_price * NEW.sales_qty)  
    FROM goods G  
    INNER JOIN sales S ON S.good_id = G.goods_id WHERE G.goods_id = NEW.good_id  
    GROUP BY G.good_name);  
	RETURN NEW;  
END;  
$TRIG_FUNC$  
  LANGUAGE plpgsql  
  VOLATILE  
  SET search_path = pract_functions, public;
  ~~~
  
  --триггер  
> CREATE TRIGGER tr_for_ins  
AFTER INSERT  
ON sales  
FOR EACH ROW  
EXECUTE PROCEDURE tf_sales_ins();  
  
* UPDATE  
~~~ CREATE OR REPLACE FUNCTION tf_sales_upd()  
RETURNS trigger  
AS  
$TRIG_FUNC$  
BEGIN  
  UPDATE good_sum_mart SET sum_sale = (select sum(G.good_price * NEW.sales_qty)  
    FROM goods G  
    INNER JOIN sales S ON S.good_id = G.goods_id WHERE G.goods_id = NEW.good_id  
    GROUP BY G.good_name) WHERE good_name = (select g.good_name  
    FROM goods G WHERE g.goods_id = NEW.good_id);  
	RETURN NEW;   
END;  
$TRIG_FUNC$  
  LANGUAGE plpgsql  
  VOLATILE  
  SET search_path = pract_functions, public;  
~~~
--триггер
> CREATE TRIGGER tr_for_upd  
AFTER INSERT  
ON sales  
FOR EACH ROW  
EXECUTE PROCEDURE tf_sales_upd();  

* DELETE
~~~CREATE OR REPLACE FUNCTION tf_sales_del()
RETURNS trigger
AS
$TRIG_FUNC$
BEGIN
  DELETE FROM good_sum_mart WHERE good_name = (SELECT good_name FROM goods WHERE good_id = OLD.good_id);
	RETURN OLD;
END;
$TRIG_FUNC$
  LANGUAGE plpgsql
  VOLATILE
  SET search_path = pract_functions, public;
  ~~~
-- триггер  
> CREATE TRIGGER tr_for_del  
AFTER INSERT  
ON sales  
FOR EACH ROW  
EXECUTE PROCEDURE tf_sales_del();  
