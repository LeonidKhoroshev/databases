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

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/cloud/cloud1.7.png)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/cloud/cloud1.8.png)

Код terraform (файл main.tf)

```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.14"
}

provider "yandex" {
  zone = "ru-central1-a"
  token = "y0_AgAAAAAp7qigAATuwQAAAADhAMvANnKdk1oJS8uomqePQxQr0K1VqSs"
  cloud_id = "b1g3ks25rm2qagep03qb"
  folder_id = "b1gadttfn3t0cohh2hk2"
}

resource "yandex_mdb_postgresql_cluster" "netology" {
  name                = "netology"
  environment         = "PRODUCTION"
  network_id          = "enpkqu7u4btrg3ucmji3"
  deletion_protection = false

  config {
    version = 15
    resources {
      resource_preset_id = "s2.micro"
      disk_type_id       = "network-ssd"
      disk_size          = 10
    }

  }

  host {
    zone      = "ru-central1-a"
    name      = "host_1"
    subnet_id = "e9bu1cd8q2f55hlq03vg"
  }

  host {
    zone      = "ru-central1-b"
    name      = "host_2"
    subnet_id = "e2lkolbvj8rtokc1s51p"
  }
}

resource "yandex_mdb_postgresql_database" "test" {
cluster_id = "yandex_mdb_postgresql_cluster.mypg.id"
name       = "test"
owner      = "leo"
depends_on = [
  yandex_mdb_postgresql_user.leo
]
}

resource "yandex_mdb_postgresql_user" "leo" {
  cluster_id = "yandex_mdb_postgresql_cluster.mypg.id"
  name       = "leo"
  password   = "12345"
}
resource "yandex_vpc_network" "default" { name = "default" }

resource "yandex_vpc_subnet" "default-ru-central1-a" {
  name           = "default-ru-central1-a"
  zone           = "ru-central1-a"
  network_id     = "e9bu1cd8q2f55hlq03vg"
  v4_cidr_blocks = ["10.128.0.0/24"]
}

resource "yandex_vpc_subnet" "default-ru-central1-b" {
  name           = "default-ru-central1-b"
  zone           = "ru-central1-b"
  network_id     = "e2lkolbvj8rtokc1s51p"
  v4_cidr_blocks = ["10.129.0.0/24"]
}
```
