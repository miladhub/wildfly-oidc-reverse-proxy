Deploying a web app on Wildfly using OIDC behind a reverse proxy
===

The goal of this sample application is to show how to deploy a webapp on
WildFly, using Keycloak as the OpenID Connect (OIDC) SSO provider, having both
WildFly and Keycloak behind a TLS-enabled reverse proxy.

# Prerequisites

* JDK >= 17
* Docker
* openssl

# Installation

Create a certificate:

```shell
openssl genrsa -des3 -passout pass:x -out localhost.pass.key 2048
openssl rsa -passin pass:x -in localhost.pass.key -out localhost.key
openssl req -new -key localhost.key -out localhost.csr \
  -subj "/C=IT/ST=Italy/L=Bologna/O=FooBar inc/OU=Nerds/CN=foo.bar.baz"
openssl x509 -req -sha256 -days 365 -in localhost.csr -signkey localhost.key -out localhost.crt
rm localhost.pass.key localhost.csr
```

Alternatively, to avoid self-signed certificate warnings, install
[mkcert](https://github.com/FiloSottile/mkcert) and issue the following commands:
```shell
mkcert --install
mkcert localhost
openssl x509 -outform der -in localhost.pem -out localhost.crt
openssl pkey -in localhost-key.pem -out localhost.key
```

Install Apache HTTPD:

```shell
docker rm -f apache-https
docker run -dit --name apache-https -p 443:443 httpd:2.4.49
docker cp httpd.conf apache-https:/usr/local/apache2/conf
docker cp httpd-ssl.conf apache-https:/usr/local/apache2/conf/extra
docker cp localhost.pem apache-https:/usr/local/apache2/conf/server.crt
docker cp localhost.key apache-https:/usr/local/apache2/conf/server.key
docker restart apache-https
```

Download [WildFly](https://github.com/wildfly/wildfly/releases/download/28.0.1.Final/wildfly-28.0.1.Final.zip)
to `~/wildfly-28.0.1.Final`.

Create a TLS trust store to allow Elytron to connect to Keycloak over HTTPS,
starting from the certificate used to configure the Apache server - here we are
choosing demo password `changeit`:

```shell
keytool -importcert -alias keycloak -keystore /path/to/cacerts.p12 -file localhost.crt
```

Add this section to `~/wildfly-28.0.1.Final/standalone/configuration/standalone.xml`
as documented at <https://docs.wildfly.org/28/Admin_Guide.html#Elytron_OIDC_Client>:

```xml
<subsystem xmlns="urn:wildfly:elytron-oidc-client:1.0">
    <secure-deployment name="myapp.war" truststore="/path/to/cacerts.p12" truststore-password="changeit">
        <provider-url>https://localhost/auth/realms/my-app-realm</provider-url>
        <ssl-required>ALL</ssl-required>
        <client-id>my-app-oidc-client</client-id>
        <principal-attribute>preferred_username</principal-attribute>
        <credential name="secret" secret="0B45EHhYFP1pyTlbhYcUyNJdpopkXrrd"/>
    </secure-deployment>
</subsystem>
```

Start WildFly:

```shell
$ ~/wildfly-28.0.1.Final/bin/standalone.sh
```

Install [Keycloak](https://www.keycloak.org) via Docker:

```shell
$ docker run --name keycloak-rev-proxy -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -p 8180:8080 quay.io/keycloak/keycloak:21.1.1 start \
  --proxy edge --http-relative-path=/auth --hostname-strict=false --hostname-url=https://localhost/auth
```

Access Keycloak at <https://localhost/auth/admin/> with credentials `admin` / `admin`
and create a realm by importing [this JSON file](realm-export.json).

Create a user in the realm: 
* Username: `foo`
* Email: `foo@fake.com`
* Email verified `true`
* Credentials > Set password `foo`, Temporary: `false`

Build and deploy the WAR:

```shell
$ mvn clean install && cp target/myapp.war \
  ~/wildfly-28.0.1.Final/standalone/deployments/
```

You can now access the app at <https://localhost/myapp> and be authenticated
via OIDC with Keycloak.

These pieces of configuration were necessary to make it all work:

```shell
<Location /myapp>
  # ...
  RequestHeader set X-Forwarded-Proto "https"
</Location>
```

```xml
<http-listener name="default" ... proxy-address-forwarding="true" />
```

See <https://groups.google.com/g/wildfly/c/4P6MQXPNRlo>.
