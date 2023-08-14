# Домашнее задание к занятию «Репликация и масштабирование. Часть 2» - Леонид Хорошев

---

### Задание 1. Опишите основные преимущества использования масштабирования методами:

1. Активный master-сервер и пассивный репликационный slave-сервер (горизонтальное масштабирование) – классический пример репликации, выполненной в рамках предыдущего домашнего задания. Основные преимущества:
   - рост производительности чтения данных;
   - повышение отказоустойчивости и надежности;
   - рост доступности данных и скорости их чтения;
   - удобство обслуживания (поскольку slave-сервер содержит данные только для чтения, то можно запускать процессы обслуживающие базу данных не прерывая работу системы). 

2. Master-сервер и несколько slave-серверов (горизонтальное масштабирование) – обладает теми же преимуществами, что и репликация, описанная выше, схема обычно используется в системах, где преобладают запросы на чтение данных (они распределяются между slave-серверами). Основное преимущество такого подхода – удобство распределения данных  (в том числе по географическому признаку). Например, можно установить сервера в разных городах на большом расстоянии друг от друга, перенаправлять клиентов и  выбирать сервер в зависимости от региона. Таким образом мы получаем выигрыш за счет уменьшения пути путешествия запроса к серверу. По сравнению со схемой с одним slave-сервером обладает более высокой доступностью и скоростью чтения данных, но с точки зрения надежности определенного выигрыша нет, так как при потере .

3. Активный сервер со специальным механизмом репликации  distributed replicated block device (отказоустойчивость).
DRBD, является частью ядра Linux и позволяет повысить  отказоустойчивость системы из нескольких серверов, объединяя данные в один RAID массив (RAID1). Данные реплицируются в виде зеркального отображения содержимого блочных устройств (жестких дисков, разделов, логических томов и т.д.) между хостами. Преимущества:
   - репликация происходит непрерывно и в режиме реального времени;
   - данные отображаются как синхронно, так и асинхронно, при синхронном отображении приложения получают уведомления о завершении записи после выполнения операций записи на всех подключенных хостах, а при асинхронном – приложения получают уведомления о завершении записи, когда записи завершаются локально, что обычно происходит до того, как они распространяются на другие хосты.

4. SAN-кластер (отказоустойчивость).
Технология для подключения внешних устройств хранения данных к серверам. Предназначена для объединения ресурсов дисковой памяти, подключаемых к одному или нескольким серверам. Как правило, характеризуется высокой пропускной способностью между элементами дисковой системы. Преимущества:
   - удобство эксплуатации за счет простого управления дисковыми устройствами (подключение и монтирование дисков и томов);
   - быстродействие за счет независимости от трафика LAN и WAN-сетей и возможность резервирования данных без загрузки локальной сети и серверов;
   - отказоустойчивость (если одно устройство выходит из строя, тут же подключается и монтируется другое устройство).

---

### Задание 2


Разработайте план для выполнения горизонтального и вертикального шаринга базы данных. База данных состоит из трёх таблиц: 

- пользователи, 
- книги, 
- магазины (столбцы произвольно). 

Опишите принципы построения системы и их разграничение или разбивку между базами данных.

Представим, что это база данных небольшого книжного интернет-магазина. Всего на нашем портале зарегистрировано 400 пользователей, данные о пользователях у нас размещены в таблице Users. Для повышения ее производительности и уменьшения скорости обработки запросов разделим ее на 2 части (выполним  партицирование). Можно делить таблицу с пользователями на более мелкие составные таблицы, но чтобы не слишком усложнять, выполним партицирование на 2 части по users_id.

Так как магазин у нас небольшой, но динамично-развивающийся, необходимо предусмотреть рост количества активных пользователей и их заказов. Для этих целей целесообразно разнести наших пользователей по нескольким серверам (чтобы не усложнять, тоже сделаем по двум). Тут  возможно несколько вариатнов, например если магазин работает в нескольких городах (Тула и Москва), то для повышения скорости обработки запросов к таблицам, их можно шардировать по географическому признаку (например по address_id). Мы же представим, что наш магазин работает в одном регионе (например Московская область) и разделим таблицы по users_id (в случае при росте количества пользователей, можно выполнять партицирование таблицы users, создавая более мелкие таблицы на каждых новых 200 пользователей и разносить их поочередно на наши сервера, то есть выполнять горизонтальный шардинг).

Данные об ассортименте книжной продукции нашего магазина будут хранится в таблице books. Таблицы books и users должны быть связаны между собой, так можно видеть, какие книги заказывают чаще всего, чтобы более качественно планировать закупки и корректировать ассортимент. Свяжем их через books_id (можно попробовать выполнить связь через заказы orders_id, так как в одном заказе может быть несколько книг, но данный подход потребует от нас создания дополнительнгой таблицы с заказами, а по условиям задания - у нас их 3, так что принимаем решение - не усложнять). Партицирование и шагдинг осуществляем аналогично таблице Users.

Третья таблица - магазины (stores). Это могут быть как оффлайн-точки продаж, так и онлайн-площадки. Скорее всего их немного, так что горизонтальное партицирование к данной таблице применяться не будет. Данная таблица предполагается кратно меньше предыдущих двух, следовательно размещаем ее на одном из серверов. Таблица stores будет связана с остальными двумя таблицами через books_id (чтобы понимать ассортимент в каждой точке продаж) и users_id (чтобы понять, насколько наши магазины посещаемы).
 
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.11.png)


---
### Задание 3*

Выполните настройку выбранных методов шардинга из задания 2.

Для выполнения задания настроим postgres на двух нодах и поднимем на них кластер. Настройка будет осуществляться на виртуальных машинах в Яндекс облаке. Несмотря на то, что ноды называются mysql1 и mysql2, потому что подняты для выполнения предыдущего домашнего задания (настройка репликаций master-slave и master-master), разворачивать на них будем именно postgres.

1. Установка пакетного менеджера, СУБД и менеджер управления репликациями.
```sql
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y postgresql14-server postgresql14-contrib postgresql14
yum install -y repmgr_14
```

2. Инициализируем первую ноду mysql1
```sql
/usr/pgsql-14/bin/postgresql-14-setup initdb
```
3. Правим конфигурационные файлы на первой ноде.
```
nano /var/lib/pgsql/14/data/postgresql.conf
```

```
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
wal_log_hints = on
archive_mode = on
archive_command = '/bin/true'
wal_keep_size = 1GB
```

```
nano /var/lib/pgsql/14/data/pg_hba.conf
```

```
host all repmgr 10.128.0.0/24 scram-sha-256
host replication repmgr 10.128.0.0/24 md5
```

4. Запускаем СУБД

```
systemctl enable --now postgresql-14
```

5. Заходим на первой ноде в postgres и создаем пользователя, необходимого нам для настройки репликации

```sql
su - postgres
psql
CREATE USER repmgr WITH ENCRYPTED PASSWORD '12345';
CREATE DATABASE repmgr OWNER repmgr;
ALTER USER repmgr WITH SUPERUSER;
ALTER USER repmgr SET search_path TO repmgr, "$user", public;
```

6. Прорисываем параметры подключения (на обоих нодах) 

```
nano ~/.pgpass
```

```
# hostname:port:database:username:password
*:*:*:repmgr:12345
```

```
chmod 600 ~/.pgpass
```

7. На второй ноде проверяем возможность запуска репликации
   
```
psql -X -U repmgr -h 10.128.0.12 -c "IDENTIFY_SYSTEM" replication=1
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.12.png)

8. Переходим к настройкам менеджера репликаций.

```
nano /etc/repmgr/14/repmgr.conf
```

Первая нода:

```
node_id=1
node_name='mysql1'
conninfo='host=10.128.0.12 dbname=repmgr user=repmgr'
data_directory='/var/lib/pgsql/14/data/'
use_replication_slots=true
log_level='INFO'
log_file='/var/log/repmgr/repmgr-14.log'
pg_bindir='/usr/pgsql-14/bin/'

service_start_command = 'sudo systemctl start postgresql-14'
service_stop_command = 'sudo systemctl stop postgresql-14'
service_restart_command = 'sudo systemctl restart postgresql-14'
service_reload_command = 'sudo systemctl reload postgresql-14'
```

Вторая нода:

```
node_id=2
node_name='mysql2'
conninfo='host=10.128.0.3 dbname=repmgr user=repmgr'
data_directory='/var/lib/pgsql/14/data/'
use_replication_slots=true
log_level='INFO'
log_file='/var/log/repmgr/repmgr-14.log'
pg_bindir='/usr/pgsql-14/bin/'

service_start_command = 'sudo systemctl start postgresql-14'
service_stop_command = 'sudo systemctl stop postgresql-14'
service_restart_command = 'sudo systemctl restart postgresql-14'
service_reload_command = 'sudo systemctl reload postgresql-14'
```

9. Регистрируем и стартуем менеджер репликаций на первой ноде

```
$ sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf master register
$ sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf cluster show
```

10. На второй ноде проверяем возможность подключения к первой и запускаем репликацию

```
sudo -u postgres /usr/pgsql-14/bin/repmgr -h 10.128.0.12 -U repmgr -d repmgr -f /etc/repmgr/14/repmgr.conf standby clone --dry-run
sudo -u postgres /usr/pgsql-14/bin/repmgr -h 10.128.0.12 -U repmgr -d repmgr -f /etc/repmgr/14/repmgr.conf standby clone
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.13.png)

11. Создаем на первой ноде базу данных test, подключаемся к ней и создаем в ней таблицы из задания 2 (users, books, stores)

```sql
CREATE DATABASE test;
\connect test;
```

Users

```sql
CREATE TABLE users (
users_id bigint not null,
lastname character varying not null,
firstname character varying not null,
books_id bigint not null,
stores_id bigint not null
);
```

Books

```sql
CREATE TABLE books (
books_id bigint not null,
author character varying not null,
title character varying not null,
stores_id bigint not null
);
```

Stores

```sql
CREATE TABLE stores (
id_stores bigint not null,
address text not null,
books_id bigint not null,
users_id bigint not null
);
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.14.png)

12. Настраиваем вертикальное партицирование таблиц users и books. Для этого создаем таблицы users_1 и users_2, в которых будем хранить данные пользователей в соответствии со схемой в задании 2, аналогичную процедуру проделываем с таблицей books.

```sql
CREATE TABLE users_1 (
CHECK ( users_id > 0 AND users_id <=200 )
) INHERITS (users)
```

```sql
CREATE TABLE users_2 (
CHECK ( users_id > 200 AND users_id <=400 )
) INHERITS (users)
```

```sql
CREATE TABLE books_1 (
CHECK ( books_id > 0 AND books_id <=200 )
) INHERITS (books)
```

```sql
CREATE TABLE books_2 (
CHECK ( books_id > 200 AND books_id <=400 )
) INHERITS (books)
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.15.png)
