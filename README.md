# NationalExchangeConfigFile

Centralised configuration repository for the **National Exchange** microservice system.
It is read at runtime by `NationalExchangeConfigService` (Spring Cloud Config Server,
port `7071`), which serves these properties to every other service.

> **This repository is public. Never commit a plaintext secret here.**
> Database passwords, JWT signing keys, mail credentials and API tokens stay in each
> service's local `application.properties`, in environment variables, or are stored here
> only as `{cipher}` values encrypted with the config server's `encrypt.key`.

## Layout

| File | Served to |
| --- | --- |
| `application.properties` | every service (shared defaults, lowest precedence) |
| `discovery-server.properties` | `discovery-server` |
| `api-gateway.properties` | `api-gateway` |
| `user-service.properties` | `user-service` |
| `bank-service.properties` | `bank-service` |

A service's own file wins over `application.properties`. A profile-specific file
(`user-service-dev.properties`) wins over both.

Branch served by default: `main` (`spring.cloud.config.server.git.default-label`).

## How a service consumes this

Add to the service `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

Then in the service's `application.properties`:

```properties
spring.config.import=optional:configserver:http://localhost:7071
```

The service name must match the file name here, so `spring.application.name=user-service`
is served `user-service.properties`.

## Verifying

```
curl http://localhost:7071/user-service/default
curl http://localhost:7071/api-gateway/dev
```

## Refreshing without a restart

Commit and push a change, then broadcast it over Spring Cloud Bus (RabbitMQ):

```
curl -X POST http://localhost:7071/actuator/busrefresh
```

Every connected service re-reads this repository. Beans that should pick up new values
need `@RefreshScope`.

## Encrypting a secret

With the config server running:

```
curl -X POST --data-raw 'mysecret' http://localhost:7071/encrypt
```

Paste the result into a property file as `some.key={cipher}AQB1c...`.
