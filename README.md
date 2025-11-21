# nginx-proxy-ssl-auth-example


Create webserver nginx cert and key

```console
openssl req -x509 -newkey 4096 -keyout nginx.webserver.key -out nginx.webserver.crt -days 365 -nodes -subj "/CN=webserver" 
```

Create proxy nginx cert and key

```console
openssl req -x509 -newkey 4096 -keyout nginx.proxy.key -out nginx.proxy.crt -days 365 -nodes -subj "/CN=proxy" 
```

Create Auth CA cert and key:

```console
openssl genrsa -out auth.ca.key 4096
openssl req -x509 -new -key auth.ca.key -days 36500 -out auth.ca.crt -subj "/CN=ca"
```

Create Auth Client cert and key:

```console
openssl genrsa -out auth.client.key 4096
openssl req -new -key auth.client.key -out auth.client.csr -subj "/CN=client"
openssl x509 -req -in auth.client.csr -CA auth.ca.crt -CAkey auth.ca.key -CAcreateserial -out auth.client.crt -days 365
openssl verify -CAfile auth.ca.crt auth.client.crt
```

## Test

Declined direct access to webserver:

```console
~$ curl --insecure --url https://localhost:8443/
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.29.3</center>
</body>
</html>
~$
```

Accepted access through authorised proxy:

```console
~$ curl --insecure --url https://localhost:9443/
<html>
    <h1>Test webserver</h1>
</html>

```