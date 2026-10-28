# Nginx Proxy SSL Auth Example (mTLS)

Демо взаимной TLS-аутентификации (mTLS) на двух контейнерах nginx:

- **webserver** отдаёт страницу только тем клиентам, которые предъявили сертификат, подписанный доверенным CA;
- **proxy** держит такой клиентский сертификат и предъявляет его webserver'у при каждом запросе, а сам
  проверяет сертификат webserver.

В итоге напрямую к webserver без сертификата не пустят, а через proxy пустят.

## Что такое mTLS

В обычном TLS (как на любом HTTPS-сайте) сертификат предъявляет только **сервер**. Клиент проверяет, что
сервер тот, за кого себя выдаёт, но сам остаётся анонимным для TLS-слоя: кто он, сервер узнаёт уже на уровне
приложения (логин/пароль, токен и т.п.).

В **mutual TLS** сертификат предъявляют **обе стороны**. Сервер во время рукопожатия запрашивает у клиента
сертификат и проверяет, что тот подписан доверенным центром сертификации (CA). Не прошёл проверку — соединение
отклоняется ещё до того, как HTTP-запрос дошёл до приложения.

Чтобы клиент мог предъявить сертификат, ему мало одного сертификата: нужен ещё **закрытый ключ** к нему.
Во время рукопожатия клиент подписывает этим ключом данные сессии и тем доказывает, что ключ у него есть.
Поэтому украденный сертификат без ключа бесполезен.

## Кто кого проверяет

- **webserver проверяет proxy.** proxy предъявляет `proxy-client.crt`, webserver проверяет, что он подписан
  `client-ca.crt`. Нет сертификата или он чужой — `400 Bad Request`.
- **proxy проверяет webserver.** webserver предъявляет `webserver.crt`, proxy сверяет его с той же копией
  `webserver.crt`, которой доверяет. Сертификат подменён — `502 Bad Gateway`.
- **curl проверяет proxy** по `proxy.crt` (`--cacert`).

## Создание сертификатов

Сертификаты в репозиторий не входят, их нужно сгенерировать самостоятельно. Все команды выполняются в корне
репозитория, имена файлов совпадают с теми, что монтируются в [docker-compose.yml](docker-compose.yml).

Используется **OpenSSL 3.x** (`openssl version`). LibreSSL не подойдёт: он не поддерживает опции `-CA`/`-CAkey`
у `openssl req`.

Каждая команда создаёт ключ и сертификат сразу, без CSR и без файлов конфигурации.

**1. CA для клиентской аутентификации.** Им подписываются клиентские сертификаты, ему доверяет webserver.

```console
openssl req -x509 -newkey rsa:4096 -nodes -keyout client-ca.key -out client-ca.crt -days 3650 -subj "/CN=Demo mTLS CA"
```

**2. Клиентский сертификат для proxy**, подписанный этим CA.

```console
openssl req -x509 -newkey rsa:4096 -nodes -keyout proxy-client.key -out proxy-client.crt -days 365 -subj "/CN=proxy-client" -CA client-ca.crt -CAkey client-ca.key -addext "basicConstraints=critical,CA:FALSE"
```

`basicConstraints=critical,CA:FALSE` запрещает использовать сертификат как CA, то есть подписывать им другие
сертификаты. Без него `openssl req -x509` по умолчанию пометит сертификат как CA. `critical` означает, что
программа, которая не понимает это расширение, обязана отклонить сертификат, а не проигнорировать ограничение.

**3. Серверный сертификат webserver** (самоподписанный).

```console
openssl req -x509 -newkey rsa:4096 -nodes -keyout webserver.key -out webserver.crt -days 365 -subj "/CN=webserver" -addext "subjectAltName=DNS:webserver,DNS:localhost"
```

`subjectAltName` перечисляет имена хостов, для которых действует сертификат. Клиент сверяет с этим списком
адрес, к которому подключился: proxy обращается к `webserver`, а curl в примерах ниже к `localhost`.

**4. Серверный сертификат proxy** (самоподписанный).

```console
openssl req -x509 -newkey rsa:4096 -nodes -keyout proxy.key -out proxy.crt -days 365 -subj "/CN=proxy" -addext "subjectAltName=DNS:proxy,DNS:localhost"
```

`subjectAltName` здесь тот же по смыслу: curl подключается к proxy по адресу `localhost`.

**Проверка.** Клиентский сертификат должен проходить проверку по CA:

```console
~$ openssl verify -CAfile client-ca.crt proxy-client.crt
proxy-client.crt: OK
~$ openssl x509 -in proxy-client.crt -noout -subject -issuer -enddate
subject=CN=proxy-client
issuer=CN=Demo mTLS CA
notAfter=...
```

Закрытые ключи лежат незашифрованными (`-nodes`), поэтому коммитить их нельзя: `*.crt` и `*.key` уже
указаны в [.gitignore](.gitignore).

## Запуск

```console
docker compose up -d
docker compose ps
```

Поднимутся два контейнера:

- `nginx_webserver` на https://localhost:8443, требует клиентский сертификат;
- `nginx_proxy` на https://localhost:9443, проксирует запросы на webserver с клиентским сертификатом.

Остановить:

```console
docker compose down
```

## Проверка

Во всех примерах используется `--cacert`: curl проверяет серверный сертификат по нашему
самоподписанному файлу.

### Как работает

**Через proxy: доступ есть.** proxy сам предъявляет webserver'у клиентский сертификат.

```console
~$ curl --cacert proxy.crt https://localhost:9443/
<html>
    <h1>Test webserver</h1>
</html>
```

**Напрямую с клиентским сертификатом: доступ есть.** Пускает именно сертификат, а не proxy: curl с тем же
сертификатом и ключом проходит так же.

```console
~$ curl --cacert webserver.crt --cert proxy-client.crt --key proxy-client.key https://localhost:8443/
<html>
    <h1>Test webserver</h1>
</html>
```

### Как не работает

**Напрямую без сертификата: отказ.**

```console
~$ curl --cacert webserver.crt https://localhost:8443/
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.29.3</center>
</body>
</html>
```

**С сертификатом от чужого CA: отказ.** Сгенерируем самоподписанный сертификат, которому webserver не доверяет:

```console
~$ openssl req -x509 -newkey rsa:2048 -nodes -keyout fake.key -out fake.crt -days 1 -subj "/CN=fake"
~$ curl --cacert webserver.crt --cert fake.crt --key fake.key https://localhost:8443/
<html>
<head><title>400 The SSL certificate error</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>The SSL certificate error</center>
<hr><center>nginx/1.29.3</center>
</body>
</html>
~$ rm fake.key fake.crt
```

Ответ `400 The SSL certificate error` придёт и тогда, когда у клиентского сертификата истёк срок действия.

**Подменённый webserver: отказ на стороне proxy.** proxy проверяет сертификат webserver
(`proxy_ssl_verify on`). Остановим настоящий webserver и запустим вместо него поддельный: тот же сервис под
тем же сетевым именем `webserver`, но со своим сертификатом. Имя в сертификате совпадает, отличается только
подпись.

```console
~$ openssl req -x509 -newkey rsa:2048 -nodes -keyout evil.key -out evil.crt -days 1 -subj "/CN=evil" -addext "subjectAltName=DNS:webserver"
~$ docker compose stop webserver
~$ docker compose run -d --name fake_webserver --use-aliases -v ./evil.crt:/etc/nginx/certs/webserver.crt:ro -v ./evil.key:/etc/nginx/certs/webserver.key:ro webserver
~$ docker compose restart proxy
```

`restart proxy` нужен потому, что nginx определяет IP-адрес `webserver` один раз при старте. Без перезапуска
proxy продолжит стучаться на старый адрес.

proxy доверяет только настоящему сертификату webserver, поэтому обрывает соединение с подделкой и отвечает 502:

```console
~$ curl --cacert proxy.crt https://localhost:9443/
<html>
<head><title>502 Bad Gateway</title></head>
...
~$ docker compose logs proxy | grep upstream
... upstream SSL certificate verify error: (18:self-signed certificate) while SSL handshaking to upstream ...
```

Вернуть всё как было:

```console
~$ docker rm -f fake_webserver && rm evil.crt evil.key
~$ docker compose up -d && docker compose restart proxy
```

**Без доверия к серверному сертификату: отказ на стороне клиента.** Это обычная проверка сервера, она есть и
в простом TLS:

```console
~$ curl https://localhost:9443/
curl: (60) SSL certificate problem: self signed certificate
```

### Что сервер просит у клиента

`openssl s_client` показывает, сертификат какого CA webserver ожидает от клиента
(сообщение CertificateRequest в рукопожатии):

```console
~$ openssl s_client -connect localhost:8443 -CAfile webserver.crt </dev/null 2>/dev/null | grep -A1 "Acceptable client"
Acceptable client certificate CA names
CN=Demo mTLS CA
```
