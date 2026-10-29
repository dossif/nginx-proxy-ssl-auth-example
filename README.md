# Nginx Proxy SSL Auth Example (mTLS)

Демонстрация взаимной TLS-аутентификации (mTLS) на двух контейнерах nginx. Сервис webserver обслуживает
только тех клиентов, которые предъявили сертификат, выпущенный доверенным удостоверяющим центром (CA).
Сервис proxy предъявляет такой сертификат при обращении к webserver. Таким образом, прямой доступ к webserver
без сертификата невозможен, а доступ через proxy разрешён.

## Принцип mTLS

В классическом TLS сертификат предъявляет только сервер. В mutual TLS аутентификация двусторонняя: в ходе
рукопожатия сервер запрашивает сертификат клиента и проверяет, что он подписан доверенным CA. При неуспешной
проверке соединение отклоняется до начала обмена на уровне HTTP. Владение сертификатом клиент подтверждает
подписью закрытым ключом, поэтому сертификат без соответствующего ключа непригоден для аутентификации.

## Схема проверок

| Проверяющая сторона | Проверяемый сертификат      | Доверенный CA      | Результат отказа  |
|---------------------|-----------------------------|--------------------|-------------------|
| webserver           | proxy (`mtls-proxy.crt`)    | `mtls-ca.crt`      | `400 Bad Request` |
| proxy               | webserver (`webserver.crt`) | `webserver-ca.crt` | `502 Bad Gateway` |
| curl                | proxy (`proxy.crt`)         | `proxy-ca.crt`     | `curl: (60)`      |

Доверие устанавливается к CA, а не к отдельному сертификату. Это позволяет перевыпускать сертификаты и
использовать несколько экземпляров webserver без изменения конфигурации proxy: принимается любой сертификат,
подписанный `webserver-ca`. Каждой роли назначен отдельный CA, что исключает смешение ролей: например,
сертификат одного webserver не может быть использован как клиентский для другого.

> В демонстрации `proxy-ca` моделирует публичный удостоверяющий центр. В промышленной эксплуатации
> сертификат proxy следует получать у публичного УЦ, например [Let's Encrypt](https://letsencrypt.org/);
> в этом случае параметр `--cacert` не требуется. `mtls-ca` и `webserver-ca` являются внутренними УЦ,
> и использование собственного УЦ для них допустимо и в промышленной среде.

## Выпуск сертификатов

Требуется **OpenSSL 3.x**: LibreSSL не поддерживает параметры `-CA`/`-CAkey` команды `openssl req`. Команды
выполняются в корне репозитория. Закрытые ключи не зашифрованы (`-nodes`), поэтому файлы `*.crt` и `*.key`
исключены из репозитория через [.gitignore](.gitignore).

CA и клиентский сертификат proxy для mTLS-аутентификации на webserver:

```console
openssl req -x509 -newkey rsa:4096 -nodes -keyout mtls-ca.key -out mtls-ca.crt -days 3650 -subj "/CN=Demo mTLS CA"
openssl req -x509 -newkey rsa:4096 -nodes -keyout mtls-proxy.key -out mtls-proxy.crt -days 365 -subj "/CN=mtls-proxy" -CA mtls-ca.crt -CAkey mtls-ca.key
```

CA и серверный сертификат webserver:

```console
openssl req -x509 -newkey rsa:4096 -nodes -keyout webserver-ca.key -out webserver-ca.crt -days 3650 -subj "/CN=Demo Webserver CA"
openssl req -x509 -newkey rsa:4096 -nodes -keyout webserver.key -out webserver.crt -days 365 -subj "/CN=webserver" -CA webserver-ca.crt -CAkey webserver-ca.key -addext "subjectAltName=DNS:webserver,DNS:localhost"
```

CA и серверный сертификат proxy:

```console
openssl req -x509 -newkey rsa:4096 -nodes -keyout proxy-ca.key -out proxy-ca.crt -days 3650 -subj "/CN=Demo Proxy CA"
openssl req -x509 -newkey rsa:4096 -nodes -keyout proxy.key -out proxy.crt -days 365 -subj "/CN=proxy" -CA proxy-ca.crt -CAkey proxy-ca.key -addext "subjectAltName=DNS:proxy,DNS:localhost"
```

Расширение `subjectAltName` задаёт имена хостов, для которых действителен серверный сертификат: proxy
обращается к `webserver`, curl — к `localhost`.

Проверка цепочек доверия:

```console
~$ openssl verify -CAfile mtls-ca.crt mtls-proxy.crt
mtls-proxy.crt: OK
~$ openssl verify -CAfile webserver-ca.crt webserver.crt
webserver.crt: OK
~$ openssl verify -CAfile proxy-ca.crt proxy.crt
proxy.crt: OK
```

## Запуск

```console
docker compose up -d
```

Сервис webserver доступен по адресу https://localhost:8443, proxy — по адресу https://localhost:9443.

Остановка:

```console
docker compose down
```

## Проверка

### Успешные сценарии

Запрос через proxy:

```console
~$ curl --cacert proxy-ca.crt https://localhost:9443/
<html>
    <h1>Test webserver</h1>
</html>
```

Прямой запрос к webserver с клиентским сертификатом. Доступ определяется сертификатом, а не источником
запроса:

```console
~$ curl --cacert webserver-ca.crt --cert mtls-proxy.crt --key mtls-proxy.key https://localhost:8443/
<html>
    <h1>Test webserver</h1>
</html>
```

Замена webserver экземпляром с другим сертификатом, выпущенным тем же CA. Конфигурация proxy не изменяется:

```console
~$ openssl req -x509 -newkey rsa:4096 -nodes -keyout webserver2.key -out webserver2.crt -days 365 -subj "/CN=webserver2" -CA webserver-ca.crt -CAkey webserver-ca.key -addext "subjectAltName=DNS:webserver"
~$ docker compose stop webserver
~$ docker compose run -d --name webserver2 --use-aliases -v ./webserver2.crt:/etc/nginx/certs/webserver.crt:ro -v ./webserver2.key:/etc/nginx/certs/webserver.key:ro webserver
~$ docker compose restart proxy
~$ curl --cacert proxy-ca.crt https://localhost:9443/
<html>
    <h1>Test webserver</h1>
</html>
~$ docker rm -f webserver2 && rm webserver2.*
~$ docker compose up -d && docker compose restart proxy
```

Перезапуск proxy необходим, поскольку nginx разрешает имя `webserver` в IP-адрес однократно при запуске.

### Сценарии отказа

Прямой запрос к webserver без клиентского сертификата:

```console
~$ curl --cacert webserver-ca.crt https://localhost:8443/
...
<head><title>400 No required SSL certificate was sent</title></head>
...
```

Клиентский сертификат, выпущенный недоверенным CA (аналогичный ответ возвращается для сертификата с истёкшим
сроком действия):

```console
~$ openssl req -x509 -newkey rsa:2048 -nodes -keyout fake.key -out fake.crt -days 1 -subj "/CN=fake"
~$ curl --cacert webserver-ca.crt --cert fake.crt --key fake.key https://localhost:8443/
...
<head><title>400 The SSL certificate error</title></head>
...
~$ rm fake.*
```

Подмена webserver: имя в сертификате совпадает, однако сертификат подписан не `webserver-ca`:

```console
~$ openssl req -x509 -newkey rsa:2048 -nodes -keyout evil.key -out evil.crt -days 1 -subj "/CN=evil" -addext "subjectAltName=DNS:webserver"
~$ docker compose stop webserver
~$ docker compose run -d --name fake_webserver --use-aliases -v ./evil.crt:/etc/nginx/certs/webserver.crt:ro -v ./evil.key:/etc/nginx/certs/webserver.key:ro webserver
~$ docker compose restart proxy
~$ curl --cacert proxy-ca.crt https://localhost:9443/
...
<head><title>502 Bad Gateway</title></head>
...
~$ docker compose logs proxy | grep upstream
... upstream SSL certificate verify error: (18:self-signed certificate) while SSL handshaking to upstream ...
~$ docker rm -f fake_webserver && rm evil.*
~$ docker compose up -d && docker compose restart proxy
```

Запрос к proxy без указания доверенного CA (или с указанием постороннего CA):

```console
~$ curl https://localhost:9443/
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

### Запрос клиентского сертификата

Сообщение CertificateRequest в рукопожатии содержит перечень CA, сертификаты которых webserver принимает от
клиентов:

```console
~$ openssl s_client -connect localhost:8443 -CAfile webserver-ca.crt </dev/null 2>/dev/null | grep -A1 "Acceptable client"
Acceptable client certificate CA names
CN=Demo mTLS CA
```
