# Домашнее задание к занятию «SQL. Часть 2» - Леонид Хорошев


### Задание 1

Одним запросом получите информацию о магазине, в котором обслуживается более 300 покупателей, и выведите в результат следующую информацию: 
- фамилия и имя сотрудника из этого магазина;
- город нахождения магазина;
- количество пользователей, закреплённых в этом магазине.

1. Получаем информацию о базе данных и таблицах которые могут нам понадобиться в формировании запроса.
   
```sql
mysql -u root -p
use sakila
show tables;
describe staff;
describe store;
describe customer;
describe city;
```

2. Выбираем нужную по условиям информацию:
   - имя и фамилию сотрудников объединяем в одну колонку;
   - через address_is к таблице staff присоединяем таблицу address;
   - через address_is к таблице address присоединяем таблицу city для вывода города нахождения магазина;
   - через address_is присоединяем таблицу customer и функцией count считаем, где покупетелей больше 300;
   - группируем вывод по сотрудникам и городам.
     
```sql
select concat(staff.last_name, staff.first_name), city.city, count(customer.customer_id)
from staff
join address on address.address_id = staff.address_id
join city on city.city_id = address.city_id
join customer on customer.store_id = staff.store_id
group by staff.last_name, staff.first_name, city.city
having count(customer.customer_id) > 300;
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.1.png)

Вывод очень странный, предполагаю, что что-то не то с моей версией базы данных, так как в таблице staff у меня всего 2 сотрудника

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.2.png)


### Задание 2

Получите количество фильмов, продолжительность которых больше средней продолжительности всех фильмов.

В данном задании нам понадобиться только таблица с фильмами (film)

```sql
 describe film;
```
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.3.png)

Выводим на экран в терминале количество фильмов, продолжитедльностью больше средней

```sql
select count(title) 
from film
where length > (select avg(length) from film);
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.4.png)

В данном задании результат вполне правдоподобен, более того, он легко проверяется вычислением общего количества фильмов и фильмов с продолжительностью ниже средней

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.5.png)


### Задание 3

Получите информацию, за какой месяц была получена наибольшая сумма платежей, и добавьте информацию по количеству аренд за этот месяц.

Вся необходимая информация содержится в таблице с платежами (payment)

```sql
describe payment;
```
Из таблицы с платежами нам необходимо:
 - сгруппировать платежи по месяцам и годам (чтобы не сложить одни и те же месяцы разных лет, например август 2005 с августом 2006), посчитать сумму платежей и их количество;
 - отсортировать их начиная с наиболее крупной суммы и вывести на экран в консоли только этот месяц.

```sql
select year(payment_date), month(payment_date), sum(amount), count(payment_id)
from payment
group by year(payment_date), month(payment_date)
order by sum(amount) desc
limit 1;
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.6.png)

Выполним проверку, убрав из запроса ограничение на вывод (limit 1). Результаты совпадают

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.7.png)

### Задание 4*

Посчитайте количество продаж, выполненных каждым продавцом. Добавьте вычисляемую колонку «Премия». Если количество продаж превышает 8000, то значение в колонке будет «Да», иначе должно быть значение «Нет».

В данном задании пошел по самому простому пути и обошелся данными таблицы rental.  Определили сотрудников по staff_id, количество продаж посчитали по rental_id.

```sql
select staff_id, count(rental_id),
case when count(rental_id) > 8000 then 'yes'
else 'no'
end as 'bonus'
from rental
group by staff_id;
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.8.png)

### Задание 5*

Найдите фильмы, которые ни разу не брали в аренду.

Смысл данного задания не понятен, так как ровно такая же задача разбиралась на лекции и скрипт указан в презентации. В целом у меня получился похожий запрос, за исключением того, что я пока опасаючь использовать сокращения имен таблиц, чтобы не запутаться (хотя понимаю, что такая практика может оптимизировать сам запрос и время отработки скрипта). В остальном скрипт идентичен запросу из презентации, за исключением того, что в моей версии вместо left join использован inner join (или просто join).

```sql
select film.title
FROM film
join inventory on inventory.film_id = film.film_id
join rental on rental.inventory_id = inventory.inventory_id
where rental.rental_id is null;
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/SQL1/SQL2.9.png)
