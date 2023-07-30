# Домашнее задание к занятию «SQL. Часть 1» - Леонид Хорошев

### Задание 1

Получите уникальные названия районов из таблицы с адресами, которые начинаются на “K” и заканчиваются на “a” и не содержат пробелов.

Подключаемся к нашей базе данных sakila, смотрим из чего состоит таблица с адресами и выбираем нужные данные.

```sql
mysql -u root -p
use sakila
describe address;
mysql> select distinct district
from address
where district like 'K%a' and district not like '% %';
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL1.1.png)

### Задание 2

Получите из таблицы платежей за прокат фильмов информацию по платежам, которые выполнялись в промежуток с 15 июня 2005 года по 18 июня 2005 года **включительно** и стоимость которых превышает 10.00.

```sql
describe payment;
select * from payment
where payment_date between '2005.06.15' and '2005.06.18'
and amount > 10;
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL1.2.png)

### Задание 3

Получите последние пять аренд фильмов.

```sql
describe rental;
select * from rental
order by rental_date desc
limit 5;
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL1.3.png)



### Задание 4

Одним запросом получите активных покупателей, имена которых Kelly или Willie. 

Сформируйте вывод в результат таким образом:
- все буквы в фамилии и имени из верхнего регистра переведите в нижний регистр,
- замените буквы 'll' в именах на 'pp'.

```sql
describe customer
select lower(replace(first_name, 'LL', 'pp')), lower(last_name)
from customer
where first_name like 'Kelly' or 'Willie'
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL1.4.png)
  
### Задание 5*

Выведите Email каждого покупателя, разделив значение Email на две отдельных колонки: в первой колонке должно быть значение, указанное до @, во второй — значение, указанное после @.

```sql
select substring_index(email, '@', 1), substring_index(email, '@', -1)
from customer
limit 5;
```
Лимит 5 добавлен для того, чтобы сделать скриншот работы скрипта, так как все данные не помещаются в экран, для работоспособности запроса он не нужен

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL1.5.png)


### Задание 6*

Доработайте запрос из предыдущего задания, скорректируйте значения в новых колонках: первая буква должна быть заглавной, остальные — строчными.

```sql
select lower(substring_index(email, '@', 1)), upper(substring_index(email, '@', -1))
from customer
limit 5;
```
