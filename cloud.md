# Домашнее задание к занятию «Базы данных в облаке» - Леонид Хорошев


### Задание 1


#### Создание кластера
1. Перейдите на главную страницу сервиса Managed Service for PostgreSQL.
1. Создайте кластер PostgreSQL со следующими параметрами:
- класс хоста: s2.micro, диск network-ssd любого размера;
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/cloud/cloud1.1.png)
- хосты: нужно создать два хоста в двух разных зонах доступности и указать необходимость публичного доступа, то есть публичного IP адреса, для них;
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/cloud/cloud1.2.png)
- установите учётную запись для пользователя и базы.
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/cloud/cloud1.3.png)

#### Подключение к мастеру и реплике 

* Используйте инструкцию по подключению к кластеру, доступную на вкладке «Обзор»: cкачайте SSL-сертификат и подключитесь к кластеру с помощью утилиты psql, указав hostname всех узлов и атрибут ```target_session_attrs=read-write```.

```sql
psql "host=rc1a-iic0vm6yotm8uevs.mdb.yandexcloud.net \
      port=6432 \
      sslmode=verify-full \
      dbname=test \
      user=leo \
      target_session_attrs=read-write"
```

* Проверьте, что подключение прошло к master-узлу.
```
select case when pg_is_in_recovery() then 'REPLICA' else 'MASTER' end;
```
* Посмотрите количество подключенных реплик:
```
select count(*) from pg_stat_replication;
```

### Проверьте работоспособность репликации в кластере

* Создайте таблицу и вставьте одну-две строки.
```
CREATE TABLE test_table(text varchar);
```
```
insert into test_table values('Строка 1');
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/cloud/cloud1.5.png)
* Выйдите из psql командой ```\q```.

* Теперь подключитесь к узлу-реплике. Для этого из команды подключения удалите атрибут ```target_session_attrs```  и в параметре атрибут ```host``` передайте только имя хоста-реплики. Роли хостов можно посмотреть на соответствующей вкладке UI консоли.

```
psql "host=rc1b-h32of0ws2s7oyx8a.mdb.yandexcloud.net \
      port=6432 \
      sslmode=verify-full \
      dbname=test \
      user=leo \
      target_session_attrs=read-only"
```

* Проверьте, что подключение прошло к узлу-реплике.
```
select case when pg_is_in_recovery() then 'REPLICA' else 'MASTER' end;
```
* Проверьте состояние репликации
```
select status from pg_stat_wal_receiver;
```

* Для проверки, что механизм репликации данных работает между зонами доступности облака, выполните запрос к таблице, созданной на предыдущем шаге:
```
select * from test_table;
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/cloud/cloud1.6.png)



### Задание 2*

Создайте кластер, как в задании 1 с помощью Terraform.


*В качестве результата вашей работы пришлите скришоты:*

*1) Скриншот созданной базы данных.*
*2) Код Terraform, создающий базу данных.*

---

Задания, помеченные звёздочкой, — дополнительные, то есть не обязательные к выполнению, и никак не повлияют на получение вами зачёта по этому домашнему заданию. Вы можете их выполнить, если хотите глубже шире разобраться в материале.
