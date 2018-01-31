# API ssl

Для ограничения доступа к `api` будет использоваться механизм `ssl`. 

> Рабочая папка `nginx-conf/ssl`.


См. ссылки по теме (некоторые довольно древние); ниже адаптированный конспект.  

[Авторизация клиентов в nginx посредством SSL сертификатов](https://habrahabr.ru/post/213741/)  
[nginx Настройка HTTPS-серверов](https://nginx.ru/ru/docs/http/configuring_https_servers.html)  
[Отозванные сертефикаты](https://jamielinux.com/docs/openssl-certificate-authority/certificate-revocation-lists.html)  
[Настраиваем HTTPS-сервер на nginx](https://habrahabr.ru/post/195808/)  
[Своё Certificate Authority — в 5 OpenSSL команд](https://habrahabr.ru/post/192446/)  
[Пример настройки двухфакторной аутентификации на ОС Linux](http://help.prognoz.com/ru/mergedProjects/Admin/03_admin/doubleauth/double_linux.htm)  

## Немного теории

Если на пальцах, то в случае использования клиентской аутентификации по `tls` есть две ветки `PKI`.  
- Сторона сервера (которая есть и без клиентской authN), которая включает root CA, intermediate CAs и серверный сертификат. 
При подключении клиента к серверу при установлении соединения сервер отдаёт свой сертификат с реверсной цепочкой не включая CA 
(т. к. он есть у пользователя). Файл с этими сертификатами в `nginx` указывается в параметре `ssl_certificate`.  
- Сторона клиента имеет независимую ветку `PKI` (которая может спокойно использовать приватный `CA`). 
При включенной `ssl_verify_client` (необходимо также указать `ssl_client_certificate`, если используется `on` или `optional`) 
сервер после `ServerHello` передаёт дополнительно фрейм `Certificate Request`, в котором перечислены `dn`'ы `CA` 
которым данный `endpoint` доверяет в контексте аутентификации клиента (в случае использования `optional_no_ca` этот список пустой). 
Клиент на этот запрос предоставляет сертификат (в случае браузера показывает окошко, где пользователь выбирает какой сертификат использовать).

И первая ветка `PKI` (серверная) в нормальном случае получается через публичный `CA` типа `LetsEncrypt`. Сторона же пользователя, в зависимости от сервиса может получаться:  
- от публичного или государственного `CA`,
- от приватного `CA` самого сервиса (который, по сути, в статье и описан).

[TSL](https://tls.dxdt.ru/tls.html)


## Шаг 1. Создание собственного самоподписанного доверенного сертификата. (Создание сертификатов Удостоверяющего Центра)


Собственный доверенный сертификат (Certificate Authority или CA) необходим для подписи  
- сертификта сервера для его прововерки клиентом;
- клиентских сертификатов и для их проверки при авторизации клиента веб-сервером.

#### Одной командой
```
openssl req -new -newkey rsa:2048 -nodes -keyout ca.key -x509 -days 10000 -subj /C=RU/ST=Moscow/L=Moscow/O=Companyname/OU=User/CN=etc/emailAddress=support@site.com -out ca.crt
```

Описание аргументов:

req Запрос на создание нового сертификата.
-new Создание запроса на сертификат (Certificate Signing Request – далее CSR).
-newkey rsa:2048 Автоматически будет создан новый закрытый RSA ключ длиной 2048 бита. Длину ключа можете настроить по своему усмотрению.
-nodes Не шифровать закрытый ключ.
-keyout ca.key Закрытый ключ сохранить в файл ca.key.
-x509 Вместо создания CSR (см. опцию -new) создать самоподписанный сертификат.
-days 500 Срок действия сертификата 500 дней. Размер периода действия можете настроить по своему усмотрению. Не рекомендуется вводить маленькие значения, так как этим сертификатом вы будете подписывать клиентские сертификаты.
-subj /C=RU/ST=Moscow/L=Moscow/O=Companyname/OU=User/CN=etc/emailAddress=support@site.com
Данные сертификата, пары параметр=значение, перечисляются через ‘/’. Символы в значении параметра могут быть «подсечены» с помощью обратного слэша «\», например «O=My\ Inc». Также можно взять значение аргумента в кавычки, например, -subj «/xx/xx/xx».

#### Альтернативный подход:

Первая команда создаёт корневой ключ
```
openssl genrsa -out rootCA.key 2048
```

Для меня ключ 2048 bit достаточен, если вам хочется, вы можете использовать ключ 4096 bit.

Вторая команда создаёт корневой сертификат.
```
openssl req -x509 -new -key rootCA.key -days 10000 -out rootCA.crt
```


Отвечать на вопросы тут можно как душе угодно. Формируются те самые `dn`'ы `CA`, которые в альтернативном методе указываются из `-subj`

Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:


10000 дней срок его годности, примерно столько живет сертификат, которым google требует подписывать андроид приложения для Google Play. Если вы паникер, подписывайте на год или два.

Все! Теперь мы можем создавать сертификаты для наших серверов и устанавливать корневой сертификат на наши клиентские машины.

## Шаг 2. Сертификат сервера


### Создание сертификата сервера

#### Конфигурационный файл 
`server.cnf`, пример:

```
[req]
prompt = no
distinguished_name = dn
req_extensions = ext

[dn]
C = RU
L = Moscow
O = Companyname
OU = User
CN = Сервер с аутентификацией клиентов
emailAddress = server.support@site.com

[ext]
subjectAltName = DNS:api.dev,DNS:*.api.dev,IP:0.0.0.0
```

> Важно! В `subjectAltName` должны быть указаны все используемые DNS-имена и IP-адреса. 
Либо следует явно задать сервер в `CN`
Иначе в конечном итоге при проверке получим
```sh
curl: (51) SSL: certificate subject name (dataFromServerCN) does not match target host name '0.0.0.0'
```

#### Создание запроса на сертификат

Создадим запрос на сертификат для `nginx` и, одновременно, `rsa` приватный ключ для запроса на сертификат/сертификат. 
При создании сертификата генерируется закрытый ключ, который используется для расшифровки данных, зашифрованных открытой 
частью сертификата. В целях безопасности ключ должен сохраняться в тайне и не передаваться третьим лицам.

Чтобы `nginx` при перезагрузке не спрашивал пароль, используем `-nodes`
```
openssl req -new -utf8 -nameopt multiline,utf8 -config server.cnf -newkey rsa:2048 -keyout server.key -nodes -out server.csr
```

#### Создание сертификата 

Подпишем сертификат нашей же собственной подписью (также используем данные секции `ext` файла настроек `server.cnf`)
```
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt -extfile server.cnf -extensions ext
```

Серийный номер - первый из выданных Удостоверяющим Центром используя доверенный сертификат (ca.crt)

### Подготовка конфига для клиентских сертификатов а также для списка отозванных сертификатов

Предварительно потребуется создать файл `openssl.cnf`, содержащий вспомогательную информацию:

```
[ ca ]
default_ca = CA_CLIENT # При подписи сертификатов # использовать секцию CA_CLIENT

[ CA_CLIENT ]
dir = ./db # Каталог для служебных файлов
certs = $dir/certs # Каталог для сертификатов
new_certs_dir = $dir/newcerts # Каталог для новых сертификатов

database = $dir/index.txt # Файл с базой данных подписанных сертификатов
serial = $dir/serial # Файл содержащий серийный номер сертификата (в шестнадцатеричном формате)
certificate = ./ca.crt # Файл сертификата CA
private_key = ./ca.key # Файл закрытого ключа CA

default_days = 365 # Срок действия подписываемого сертификата
default_crl_days = 7 # Срок действия CRL
default_md = md5 # Алгоритм подписи

policy = policy_anything # Название секции с описанием политики в отношении данных сертификата

[ policy_anything ]
countryName = optional # Поля optional - не обязательны, supplied - обязательны
stateOrProvinceName = optional
localityName = optional
organizationName = optional
organizationalUnitName = optional
commonName = optional
emailAddress = optional
```

Далее надо подготовить структуру каталогов и файлов, соответствующую описанной в конфигурационном файле

```sh
mkdir db
mkdir db/certs
mkdir db/newcerts
touch db/index.txt
echo "01" > db/serial
```

### Создание списка отозванных сертфикатов (CRL)

Изначально отозванных сертификатов нет. Команда общая для создания и для обновления. При первичной инициализации создаст "пустой" `CRL`.
> `openssl.cnf` используется из следующей главы - следует создать его и структуру каталогов базы.

```sh
$ openssl ca -config openssl.cnf -gencrl -out crl.pem
```

> Примечание: Секция `CRL OPTIONS` страницы документации `ca man` содержит больше информации о создании `CRL`ов.


Вы можете проверить содержание списка отозванных сертификатов (CRL) с помощью инструмента `openssl crl`
```sh
$ openssl crl -in crl.pem -noout -text
...
No certificates have been revoked yet, so the output will state No Revoked Certificates.
...
```

Вы должны регулярно повторно пересоздавать CRL. По умолчанию CRL устаревает через 30 дней. Это устанавливается опцией `default_crl_days` секции `[ CA_default ]`

# Автообновление списка отозванных сертификатов

```
touch /home/dev/projects/yii2advanced_rbac_crl_update
chmod +x /home/dev/projects/yii2advanced_rbac_crl_update
```
Скрипт
```
#!/bin/bash
cd /home/dev/projects/docker-yii2-app-advanced-rbac/nginx-conf/ssl
openssl ca -config openssl.cnf -gencrl -out crl.pem
/usr/local/bin/docker-compose -f /home/dev/projects/docker-yii2-app-advanced-rbac/docker-run/docker-compose.yml restart nginx
/usr/local/bin/docker-compose -f /home/dev/projects/docker-yii2-app-advanced-rbac/docker-compose.yml restart nginx
```

> Вниманме: Сведения из `crl.pem` получаюся `nginx` при чтении конфига. 
Следовательно после обовления `crl.pem` нужно перечитать конфиг.

crontab -e
```
* 11 */3 * 1 /home/dev/projects/yii2advanced_rbac_crl_update > /dev/null 2>&1
```



### Конфиг nginx
```conf
    listen 443;
    keepalive_timeout       70;

    ssl                     on;
    ssl_verify_client       on;
    ssl_certificate         /etc/nginx/conf.d/ssl/server.crt;
    ssl_certificate_key     /etc/nginx/conf.d/ssl/server.key;
    ssl_client_certificate  /etc/nginx/conf.d/ssl/ca.crt;
    ssl_crl                 /etc/nginx/conf.d/ssl/crl.pem; # Отозванные сертификаты (Certificate revocation lists)
    # Добавить в отозванные: 
    # пометить сертификат как отозванный
    # openssl ca -config openssl.cnf -revoke db/newcerts/01.pem
    # обновить список отозванных сертификатов
    # openssl ca -config openssl.cnf -gencrl -out crl.pem
    
    fastcgi_param SSL_VERIFIED $ssl_client_verify; #возвращает результат проверки клиентского сертификата: “SUCCESS”, “FAILED:reason” и, если сертификат не был предоставлен, “NONE”;
    fastcgi_param SSL_CLIENT_SERIAL $ssl_client_serial; #возвращает серийный номер клиентского сертификата для установленного SSL-соединения;
    fastcgi_param SSL_CLIENT_CERT $ssl_client_escaped_cert; #возвращает клиентский сертификат в формате PEM (закодирован в формате urlencode) для установленного SSL-соединения (1.13.5);
    fastcgi_param SSL_DN $ssl_client_i_dn; #возвращает строку “issuer DN” клиентского сертификата для установленного SSL-соединения согласно RFC 2253 (1.11.6);
```

Сертификат сервера является публичным. Он посылается каждому клиенту, соединяющемуся с сервером. Секретный ключ следует хранить в файле с ограниченным доступом (права доступа должны позволять главному процессу nginx читать этот файл). 

теперь сервер готов принимать запросы на `https`.
в переменных к бекенду появились переменные с информацией о сертификате, в первую очередь `SSL_VERIFIED` (принимает значение `SUCCESS`).

Однако если вы попытаетесь зайти на сайт 
(для `docker-compose -f docker-run/docker-compose.yml up -d` адрес будет `https://0.0.0.0:8082/`), 
он выдаст ошибку:
```
400 Bad Request
No required SSL certificate was sent
```

Что ж, логично, в этом-то и вся соль!

## Шаг 3. Создание клиентских сертификатов

### Создание клиентского закрытого ключа и запроса на сертификат (CSR)

Для создания подписанного клиентского сертификата предварительно необходимо создать запрос на сертификат, для его последующей подписи. 
Аргументы команды полностью аналогичны аргументам использовавшимся при создании самоподписанного доверенного сертификата, 
но отсутствует параметр `-x509`.
```sh
openssl req -new -newkey rsa:2048 -nodes -keyout client01.key -subj /C=RU/ST=Moscow/L=Moscow/O=Companyname/OU=User/CN=etc/emailAddress=support@site.com -out client01.csr
```

В результате выполнения команды появятся два файла `client01.key` и `client01.csr`.

### Подпись запроса на сертификат (CSR) с помощью доверенного сертификата (CA).

При подписи запроса используются параметры заданные в файле `openssl.cnf`
```sh
openssl ca -config openssl.cnf -in client01.csr -out client01.crt -batch
```

В результате выполнения команды появится файл клиентского сертификата `client01.crt`.

Для создания следующих сертификатов нужно повторять эти два шага.

> Также изменятся файлы в папке базы - `serial` получит инкремент (16-ричный серийник), 
> в `index.txt` появится новая запись вида
```
V 180921144337Z 01 unknown ... /CN=bob@example.com
```
> `client01.crt` также будет скопирован в новые сертификаты `nginx-conf/ssl/db/newcerts/01.pem`

Для проверки сертификата воспользуемся командой
```sh
$ openssl verify -CAfile ca.crt db/newcerts/01.pem
db/newcerts/01.pem: OK
```

### Массовое доавление в цикле
```sh
for (( i=01; i <= 10; i++ ));\
do printf -v c 'client%02d' $i;\
openssl req -new -newkey rsa:2048 -nodes -keyout $c.key -subj /C=RU/ST=Moscow/L=Moscow/O=Companyname/OU=User/CN=api$c/emailAddress=support@site.com -out $c.csr;\
openssl ca -config openssl.cnf -in $c.csr -out $c.crt -batch;\
done
```

### Отзыв сертификатов

Для отзыва сертификатов используется следующие команды `openssl ca` (рабочая папка `nginx-conf/ssl`):  
пометить сертификат как отозванный
```sh
$ openssl ca -config openssl.cnf -revoke db/newcerts/01.pem
Using configuration from openssl.cnf
Revoking Certificate 01.
Data Base Updated
```
Выполнить скрипт обновления списка отозванных сертификатов
```sh
sh /home/dev/projects/yii2advanced_rbac_crl_update
```
либо обновить список отозванных сертификатов и выполнить перезагрузку контейнера `nginx` 
```sh
$ openssl ca -config openssl.cnf -gencrl -out crl.pem
Using configuration from openssl.cnf
$ /usr/local/bin/docker-compose -f /home/dev/projects/docker-yii2-app-advanced-rbac/docker-compose.yml restart nginx
```
проверить список отозванных сертификатов
```sh
$ openssl crl -in crl.pem -noout -text
...
Revoked Certificates:
    Serial Number: 04
        Revocation Date: Sep 22 10:17:05 2017 GMT
...
```
Проверить сертификат после отзыва и обновления списка отозванных сертификатов
```sh
$ openssl verify -CAfile ca.crt db/newcerts/04.pem
```

[см. также](https://jamielinux.com/docs/openssl-certificate-authority/certificate-revocation-lists.html)
                                                                 

### Подключение к полученному `ssl` серверу с помощью `curl`
Тестовое подключение: 
- из корня проекта
```sh
$ curl -v --cacert nginx-conf/ssl/ca.crt --key nginx-conf/ssl/client01.key --cert nginx-conf/ssl/client01.crt --url https://0.0.0.0:8082/v1/feedback/create --request POST --data "name=tester&email=tester@gmail.com&subject=test&body=message body"
```
- из дирректории с сертификатами - `client01.crt`  + `client01.key` и `ca.crt`
```sh
$ curl -v --cacert ca.crt --key client01.key --cert client01.crt --url https://0.0.0.0:8082/v1/feedback/create --request POST --data "name=tester&email=tester@gmail.com&subject=test&body=message body"
```
### Примеры клиента

Получив положительный результат тестового подключения, можно попробовать [примеры php-клиентов](./example-api-ssl-client.md).

### Создание сертификата в формате `PKCS#12` для браузера клиента

Это на тот случай, если к вашему серверу подключаются не бездушные машины, как в моём случае, а живые люди через браузер.
Запароленный файл `PKCS#12` надо скормить браузеру, чтобы он смог посещать ваш сайт.
```sh
openssl pkcs12 -export -in client01.crt -inkey client01.key -certfile ca.crt -out client01.p12 -passout pass:q1w2e3
```

### Конвертирование в другие форматы

[Форматы сертификатов](./about-ssl-cert-form.md)  

Для других вариантов создания подключения 
(например `php`: `stream_context_create` + `fopen`; `arduino esp8266` : `WiFiClientSecure.connect`) 
может понадобится конвертация ключей:

```sh
$ openssl x509 -in ca.crt -out ca.crt.der -outform DER 
$ openssl x509 -in client01.crt -out client01.crt.der -outform DER 
$ openssl rsa -in client01.key -out client01.key.der -outform DER
```

`-outform` выбирайте соответственно потребностям.

### Итоги
В результате были созданы:

- корневой сертификат - `ca.crt`;

- сертификат сервера - `server.crt`;

- сертификат клиента и ключ клиента `client01.crt` + `client01.key` / сертификат клиента с зашифрованным ключом - `client01.p12`.

Сертификаты необходимо разместить следующим образом:

- на сервере: корневой сертификат и серверный сертификат с ключом (`ca.crt`, `server.crt` и `server.key`);

- на клиенте: клиентский сертификат `client01.crt`, секретный ключ клиента `client01.key`, корневой сертификат без ключа `ca.crt`

- на клиенте с доступом из браузера: клиентский сертификат с зашифрованным ключом в формате *.p12 и корневой сертификат без ключа.
    Клиентский сертификат устанавливается в личное хранилище пользователя, корневой сертификат импортируется в хранилище 
    доверенных корневых центров сертификации локального компьютера.
