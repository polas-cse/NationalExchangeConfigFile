# NationalExchangeConfigFile

Centralised configuration for the **National Exchange** microservice system.

These files are not read by hand. They are served at runtime by
**`NationalExchangeConfigService`** (Spring Cloud Config Server, port `7071`),
which clones this repository and hands each service the properties meant for it.

> ### This repository is public — never commit a plaintext secret
> Database passwords, JWT signing keys, broker credentials and API tokens do
> **not** belong here as plain text. Keep them in each service's local
> `application.properties`, in environment variables, or store them here
> encrypted as `{cipher}` values. See [Encrypting a secret](#encrypting-a-secret).

---

## Which file is used by which service

The config server matches a file to a service by its **`spring.application.name`**.
A service called `user-service` is served `user-service.properties`; nothing else
is sent to it.

| File | Served to | `spring.application.name` | Port | Keys |
| --- | --- | --- | --- | --- |
| `application.properties` | **every service** | — (shared by all) | — | 26 |
| `discovery-server.properties` | Discovery Server (Eureka) | `discovery-server` | 8761 | 6 |
| `api-gateway.properties` | API Gateway | `api-gateway` | 8080 | 12 |
| `user-service.properties` | User Service | `user-service` | 9090 | 23 |
| `bank-service.properties` | Bank Service | `bank-service` | 9091 / 9191 gRPC | 20 |

`NationalExchangeConfigService` itself has **no file here** — it is the server
that serves this repository, so it reads its own local configuration instead.

### What each file holds

**`application.properties`** — shared defaults every service inherits:
Eureka client registration, actuator exposure (including `busrefresh`),
RabbitMQ host/port/vhost for the config bus, config-client retry and fail-fast
behaviour, Jackson and error-response defaults, root logging levels and the
trace-id log pattern, tracing sample rate.

**`discovery-server.properties`** — Eureka server settings: port 8761,
`register-with-eureka=false` and `fetch-registry=false` (the registry must not
register with itself), self-preservation and eviction interval.

**`api-gateway.properties`** — port 8080, static-resource handling disabled so
it stays a pure routing layer, gateway discovery locator, HTTP client connect
and response timeouts, CORS allow-lists, gateway log level.

**`user-service.properties`** — port 9090, R2DBC and JDBC URLs for the
`user_service` schema, Liquibase schemas, RabbitMQ listener and publisher
tuning on the `national-exchange` vhost, the gRPC client pointing at
bank-service, and JWT token lifetimes (**the signing key is not here**).

**`bank-service.properties`** — port 9091 plus gRPC server port 9191, R2DBC and
JDBC URLs for the `banking_service` schema, Liquibase schemas, and the matching
RabbitMQ listener and publisher tuning.

---

## How precedence works

A service receives the shared file **and** its own file. Its own file wins:

```
application.properties          lowest  — shared defaults
<service>.properties                    — overrides the shared defaults
<service>-<profile>.properties  highest — overrides both
```

A profile-specific file is optional. To give user-service different settings in
production, add `user-service-prod.properties` and start it with
`spring.profiles.active=prod`.

Local properties inside the service still override everything served here,
unless the service sets `spring.cloud.config.override-none=false`.

Branch served by default: **`main`**
(`spring.cloud.config.server.git.default-label`).

---

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
spring.application.name=user-service
spring.config.import=optional:configserver:http://localhost:7071
```

The name must match the file name here, or the service silently gets only
`application.properties`.

---

## Checking what a service will receive

With the config server running on 7071:

```bash
curl http://localhost:7071/user-service/default
```

Merged, flattened view — exactly the values the service ends up with:

```bash
curl http://localhost:7071/user-service-default.properties
```

Swap the name for `discovery-server`, `api-gateway` or `bank-service`.
Full endpoint reference lives in the config server's own README.

---

## Refreshing without a restart

Commit and push a change here, then broadcast it over Spring Cloud Bus
(RabbitMQ). The `Content-Type` header is required — without it the endpoint
returns `415 Unsupported Media Type`:

```bash
curl -X POST -H "Content-Type: application/json" http://localhost:7071/actuator/busrefresh
```

Every connected service re-reads this repository. Beans that should pick up new
values need `@RefreshScope`.

---

## Encrypting a secret

With the config server running:

```bash
curl -X POST -H "Content-Type: text/plain" --data-raw 'my-real-password' http://localhost:7071/encrypt
```

Paste the result into a property file here with the `{cipher}` prefix and **no
quotes**:

```properties
spring.r2dbc.password={cipher}a3b6c11105a0032f9b54250573845b54ef92c62a18f69b2...
```

The config server decrypts it before serving, so the client receives the
plaintext and needs no key of its own.

The ciphertext is different every time you encrypt the same input — that is the
random salt, not a bug. Verify a value by decrypting it rather than by
comparing strings:

```bash
curl -X POST -H "Content-Type: text/plain" --data-raw '<cipher-text>' http://localhost:7071/decrypt
```

---

## Adding a new service

1. Set `spring.application.name=<name>` in the service.
2. Create `<name>.properties` in this repository.
3. Put only values that differ from `application.properties` in it.
4. Commit, push, then `busrefresh`.
5. Confirm with `curl http://localhost:7071/<name>/default`.
