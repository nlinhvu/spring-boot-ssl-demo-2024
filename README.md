## Enable HTTPS for Spring Boot App by Configuring SSL With Base64-encoded PEM Certificates

1. Create **Base64 PEM format** `the unencrypted private key (server.key)` and `the self-signed certificate (server.crt)` for configuring SSL by using **openssl**
```shell
openssl req -newkey rsa:2048 -x509 -sha256 -keyout server.key -out server.crt -days 365 -subj "/CN=self-signed-no-encrypt-key-cert" -nodes
```
* `-newkey rsa:2048`: create a new private key using the RSA algorithm, keysize=2048bits
* `-x509`: generate a self-signed X.509 certificate
* `-sha256`: hash function used to create the signature in the certificate
* `-keyout server.key`: the output private key file
* `-out server.crt`: the output certificate file
* `-days 365`: the validity period of the certificate: 1 year
* `-subj "/CN=self-signed-no-encrypt-key-cert"`:  "Common Name" field in Subject of the certificate
* `-nodes`: don't encrypt the private key with passphrase

The `the unencrypted private key (server.key)` will look like this
```text
-----BEGIN PRIVATE KEY-----
...
meTWwXUH7BGfDHfNTdEAodKAtJ8nFviFdhS3hn5DYMdom7skdW6KNgr37hWdI5WU
hagbxaoBfteksZuXkurMGeCXRA==
...
-----END PRIVATE KEY-----
```
2. Move `the unencrypted private key (server.key)` and `the certificate (server.crt)` to **./server/certs** folder
3. Configure SSL using PEM-encoded files using [old mechanism server.ssl.*](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto.webserver.configure-ssl.pem-files)
```yaml
server:
  ssl:
    certificate: server/certs/server.crt
    certificate-private-key: server/certs/server.key
  port: 8443
```
4. Run the application
```
Connector [https-jsse-nio-8443], TLS virtual host [_default_],...
Tomcat started on port 8443 (https)...
```
5. Send a sample request using curl, pass `--insecure` since it's a self-signed certificate and not trusted by curl
```shell
curl --insecure -v https://localhost:8443/hello
```
we will see `-subj "/CN=self-signed-no-encrypt-key-cert"` that we passed into openssl command
```
* Server certificate:
*  subject: CN=self-signed-no-encrypt-key-cert
```
---
#### What if we create `the encrypted private key` with `a passphrase (123456)` like
```shell
openssl req -newkey rsa:2048 -x509 -sha256 -keyout server.key -out server.crt -days 365 -subj "/CN=self-signed-cert-1" -passout pass:123456
```
#### The `the encrypted private key (server.key)` will look like this
```text
-----BEGIN ENCRYPTED PRIVATE KEY-----
...
meTWwXUH7BGfDHfNTdEAodKAtJ8nFviFdhS3hn5DYMdom7skdW6KNgr37hWdI5WU
hagbxaoBfteksZuXkurMGeCXRA==
...
-----END ENCRYPTED PRIVATE KEY-----
```
#### Override `the encrypted private key (server.key)` and `the certificate (server.crt)` into **./server/certs** folder
#### Run the application
```text
org.springframework.context.ApplicationContextException: Unable to start web server
Caused by: java.lang.IllegalArgumentException: Password is required for an encrypted private key
```

## [New SSL Bundles from Spring Boot 3.1](https://spring.io/blog/2023/06/07/securing-spring-boot-applications-with-ssl)

1. Apply new configuration properties of SSL bundles (`spring.ssl.bundle.pem`)
```yaml
spring:
  ssl:
    bundle:
      pem:
        server:
          keystore:
            certificate: server/certs/server.crt
            private-key: server/certs/server.key
            private-key-password: 123456

server:
  ssl:
    bundle: server
  port: 8443
```
Here, we configured a **bundle** named `server` under Base64-encoded PEM files (`spring.ssl.bundle.pem`)

And we used this `server` **bundle** for configuring our `server.ssl` 
2. Run the application
```
Connector [https-jsse-nio-8443], TLS virtual host [_default_],...
Tomcat started on port 8443 (https)...
```
3. Send a request again
```shell
curl --insecure -v https://localhost:8443/hello
```
we will see `-subj "/CN=self-signed-cert-1"` that we passed into openssl command
```
* Server certificate:
*  subject: CN=self-signed-cert-1
```

## [SSL hot reload with SSL Bundles from Spring Boot 3.2](https://spring.io/blog/2023/11/07/ssl-hot-reload-in-spring-boot-3-2-0)
1. Let's add `reload-on-update=true` for our current `server` bundle
```yaml
spring:
  ssl:
    bundle:
      pem:
        server:
          reload-on-update: true
```
2. Restart the application to apply the configuration property above
3. Send a request again
```shell
curl --insecure -v https://localhost:8443/hello
```
It will return the same as previous
```
* Server certificate:
*  subject: CN=self-signed-cert-1
```
4. Generate a new pair of `the encrypted private key` and `the certificate` 
with the same names are `server.key`, `server.crt` respectively and the same passphrase `123456` 
but this time, the common name is `self-signed-cert-2`:
```shell
openssl req -newkey rsa:2048 -x509 -sha256 -keyout server.key -out server.crt -days 365 -subj "/CN=self-signed-cert-2" -passout pass:123456
```
5. Override these 2 files into **./server/certs** folder
6. Take a look at the console log, the certificate will be rotated and logged out to our console
```text
Connector [https-jsse-nio-8443], TLS virtual host [_default_],...
```
6. Send a request again
```shell
curl --insecure -v https://localhost:8443/hello
```
We should see the subject is now changed from `self-signed-cert-1` to `self-signed-cert-2`
```
* Server certificate:
*  subject: CN=self-signed-cert-2
```

---
### References:
[Old SSL Mechanism](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto.webserver.configure-ssl)

[Spring Official Documentation - SSL](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.ssl)

[Securing Spring Boot Applications With SSL](https://spring.io/blog/2023/06/07/securing-spring-boot-applications-with-ssl)

[SSL hot reload in Spring Boot 3.2.0](https://spring.io/blog/2023/11/07/ssl-hot-reload-in-spring-boot-3-2-0)
