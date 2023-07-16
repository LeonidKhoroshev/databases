# Домашнее задание к занятию  «Очереди RabbitMQ» - Леонид Хорошев

---

### Задание 1. Установка RabbitMQ

Используя Vagrant или VirtualBox, создайте виртуальную машину и установите RabbitMQ.
Добавьте management plug-in и зайдите в веб-интерфейс.

Установка RabbitMQ на виртуальную машину Debian 11 в Virtualbox (все действия выполняются от пользователя root):

Добавляем репозиторий

```
tee /etc/apt/sources.list.d/rabbitmq.list <<EOF
## Provides modern Erlang/OTP releases from a Cloudsmith mirror
##
deb [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-erlang/deb/debian bullseye main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-erlang/deb/debian bullseye main

## Provides RabbitMQ from a Cloudsmith mirror
##
deb [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-server/deb/debian bullseye main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-server/deb/debian bullseye main
EOF
```

Производим обновление системы

```
sudo apt-get update -y
```

Устанавливаем пакеты Erlang 

```
apt install -y erlang-base \
 erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
 erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
 erlang-runtime-tools erlang-snmp erlang-ssl \
 erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl
```

Устанавливаем rabbitmq-server

```
sudo apt-get install rabbitmq-server -y --fix-missing
```

Создаем виртуальный хост, пользователя с правами администратора и подключаем плагин

```
rabbitmqctl add_vhost test
rabbitmqctl add_user leo
rabbitmqctl set_permissions -p test leo ".*" ".*" ".*"
rabbitmq-plugins enable rabbitmq_management
```

Проверяем статус rabbit

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit1.1.png)

Скриншот интерфейса

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit1.2.png)

---

### Задание 2. Отправка и получение сообщений

Используя приложенные скрипты, проведите тестовую отправку и получение сообщения.
Для отправки сообщений необходимо запустить скрипт producer.py.

Для работы скриптов вам необходимо установить Python версии 3 и библиотеку Pika.
Также в скриптах нужно указать IP-адрес машины, на которой запущен RabbitMQ, заменив localhost на нужный IP.

```shell script
$ pip install pika
```

На нашей виртуальной машине уже установлен pyhton версии 3.9 при выполнении прошлых заданий.

Создаем папку rabbit, в которую помещаем прилагаемые к заданию скрипты

```
mkdir rabbit
cd rabbit
```

В скрипте producer.py меняем:
- имя нашей виртуальной машины на test (строка 5);
- название очереди на my_queue (строка 7);
- сообщение на Salut (строка 8).

Первое изменение необходимо - чтобы все работало, так как в веб-интерфейс мы входим не с localhost, а остальные - "для закрепления материала".

```
#!/usr/bin/env python
# coding=utf-8
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('test'))
channel = connection.channel()
channel.queue_declare(queue='my_queue')
channel.basic_publish(exchange='', routing_key='my_queue', body='Salut!')
connection.close()
```
Запускаем скрипт

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit2.1.png)

Создадим скрипт comsumer.py

Здесь изменения следующие:
- добавлена авторизация по пользователю и паролю (строка 4);
- изменено имя виртуальной машины на test (строка 5);
- имя очереди на my_queue (строка 8);
- полностью переписана строка 15, так как исходный вариант был не рабочий (unexpected argument) и заменен на показанный в лекции.
  
```
#!/usr/bin/env python
# coding=utf-8
import pika
credentials = pika.PlainCredentials('leo', 'leo')
parameters = pika.ConnectionParameters('test', 5672, 'test', credentials)
connection = pika.BlockingConnection(parameters)
channel = connection.channel()
channel.queue_declare(queue='my_queue')


def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

channel.basic_consume(on_message_callback=callback, queue='my_queue', auto_ack=False)
channel.start_consuming()

```

Скриншот с добавленным консьюмером

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit2.2.png)

---

### Задание 3. Подготовка HA кластера

Используя Vagrant или VirtualBox, создайте вторую виртуальную машину и установите RabbitMQ.
Добавьте в файл hosts название и IP-адрес каждой машины, чтобы машины могли видеть друг друга по имени.

Пример содержимого hosts файла:
```shell script
$ cat /etc/hosts
192.168.0.10 rmq01
192.168.0.11 rmq02
```
После этого ваши машины могут пинговаться по имени.

Затем объедините две машины в кластер и создайте политику ha-all на все очереди.

*В качестве решения домашнего задания приложите скриншоты из веб-интерфейса с информацией о доступных нодах в кластере и включённой политикой.*

Также приложите вывод команды с двух нод:

```shell script
$ rabbitmqctl cluster_status
```

Для закрепления материала снова запустите скрипт producer.py и приложите скриншот выполнения команды на каждой из нод:

```shell script
$ rabbitmqadmin get queue='hello'
```

После чего попробуйте отключить одну из нод, желательно ту, к которой подключались из скрипта, затем поправьте параметры подключения в скрипте consumer.py на вторую ноду и запустите его.

Создаем в Virtualbox вторую виртуальную машину и вносим соответствующие изменения в файл etc/hosts

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.1.png)

Далее объединяем обе виртуальные машины в кластер следующими действиями:

На ноде slave прописываем политику ha-all (в целях возможности репликации очередей)

```
 rabbitmqctl set_policy ha-all "" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
```

Далее проверяем на обоих нодах содержимое файла /var/lib/rabbitmq/.erlang.cookie (оно должно быть идентичным).
В нашем случае обошлось без копирования, так как вторая нода была создана клонированием первой виртуальной машины в Virtualbox.

Затем задаем права, а также владельца и группу на файл .erlang.cookie на обоих нодах

```
chmod 400 /var/lib/rabbitmq/.erlang.cookie && chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
```

Следующим действием создаем кластер из двух нод (изначально планировалось присоединить ноду slave к ноде test, но кластер "не взлетел", поэтому выбран вариант присоединения ноды test к slave)

```
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@slave
rabbitmqctl start_app
```

Проверяем результат на обоих нодах

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.2.png)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.3.png)

Смотрим веб-нитерфейс по обоим адресам:

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.4.png)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.5.png)

Запускаем скрипт producer.py из задания №2, но команда $ rabbitmqadmin get queue='hello' не заработала ни на одной из нод.

Загружаем скрипт и выдаем разрешения

```
wget https://raw.githubusercontent.com/rabbitmq/rabbitmq-management/v3.7.8/bin/rabbitmqadmin
chmod 777 rabbitmqadmin
```
Так как используем python3, меняем первую строчку скрипта с

```
#!/usr/bin/env python
```
на

```
#!/usr/bin/env python3
```

Проверка в консоли

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.8.png)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.9.png)

Наша очередь в веб-интерфейсе:

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.7.png)

В скрипте consumer.py из Задания 2 изменим название хоста с test на slave и попробуем подключиться к очереди:

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.14.png)

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.11.png)

Далее отключаем ноду slave и меняем в скрипте consumer.py хост обратно на test

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.12.png)

Запускаем

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/rabbit/rabbit3.13.png)

### * Задание 4. Ansible playbook

Напишите плейбук, который будет производить установку RabbitMQ на любое количество нод и объединять их в кластер.
При этом будет автоматически создавать политику ha-all.

*Готовый плейбук разместите в своём репозитории.*

