# Домашнее задание к занятию «Защита сети» - Леонид Хорошев 

------

### Подготовка к выполнению заданий

Задание выполнено на двух хостах Debian 11 (защищаемый хост) и Kali (хост "условного" злоумышленника).
1. Подготовка защищаемой системы:

Установика **Suricata**:
```
sudo su
apt install software-properties-common
add-apt-repository ppa:oisf/suricata-stable
apt update
apt install suricata
suricata-update
```
Настройка конфигурации:
```
nano /etc/suricata/suricata.yaml
```
меняем значение параметра EXTERNAL_NET на "any" путем раскомментирования данной строки
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network0.1.png)

Прописываем правила и запускаем утилиту:
```
sudo suricata-update -o /etc/suricata/rules
systemctl start suricata
systemctl status suricata
```
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network0.2.png)

Установка и запуск **Fail2Ban**.

```
apt update
apt install fail2ban
fail2ban-client --version
systemctl start fail2ban
systemctl status fail2ban
```
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network0.3.png)
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network0.4.png)

Настройка Fail2Ban:
В директории etc/fail2ban создаем новый файл jail.local, в котором прописываем настройки, необходимые для выполнения Задания 2 (контроль подключений по ssh). 
```
nano /etc/fail2ban/jail.local
[DEFAULT]
bantime = 1m
findtime = 10s
maxretry = 2

ignoreip = 127.0.0.1

[sshd]
enabled = true
```
- bantime - время блокировки забаненного ip адреса;
- findime - временное окно перед баном;
- maxretry - максимальное количество неверных попыток ввода пароля;
- ignoreip - исключение для определенных ip адресов (необходимо, чтобы не заблокировать собственный хост или хосты, доступ с которых необходим в любое время).
- 
По умолчанию конфигурация "ловушек", которые использует fail2ban содержится в файле jail.conf, однако внесенные в него изменения могут быть удалены при обновлении пакетов, кроме того файл крайне объемен и создание новой конфигурации менее трудозатратно.
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network0.11.png)


Далее для удобства восприятия информации из логов настроим их отправку в Elasticsearch и визуализацию в интерфейсе Kibana.
Устанавливаем и настроиваем на защищаемом хосте следующее программное обеспечение:
- Elasticsearch:
```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/e/elasticsearch/elasticsearch-8.6.2-amd64.deb
dpkg -i elasticsearch-8.6.2-amd64.deb
rm elasticsearch-8.6.2-amd64.deb
```
Настройки:
```
nano /etc/elasticsearch/elasticsearch.yml
xpack.security.enabled: false
```

- Kibana:
```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/k/kibana/kibana-8.6.2-amd64.deb
dpkg -i kibana-8.6.2-amd64.deb
rm kibana-8.6.2-amd64.deb
```
Настройки:
```
nano /etc/kibana/kibana.yml
server.host: "0.0.0.0"
```

- Logstash:
```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/l/logstash/logstash-8.6.2-amd64.deb
dpkg -i logstash-8.6.2-amd64.deb
rm logstash-8.6.2-amd64.deb
```
- Filebeat:
```
wget https://mirror.yandex.ru/mirrors/elastic/8/pool/main/f/filebeat/filebeat-8.6.2-amd64.deb
dpkg -i filebeat-8.6.2-amd64.deb
rm filebeat-8.6.2-amd64.deb
```

Более подробно информацию об установке и настройках стека ELK можно посмотреть по данной ссылке: https://github.com/LeonidKhoroshev/databases/blob/main/ELK.md

Настраиваем отображение логов suricata через соответствующий модуль:
```
filebeat modules enable suricata
filebeat setup -e
nano /etc/filebeat/modules.d/suricata.yml
# Module: suricata
# Docs: https://www.elastic.co/guide/en/beats/filebeat/8.6/filebeat-module-suri>

- module: suricata
  # All logs
  eve:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["var/log/suricata/eve.json"]
```


![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network0.6.png)

Для fail2ban готового модуля найти не удалось, поэтому настройка доставки логов в Elasticsearch выполнена вручную путем создания соответствующего конфигурационного файла:
```
 nano /etc/logstash/conf.d/fail2ban.conf

input {
  file {
    path => "/var/log/fail2ban.log"
    type => "fail2ban"
  }
}

filter {
  if [type] == "fail2ban" {
    grok {
      patterns_dir => ["/etc/logstash/patterns"]
      match => [ "message", "%{FAIL2BAN_BAN}" ]
      add_tag => [ "ban" ]
      named_captures_only => true
    }
    grok {
      patterns_dir => [ "/etc/logstash/patterns" ]
      match => [ "message", "%{FAIL2BAN_UNBAN}" ]
      add_tag => [ "unban" ]
      named_captures_only => true
    }
    grok {
      patterns_dir => [ "/etc/logstash/patterns" ]
      match => [ "message", "%{FAIL2BAN_ALREADYBAN}" ]
      add_tag => [ "already_ban" ]
      named_captures_only => true
    }

    mutate {
      remove_tag => ["_grokparsefailure"]
    }
  }
}

output {
    elasticsearch {
    hosts => "localhost:9200"
    data_stream => "true"
    }
}
```


2. Подготовка системы злоумышленника: установите **nmap** и **thc-hydra** либо скачайте и установите **Kali linux**.

В качестве системы злоумышленника используется ОС Kali linux, развернутая в VirtualBox в рамках выполнения домашних заданий по предыдущим темам.
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network0.5.png)

------

### Задание 1

Проведите разведку системы и определите, какие сетевые службы запущены на защищаемой системе:

**sudo nmap -sA < ip-адрес >**
Сканирование на наличие правил брандмауэра 
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.1.png)
Сканирование показало, что хост запущен, 1000 портов tcp не предоставили никакой информации о своем статусе, что может значить, что на хосте настроен брандмауэр (или иной защитный механизм), препядствующий подключению.

**sudo nmap -sT < ip-адрес >**
Сканирование наиболее часто используемых tcp портов
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.2.png)
По 998 портам в соединении отказано, что также свидетельствует о наличии защиты, но по 2м портам уже получена информация - на 22 порт используется ssh, а на 9200 wap-wsp (протокол беспроводного доступа к сети, на данном порту у нас функционирует elasticsearch).

**sudo nmap -sS < ip-адрес >**
Полуоткрытое сканирование (без установки полного подключения)
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.3.png)
Результаты те же, что и при обычном tcp сканировании, но теоретически полуоткрытое сканированее сложнее обнаружить.

**sudo nmap -sV < ip-адрес >**
Сканирование на предмет запущенных сервисов
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.4.png)
По результатам сканирования видна текущая версия установленной ОС (Linux Debian 11), версия OpenSSH (8.4.p1), далее следует трудночитаемый вывод, из которого видно имя хоста (elk), установленный elasticsearsh с именем кластера и прочей информацией.

Переходим на атакуемый хост с запущенной утилитой Suricata и смотрим логи файла /var/log/suricata/fast.log
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.5.png)

По записям "Misc Attack", "Potentially bad traffic" понятно, что были попытки проникновений с номерами соответствующих портов и ip адресов. Вывод файла трудночитаем, поэтому анализируем информацию через стандартный дашборд, развернутый ранее в kibana.

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.7.png)
Графически информация гораздо более понятная, но явно избыточная, поэтому пробуем применить фильты, например на предмет каких-либо аномалий (suricata.eve.event_type: anomaly).
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.8.png)
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.9.png)
Теперь дашборд показывает, что было 4 попытки противоправных действий в 12:20, 12:31 и дважды в 12:38. Пакеты шли с ip адреса 31.173.87.88 (поскольку Kali развернута в VirtualBox с настройками соединения типа "сетевой мост", то отображаеися публичный ip адрес хостовой ОС) на хост 192.168.0.5 (внутренний ip атакуемого хоста в yandex cloud), все 4 атаки было из РФ (где я и нахожусь).
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network1.10.png)
По собранной информации можно провести более детальный анализ, так как в логах ее в избытке, однако пока остался открытым вопрос об ip адресах 31.173.169.254 и 169.254.169.254. Устройств с такими адресами я не нашел.

------

### Задание 2.

Проведение атаки на подбор пароля для службы SSH:

**hydra -L users.txt -P pass.txt < ip-адрес > ssh**

1. Настройка **hydra**: 
 
Cоздание файлов с именами пользователей и паролями: **users.txt** и **pass.txt**;
```
nano users.txt
user1
user2
max
leo
nic
root
user33
kali
```

```
nano pass.txt
321
default
12345
5051
321
111111111
ls2tdj3
0000
```

Проводим атаку на наш хост (в отличие от задания 1, публичный ip адрес уже другой, так как он динамический, но хост используется тот же).

Атака довольно быстро завершена, пароли для пользователей leo и root подобраны верно.
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network2.1.png)
Также сведения об атаке на порт 22 (SSH) можно увидеть в дашборде, который был использован при выполнении задания 1. В данном случае подозрительных активностей гораздо больше, чем при сканировании портов nmap.
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network2.3.png)
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network2.2.png)



2. Включение защиты SSH для Fail2Ban:
   
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network2.4.png)
Как видно, блокировка сработала, но не сразу. Попробуем откорректировать jail.local, сократив findtime до 5 секунд, а maxretry до 1.

![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network2.5.png)
На подбор пароля ушло еще больше времени, а исходя из анализа логов видно, что ip адрес, с которого шла атака был неоднократно заблокирован.
![alt text](https://github.com/LeonidKhoroshev/databases/blob/main/network/network2.6.png)



