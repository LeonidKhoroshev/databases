# Домашнее задание к занятию «Репликация и масштабирование. Часть 1» - Леонид Хорошев


### Задание 1

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

В репликации master-slave один сервер выполняет роль master, то есть позволяет редактировать (вносить, удалять, изменять) данные, а остальные сервера (slave) могут только читать информацию из базы данных, в то время как в репликации master-master каждый сервер является мастером и слейвом одновременно, то есть редактировать данные можно на любой ноде.

---

### Задание 2

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

1. В Яндекс облаке создаем 2 виртуальные машины Centos7 (mysql1 и mysql2) с минимальными характеристиками (в целях экономии наших ресурсов).

2. Устанавливаем на 2 машины mysql клиент и mysql сервер (устанавливаем ключ, пакетный менеджер и требуемое программное обеспечение).

```
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el7-6.noarch.rpm
yum -y install mysql-server mysql-client
```

3. Создаем директорию для записи логов, инициализируем СУБД и предоставляем права доступа к директориям c логами и изменяемыми файлами. 

```
mkdir -p /var/log/mysql
mysqld --initialize
cat /var/log/mysqld.log
chown -R mysql: /var/log/mysql
chown -R mysql: /var/lib/mysql
```

4. Прописываем конфигурации на обоих нодах (на данном этапе критически важно прописать отличные друг от друга параметры server-id, иначе репликация не заработает).

```
nano /etc/my.cnf
```

Конфигурация mysql1

```
bind-address=0.0.0.0
server-id=1
log_bin=/var/log/mysql/mybin.log
```

Конфигурация mysql2

```
bind-address=0.0.0.0
server-id=2
log_bin=/var/log/mysql/mybin.log
```

5. Запускаем mysql на обоих нодах

```
systemctl start mysqld
systemctl status mysqld
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.1.png)
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.2.png)

6. Подключаемся к нашим СУБД на обеих нодах через пароль, который мы увидели в логах (п.3, команда cat), и меняем пароль для корректной работы репликации

```sql
mysql -p
ALTER USER 'root'@'localhost' IDENTIFIED BY '12345';
FLUSH PRIVILEGES;
```


6. Создаем пользователя на обоих нодах для применения репликации
```sql
CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY '12345';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
```

7. На первой ноде mysql1 выводим на экран информацию, необходимую для подключения к ней второй ноды mysql2.

```sql
SHOW MASTER STATUS;
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.3.png)

8. Подключаем вторую ноду к первой в качестве slave.

```sql
CHANGE MASTER TO MASTER_HOST='158.160.108.90', MASTER_USER='replication', MASTER_PASSWORD='12345', MASTER_LOG_FILE = 'mybin.000001', MASTER_LOG_POS=1519;
START SLAVE;
SHOW SLAVE STATUS\G;
```

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.4.png)

9. Проверяем корректность работы репликации создавая и удаляя базы данных на мастере mysql1 и отслеживая изменения на slave mysql2.

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.5.png)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/replication/replication2.6.png)

---

### Задание 3* 

Выполните конфигурацию master-master репликации. Произведите проверку.


