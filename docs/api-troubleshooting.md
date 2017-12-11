# Решение проблем с API

### Автообновление списка отозванных сертификатов

Ошибки вида 
```html
<html>

<head><title>400 The SSL certificate error</title></head>

<body bgcolor="white">

<center><h1>400 Bad Request</h1></center>

<center>The SSL certificate error</center>

<hr><center>nginx/1.13.7</center>

</body>

</html>
```
И соответствующие им сообщения клиента могут воникать при неотработке скрипта обновления `crl`  
Причина: отсутствие запущенных контейнеров на момент выполнения скрипта по расписанию
либо ошибка в настройке расписания.  
Решение: запустить скрипт обновления вручную 
```
sh /home/dev/projects/yii2advanced_rbac_crl_update
```
и проверить расписание `crontab -l`. Предлагаемое значение `* 11 */3 * 1 ` с запасом перекрывает семидневный интервал.

см. раздел ["Автообновление списка отозванных сертификатов" тут](about-api-ssl.md)

### Отзыв сертификата

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

см. раздел ["Отзыв сертификатов" тут](about-api-ssl.md)

### Создание новых сертификатов

Для создания подписанного клиентского сертификата предварительно необходимо создать запрос на сертификат, для его последующей подписи. 
```sh
openssl req -new -newkey rsa:2048 -nodes -keyout client01.key -subj /C=RU/ST=Moscow/L=Moscow/O=Companyname/OU=User/CN=etc/emailAddress=support@site.com -out client01.csr
```

В результате выполнения команды появятся два файла `client01.key` и `client01.csr`.

Подпись запроса на сертификат (CSR) с помощью доверенного сертификата (CA):

(при подписи запроса используются параметры заданные в файле `openssl.cnf`)
```sh
openssl ca -config openssl.cnf -in client01.csr -out client01.crt -batch
```

см. раздел ["Шаг 3. Создание клиентских сертификатов" тут](about-api-ssl.md)