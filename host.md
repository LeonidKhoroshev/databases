# Домашнее задание к занятию  «Защита хоста» - Леонид Хорошев

------

### Задание 1

1. Установите **eCryptfs**.
Домашнее задание выполняется на ВМ Debian 11, развернутой в Virtualbox. 
```
apt update
apt upgrade
apt install ecryptfs-utils
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host1.1.png)

2. Добавьте пользователя cryptouser.

В Debian 11 отсутствует опция --encrypt-home, которая при использовании команды  adduser создает зашифрованный каталог для нового пользователя, поэтому задание выполняется в несколько этапов.
```
sudo adduser cryptouser
usermod -aG sudo cryptouser
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host1.2.png)

3. Зашифруйте домашний каталог пользователя с помощью eCryptfs.

Для шифрования каталога добавляем соответствующий модуль ядра и устанавливаем утилиту rsync (необходима для копирования и синхронизации данных)
```
apt install rsync
modprobe ecryptfs
su cryptouser
sudo ecryptfs-migrate-home -u cryptouser
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host1.3.png)

Переходим в домашний каталог под логином нового пользователя, создаем несколько файлов и смотрим содержимое домашнего каталога.
```
cd cryptouser
sudo touch 123 456
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host1.4.png)
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host1.5.png)

Далее перелогиниваемся под  другого пользователя (в нашем случае root) и смотрим содержимое каталога.
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host1.6.png)


### Задание 2

1. Установите поддержку **LUKS**.
```
apt update
apt upgrade
apt install ecryptfs-utils cryptsetup
```

2. Создайте небольшой раздел, например, 100 Мб.

Используем утилиту fdisk
```
fdisk /dev/sda2
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host2.1.png)

Проверяем созданный раздел
```
sudo fdisk -l /dev/sda5
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host2.2.png)

3. Зашифруйте созданный раздел с помощью LUKS.

Готовим соответствующий раздел sda5
```
sudo cryptsetup -y -v --type luks2 luksFormat /dev/sda5
```
Монтируем раздел
```
sudo cryptsetup luksOpen /dev/sda5 disk
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host2.3.png)

Форматируем созданный раздел
```
sudo dd if=/dev/zero of=/dev/mapper/disk
sudo mkfs.ext4 /dev/mapper/disk
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host2.4.png)

Монтируем раздел и завершаем работу
```
mkdir .secret
sudo mount /dev/mapper/disk .secret/
sudo umount .secret
sudo cryptsetup luksClose disk
```

Для проверки шифрования перейдем в GUI и запустим утилиту gparted.

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host2.5.png)


### Задание 3 *

1. Установите **apparmor**.
```
sudo apt install apparmor-profiles apparmor-utils
```
![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host2.6.png)

2. Повторите эксперимент, указанный в лекции.

Эксперимент, показанный в лекции представляет собой замену команды man  командой ping

Первым действием мы делаем резервную копию утилиты man, затем "подменяем" команду man командой ping копированием соответствующего бинарного файла.
```
sudo cp /usr/bin/man /usr/bin/man1
sudo cp /bin/ping /usr/bin/man
```
Далее сталкиваемся с поведением системы, возникшем в ходе лекции (apparmor блокирует выполнение команды man)

4. Отключите (удалите) apparmor.

При отключении apparmor эксперимент завершается успехом.

![Alt text](https://github.com/LeonidKhoroshev/databases/blob/main/host/host2.7.png)




