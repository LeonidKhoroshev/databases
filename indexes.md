# Домашнее задание к занятию «Индексы» - Леонид Хорошев


### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

Для решения данной задачи необходимо найти размер всех индексов и таблиц, поэтому сначала смотрим, какие данные у нас есть:

```sql
describe information_schema.tables;
```
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/indexes/index1.1.png)

Из данного вывода нам необходимы  DATA_LENGTH (размер данных в базе данных) и INDEX_LENGTH (размер индексов в базе данных).

```sql
select table_schema, round(sum(index_length)*100/sum(index_length+data_length)) as '% memory'
from information_schema.tables
where table_schema = 'sakila';
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/indexes/index1.2.png)

Результат - индексы занимают 35% от общего объема хранимой информации в базе данных "Sakila".

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

Вывод анализа запроса:

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/indexes/index2.1.png)

Из него не особо понятно, что не так, выполним сам запрос:

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/indexes/index2.2.png)

Из таблицы видно, что в выводе запроса у нас участвуют данные из таблиц customer, payment и film не участвуют данные из inventory и rental, хотя все они участвуют в запросе. Попробуем их от туда убрать.

```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, customer c, film f
where date(p.payment_date) = '2005-07-30' and p.customer_id = c.customer_id;
```

Собственно вывод не изменился:

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/indexes/index2.3.png);

Сделаем повторный анализ запроса:

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/indexes/index2.4.png);




### Задание 3*

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

*Приведите ответ в свободной форме.*
