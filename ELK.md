# Домашнее задание к занятию «ELK» - Леонид Хорошев

---

### Задание 1. Elasticsearch 

Установите и запустите Elasticsearch, после чего поменяйте параметр cluster_name на случайный. 

Установка из официального репозитоия оказалась недоступна, поэтому пакет скачан с яндекс зеркала (к сожалению на данный момент репозиторий яндекса содержит только deb пакеты, поэтому все работы выполнены на ВМ с ОС Debian11, созданной в Яндекс облаке специально под выполнение данного задания).

```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/e/elasticsearch/elasticsearch-8.6.2-amd64.deb
dpkg -i elasticsearch-8.6.2-amd64.deb
rm elasticsearch-8.6.2-amd64.deb
```
Далее находим файл elasticsearch.yml и редактируем настройки Elasticsearch, меняем имя кластера со стандартного на leonid, а также параметр xpack.security.enabled с true на false, это необходимо для выполнения команды curl -X GET, в противном случае получаем пустой ответ от сервера curl:(52).

```
find / -name elasticsearch.yml
nano /etc/elasticsearch/elasticsearch.yml
cluster.name: leonid
xpack.security.enabled: false
systemctl enable elasticsearch
systemctl start elasticsearch
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk1.1.png)

---

### Задание 2. Kibana

Установите и запустите Kibana.

Устанавливаем Kibana аналогично заданию 1

```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/k/kibana/kibana-8.6.2-amd64.deb
dpkg -i kibana-8.6.2-amd64.deb
rm kibana-8.6.2-amd64.deb
```

Меняем настройки Kibana для возможности подключения через веб-интерфейс с других хостов.

```
nano /etc/kibana/kibana.yml
server.host: "0.0.0.0"
systemctl enable kibana
systemctl start kibana
```

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk2.1.png)

---

### Задание 3. Logstash

Установите и запустите Logstash и Nginx. С помощью Logstash отправьте access-лог Nginx в Elasticsearch.

Устанавливаем Logstash аналогично заданиям 1 и 2, а также Nginx из apt-репозитория

```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/l/logstash/logstash-8.6.2-amd64.deb
dpkg -i logstash-8.6.2-amd64.deb
rm logstash-8.6.2-amd64.deb
apt install ngnix
systemctl enable logstash
systemctl start logstash
systemctl enable nginx
systemctl start nginx
```
Проверяем, что установка прошла корректно
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk3.1.png)

Создаем файл logstash.conf в директории etc/logstash/conf.d, где прописываем настройки отправки access-логов Nginx в Elasticsearch:
- секция input описывает, где мы берем данные (в нашем случае это файл access.log);
- секция filter - то как мы обрабатываем и приводим данные в вид, удобный для восприятия пользователем;
- секция output описывает вывод данных для просмотра. 
```
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
 }
}

filter {
  grok {
    match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user_name}
\[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url}
HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes}
\"%{DATA:referrer}\" \"%{DATA:agent}\"" }
    }
    mutate {
         remove_field => [ "host" ]
    }
}


output {
  elasticsearch {
  hosts => "localhost:9200"
  data_stream => "true"
```

Проверяем корректность настроек
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk3.2.png)

Проверяем наличие логов nginx
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk3.3.png)

---

### Задание 4. Filebeat. 

Установите и запустите Filebeat. Переключите поставку логов Nginx с Logstash на Filebeat. 

```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/f/filebeat/filebeat-8.6.2-amd64.deb
dpkg -i filebeat-8.6.2-amd64.deb
rm filebeat-8.6.2-amd64.deb
systemctl enable filebeat
systemctl start filebeat
```

Проверяем, что установка прошла корректно
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk4.1.png)

Прописываем настройки filebeat в файле /etc/filebeat/filebeat.yml

```
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - '/var/log/nginx/access.log'

processors:
- decode_json_fields:
    fields: ["message"]
    target: "json"
    overwrite_keys: true

output.elasticsearch:
  hosts: ["localhost:9200"]
  indices:
    - index: "filebeat-nginx-%{[agent.version]}-%{+yyyy.MM.dd}"

logging.json: true
logging.metrics.enabled: false
```

Смотрим в Kibana наличие нашего нового индекса
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk4.2.png)

Проверяем наличие логов nginx
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk4.3.png)


### Задание 5*. Доставка данных 

Настройте поставку лога в Elasticsearch через Logstash и Filebeat любого другого сервиса, но не Nginx. 
Для этого лог должен писаться на файловую систему, Logstash должен корректно его распарсить и разложить на поля.

В качестве сервиса, выбрана Suricata - система обнаружения вторжений (IDS) с открытым исходным кодом. Данное программное обеспечение фиксирует информацию в виде логов в файле  /var/log/suricata/fast.log, следовательно анализировать ее удобнее через графический интерфейс.

1.Установка Suricata
```
sudo su
apt install software-properties-common
add-apt-repository ppa:oisf/suricata-stable
apt update
apt install suricata
suricata-update
```

2. Настройка Suricata
```
nano /etc/suricata/suricata.yaml
```
Меняем значение параметра EXTERNAL_NET на "any" путем раскомментирования соответствующей строки
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk5.2.png)

Прописываем правила и запускаем утилиту
```
sudo suricata-update -o /etc/suricata/rules
systemctl start suricata
systemctl status suricata
```
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk5.1.png)

3. Корректировка конфигурации filebeat


В дополнение к коду, прописанному в рамках выполнения Задания 4 дописываем индекс filebeat-suricata

```
filebeat.inputs:
- type: log
  enabled: true
  paths:
#    - '/var/log/nginx/access.log'
    - '/var/log/suricata/eve.json'

processors:
- decode_json_fields:
    fields: ["message"]
    target: "json"
    overwrite_keys: true

output.elasticsearch:
  hosts: ["localhost:9200"]
  indices:
#    - index: "filebeat-nginx-%{[agent.version]}-%{+yyyy.MM.dd}"
    - index: "filebeat-suricata"

```

Перезапускаем службы
```
systemctl restart elasticsearch
systemctl restart kibana
```

4. Проверяем интерфейс в kibana
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk5.3.png)
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk5.4.png)
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/ELK/elk5.5.png)
