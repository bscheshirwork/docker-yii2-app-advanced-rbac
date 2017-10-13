# Примеры клиентов для использования с api по ssl

## php: fopen (аналог StreamTransport)

Отправляет HTTP-сообщения, используя [потоки](http://php.net/manual/ru/book.stream.php).
Он не требует каких-либо установленных дополнительных PHP расширений или библиотек

```php
        // из контейнера можно обратится по внутреннему DN и алиасу
        // порты внутренние.
        // добавлены алиасы для внутренней сети докера - api.dev -> nginx
        $url = 'https://api.dev:8084/v1/feedback/create';
        $postContent = json_encode([
            'name' => 'test stream',
            'email' => 'tester@gmail.com',
            'subject' => 'test stream json_encode',
            'body' => 'message body кирилица += ! ',
        ]);

        $contextOptions = [
            'http' => [
                'protocol_version' => '1.1',
                'method' => 'POST',
                'ignore_errors' => true,
                //Извлечь содержимое даже в случае, если присутствует код статуса неуспешного завершения
                'headers' => [
                    'Content-Type: application/json; charset=UTF-8',
                    'Host: api.dev',
                ],
                'content' => $postContent,
            ],
            'ssl' => [
                'verify_peer' => true,
                'allow_self_signed' => true,
                'cafile' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'ca.crt',
                // сертификат CA. PEM сертификаты могут иметь расширение .pem, .crt, .cer, и .key (файл приватного ключа)
                // Расположение файла сертификата в локальной файловой системе, который следует использовать с опцией контекста verify_peer для проверки подлинности удалённого узла.
                'local_cert' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.crt',
                // сертификат клиента
                // Путь к локальному сертификату в файловой системе. Это должен быть файл, закодированный PEM, который содержит ваш сертификат и приватный ключ. Он дополнительно может содержать публичный ключ эмитента. Приватный ключ также может содержаться в отдельном файле, заданным local_pk.
                'local_pk' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.key',
                // ключ клиента
                // Путь к локальному файлу с приватным ключем в случае отдельных файлов сертификата (local_cert) и приватного ключа.
                // 'passphrase' => 'client01 crt example pass phrase',
                // Идентификационная фраза, с которой ваш файл local_cert был закодирован.
            ],
        ];
        try {
            $context = stream_context_create($contextOptions);
            $stream = fopen($url, 'rb', false, $context);
            $responseContent = stream_get_contents($stream);
            // see http://php.net/manual/ru/reserved.variables.httpresponseheader.php
            $responseHeaders = $http_response_header;
            fclose($stream);
            var_dump($responseContent, $responseHeaders);
            switch ($responseHeaders[0] ?? false) {
                case 'HTTP/1.1 400 Bad Request':
                    echo "Запрос сформулирован неверно...\n";
                    break;
                case 'HTTP/1.1 204 No Content':
                    echo "Ok\n";
                    break;
                case 'HTTP/1.1 422 Data Validation Failed.':
                    echo "Данные не прошли проверку...\n";
                    break;
                default:
                    break;
            }
        } catch (\Exception $e) {
            //Yii::$app->errorHandler->handleException($e);
            //Exception 'Exception' with message 'fopen(https://0.0.0.0:8084/v1/feedback/create): failed to open stream: Connection refused'
            echo $e->getMessage(), $e->getCode(), "\n";
        }
```

## php cURL (аналог CurlTransport)
Отправляет HTTP-сообщения, используя [cURL](http://php.net/manual/ru/book.curl.php)
Для этого требуется установленное PHP расширение 'curl'

```php
        // из контейнера можно обратится по внутреннему DN и алиасу
        // порты внутренние.
        // добавлены алиасы для внутренней сети докера - api.dev -> nginx
        $url = 'https://api.dev:8084/v1/feedback/create';
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL,
            $url); // Загружаемый URL. Данный параметр может быть также установлен при инициализации сеанса с помощью curl_init().
        curl_setopt($ch, CURLOPT_FAILONERROR,
            true); // TRUE для подробного отчета при неудаче, если полученный HTTP-код больше или равен 400. Поведение по умолчанию возвращает страницу как обычно, игнорируя код.
        curl_setopt($ch, CURLOPT_CERTINFO,
            true); // TRUE для вывода информации о сертификате SSL в поток STDERR при безопасных соединениях.    Добавлена в cURL 7.19.1. Доступна, начиная с версии PHP 5.3.2. Для корректной работы требует включенной опции CURLOPT_VERBOSE.
        curl_setopt($ch, CURLOPT_HEADER, true); // TRUE для включения заголовков в вывод.
        curl_setopt($ch, CURLINFO_HEADER_OUT, true); // TRUE для отслеживания строки запроса дескриптора.
        curl_setopt($ch, CURLOPT_RETURNTRANSFER,
            true); // TRUE для возврата результата передачи в качестве строки из curl_exec() вместо прямого вывода в браузер.
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER,
            true); // FALSE для остановки cURL от проверки сертификата узла сети. Альтернативные сверяемые сертификаты могут быть указаны с помощью параметра CURLOPT_CAINFO или директории с сертификатами, указываемой параметром CURLOPT_CAPATH.
        curl_setopt($ch, CURLOPT_SSL_VERIFYSTATUS,
            false); // TRUE для проверки статуса сертификата.     Доступно с PHP 7.0.7. (требует использования OCSP, не работает с crl)
        curl_setopt($ch, CURLOPT_VERBOSE,
            true); // TRUE для вывода дополнительной информации. Записывает вывод в поток STDERR, или файл, указанный параметром CURLOPT_STDERR.
        //curl_setopt($ch, CURLOPT_PORT, 8084); // Альтернативный порт соединения.
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST,
            2); // Используйте 2 для проверки существования общего имени сертификате SSL, а также его совпадения с указанным хостом. 0 чтобы не проверять имена. В боевом окружении значение этого параметра должно быть 2 (установлено по умолчанию).
        curl_setopt($ch, CURLOPT_SSH_AUTH_TYPES,
            CURLSSH_AUTH_ANY); // Битовая маска, состоящая из одной или более констант: CURLSSH_AUTH_PUBLICKEY, CURLSSH_AUTH_PASSWORD, CURLSSH_AUTH_HOST, CURLSSH_AUTH_KEYBOARD. Установите CURLSSH_AUTH_ANY, чтобы libcurl сам выбрал одну из них.
        curl_setopt($ch, CURLOPT_CAINFO,
            Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'ca.crt'); // Имя файла, содержащего один или более сертификатов, с которыми будут сверяться узлы. Этот параметр имеет смысл только при использовании совместно с CURLOPT_SSL_VERIFYPEER. 	Может потребоваться абсолютный путь.
        curl_setopt($ch, CURLOPT_SSLKEY,
            Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.key'); // Имя файла с закрытым ключом SSL.
        curl_setopt($ch, CURLOPT_SSLCERT,
            Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.crt'); // Имя файла с корректно отформатированным PEM-сертификатом.  PEM сертификаты могут иметь расширение .pem, .crt, .cer, и .key (файл приватного ключа)
        curl_setopt($ch, CURLOPT_SSLCERTTYPE,
            'PEM'); // Формат сертификата. Поддерживаются форматы "PEM" (по умолчанию), "DER" и "ENG". 	Добавлен в версии cURL 7.9.3.
        //curl_setopt($ch, CURLOPT_SSLCERTPASSWD, 'client01 crt example pass phrase'); // Пароль, необходимый для использования сертификата CURLOPT_SSLCERT. / Идентификационная фраза, с которой ваш файл CURLOPT_SSLCERT был закодирован.
        //curl_setopt($ch, CURLOPT_WRITEHEADER, ); // Файл, в который будут записаны заголовки текущей операции.
        curl_setopt($ch, CURLOPT_REFERER,
            ''); // Содержимое заголовка "Referer: ", который будет использован в HTTP-запросе.
        curl_setopt($ch, CURLOPT_USERAGENT,
            'API клиент'); // Содержимое заголовка "User-Agent: ", посылаемого в HTTP-запросе.
        curl_setopt($ch, CURLOPT_HTTPHEADER,
            [
                'Content-type: application/json',
                'Host: api.dev',
            ]); // Массив устанавливаемых HTTP-заголовков, в формате ['Content-type: text/plain', 'Content-length: 100']
        curl_setopt($ch, CURLOPT_POSTFIELDS,
            $postContent); // Все данные, передаваемые в HTTP POST-запросе. Для передачи файла, укажите перед именем файла @, а также используйте полный путь к файлу. Тип файла также может быть указан с помощью формата ';type=mimetype', следующим за именем файла. Этот параметр может быть передан как в качестве url-закодированной строки, наподобие 'para1=val1&para2=val2&...', так и в виде массива, ключами которого будут имена полей, а значениями - их содержимое. Передача массива в CURLOPT_POSTFIELDS закодирует данные в виде multipart/form-data, тогда как передача URL-кодированной строки закодирует данные в виде application/x-www-form-urlencoded. Начиная с версии PHP 5.2.0, при передаче файлов с префиксом @, value должен быть массивом. С версии PHP 5.5.0, префикс @ устарел и файлы можно отправлять с помощью CURLFile. Префикс @ можно отключить, чтобы можно было передавать значения, начинающиеся с @, задав опцию CURLOPT_SAFE_UPLOAD в значение TRUE.
        curl_setopt($ch, CURLOPT_TIMEOUT, 10);

        $rawHTML = curl_exec($ch);
        if ($errno = curl_errno($ch)) {
            $error_message = curl_strerror($errno);
            echo "cURL error ({$errno}):\n {$error_message}";
        }

        curl_close($ch);
        $arrHTML = preg_split('/\r\n/', $rawHTML);
        var_dump($arrHTML);
        echo "\n";
        echo 'HTML: ' . $rawHTML, "\n";

        switch (($arrHTML ?? [])[0] ?? false) {
            case 'HTTP/1.1 400 Bad Request':
                echo "Запрос сформулирован неверно...\n";
                break;
            case 'HTTP/1.1 204 No Content':
                echo "Ok\n";
                break;
            case 'HTTP/1.1 422 Data Validation Failed.':
                echo "Данные не прошли проверку...\n";
                break;
            default:
                break;
        }
```

## yiisoft/yii2-httpclient
Для приложений на базе `yii2`. Требует установки [yiisoft/yii2-httpclient](https://github.com/yiisoft/yii2-httpclient) (В некоторых случаях уже используется, т.к. упомянуто в зависимостях)

```php
        $client = new Client([
            'requestConfig' => [
                'format' => Client::FORMAT_JSON,
            ],
            'responseConfig' => [
                'format' => Client::FORMAT_JSON,
            ],
        ]);

        $response = $client->createRequest()
            ->setMethod('post')
            ->setUrl('https://api.dev:8084/v1/feedback/create')
            ->setData(['name' => 'test yii2-httpclient', 'email' => 'tester@gmail.com', 'subject' => 'test yii2-httpclient', 'body' => 'message body кирилица += ! '])
            ->setOptions([
                // 'ssl' . Inflector::underscore
                'sslVerifyPeer' => true,
                // not sslCaFile!
                'sslCafile' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'ca.crt',
                // сертификат CA. PEM сертификаты могут иметь расширение .pem, .crt, .cer, и .key (файл приватного ключа)
                // Расположение файла сертификата в локальной файловой системе, который следует использовать с опцией контекста verify_peer для проверки подлинности удалённого узла.
                'sslLocalCert' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.crt',
                // сертификат клиента
                // Путь к локальному сертификату в файловой системе. Это должен быть файл, закодированный PEM, который содержит ваш сертификат и приватный ключ. Он дополнительно может содержать публичный ключ эмитента. Приватный ключ также может содержаться в отдельном файле, заданным local_pk.
                'sslLocalPk' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.key',
                // ключ клиента
                // Путь к локальному файлу с приватным ключем в случае отдельных файлов сертификата (local_cert) и приватного ключа.
                // 'sslPassphrase' => 'client01 crt example pass phrase',
                // Идентификационная фраза, с которой ваш файл local_cert был закодирован.
            ])
            ->send();
        var_dump($response);

        $client = new Client([
            'transport' => 'yii\httpclient\CurlTransport',
            'requestConfig' => [
                'format' => Client::FORMAT_JSON,
            ],
            'responseConfig' => [
                'format' => Client::FORMAT_JSON,
            ],
        ]);
        $response = $client->createRequest()
            ->setMethod('post')
            ->setUrl('https://api.dev:8084/v1/feedback/create')
            ->setData(['name' => 'test yii2-httpclient curl', 'email' => 'tester@gmail.com', 'subject' => 'test yii2-httpclient curl', 'body' => 'message body кирилица += ! '])
            ->setOptions([
                // 'ssl' . Inflector::underscore
                'sslVerifyPeer' => true,
                // not sslCaFile!
                'sslCafile' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'ca.crt',
                // сертификат CA. PEM сертификаты могут иметь расширение .pem, .crt, .cer, и .key (файл приватного ключа)
                // Расположение файла сертификата в локальной файловой системе, который следует использовать с опцией контекста verify_peer для проверки подлинности удалённого узла.
                'sslLocalCert' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.crt',
                // сертификат клиента
                // Путь к локальному сертификату в файловой системе. Это должен быть файл, закодированный PEM, который содержит ваш сертификат и приватный ключ. Он дополнительно может содержать публичный ключ эмитента. Приватный ключ также может содержаться в отдельном файле, заданным local_pk.
                'sslLocalPk' => Yii::$app->basePath . DIRECTORY_SEPARATOR . 'config' . DIRECTORY_SEPARATOR . 'client01.key',
                // ключ клиента
                // Путь к локальному файлу с приватным ключем в случае отдельных файлов сертификата (local_cert) и приватного ключа.
                // 'sslPassphrase' => 'client01 crt example pass phrase',
                // Идентификационная фраза, с которой ваш файл local_cert был закодирован.
            ])
            ->send();
        var_dump($response);
```
