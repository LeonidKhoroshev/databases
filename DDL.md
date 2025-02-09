# Домашнее задание к занятию «Работа с данными (DDL/DML)» - Леонид Хорошев

### Задание 1
1.1. Поднимите чистый инстанс MySQL версии 8.0+. Можно использовать локальный сервер или контейнер Docker.

MySQL 8.0 database server поднят в виртуальной машине Alma Linux 8, развернутой в Яндекс облаке.

```
dnf install epel-release
dnf update
dnf install -y mysql-server mysql
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML1.1.png)

1.2. Создайте учётную запись sys_temp. 

```sql
mysql -u root -p
CREATE USER 'sys_temp'@'localhost' IDENTIFIED BY 'password';
```

1.3. Выполните запрос на получение списка пользователей в базе данных. (скриншот)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML1.2.png)

1.4. Дайте все права для пользователя sys_temp.

```sql
GRANT ALL PRIVILEGES ON *.* TO 'sys_temp'@'localhost';
```

1.5. Выполните запрос на получение списка прав для пользователя sys_temp. (скриншот)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML1.3.png)

1.6. Переподключитесь к базе данных от имени sys_temp.

```sql
ALTER USER 'sys_temp'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
SET PASSWORD FOR 'sys_temp'@'localhost' = '321';
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML1.4.png)

1.7. По ссылке https://downloads.mysql.com/docs/sakila-db.zip скачайте дамп базы данных.

```
wget https://downloads.mysql.com/docs/sakila-db.zip
unzip sakila-db.zip
rm sakila-db.zip
```

1.8. Восстановите дамп в базу данных.

Создаем у себя на хосте базу данных sakila 

```sql
mysql -u root
create database sakila;
exit
```

Дамп в базу данных sakila

```
mysql < sakila-schema.sql
mysql < sakila-data.sql
mysqldump -u root -p sakila > sakila-data.sql
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML1.5.png)


1.9. При работе в IDE сформируйте ER-диаграмму получившейся базы данных. При работе в командной строке используйте команду для получения всех таблиц базы данных. (скриншот)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML1.6.png)

### Задание 2
Составьте таблицу, используя любой текстовый редактор или Excel, в которой должно быть два столбца: в первом должны быть названия таблиц восстановленной базы, во втором названия первичных ключей этих таблиц. Пример: (скриншот/текст)
```
Название таблицы           | Название первичного ключа
actor                      | actor_id
actor_info                 | -
address                    | address_id
category                   | category_id
city                       | city_id
country                    | country_id
customer                   | customer_id
customer_list              | -
film                       | film_id
film_actor                 | actor_id, film_id
film_category              | category_id, film_id
film_list                  | -
film_text                  | film_id
inventory                  | inventory_id
language                   | language_id
nicer_but_slower_film_list | -
payment                    | payment_id
rental                     | rental_id
sales_by_film_category     | -
sales_by_store             | -
staff                      | staff_id 
staff_list                 | -
store                      | store_id
```
Список таблиц получен командой 

```sql
show tables;
```

Ключи получены командой

```sql
SHOW KEYS FROM table_name WHERE Key_name = 'PRIMARY';
```
где вместо table_name - названия таблиц (это очень неудобно, но к сожалению, я пока не нашел способ выводить все ключи одним запросом).

Там, где напротив таблиц стоят прочерки первичные ключи не обнаружены.

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DNL2.1.png)

Вывод с несколькими ключами, где они записаны через запятую.

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML2.2.png)

Вывод таблицы film_text выдал первичный ключ, совпадающий с таблицей film.

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML2.3.png)

### Задание 3*
3.1. Уберите у пользователя sys_temp права на внесение, изменение и удаление данных из базы sakila.

По условиям задания 1.4 пользователь sys_temp обладает всеми правами (скриншот к заданию 1.5), где права на внесение - INSERT, изменение - UPDATE и удаление - DELETE.

Удалим их командой REVOKE

```sql
REVOKE INSERT, UPDATE, DELETE ON sakila.* FROM 'sys_temp'@'localhost';
FLUSH PRIVILEGES;
```

3.2. Выполните запрос на получение списка прав для пользователя sys_temp. (скриншот)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/DDL/DDL_DML2.4.png)
