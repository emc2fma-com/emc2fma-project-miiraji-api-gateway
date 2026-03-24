# EMC2FMA Project Mira-j`-i : API Gateway

A Spring WebFlux-based API Gateway providing JWT authentication, dynamic route proxying, CORS enforcement, rate limiting, circuit breaking, and pluggable user management — configurable entirely through environment variables or `application.yml`.
The environment variables are available in env-file, [.env](https://github.com/emc2fma-com/emc2fma-project-miiraji-api-gateway/blob/main/.env-file) or [application.properties](https://github.com/emc2fma-com/emc2fma-project-miiraji-api-gateway/blob/main/application.properties)

---

## Table of Contents

1. [Features](#features)
2. [User Repository — Internal vs External](#1-user-repository--internal-vs-external)
3. [Service Discovery & Route Configuration](#2-service-discovery--route-configuration)
4. [JWT Authentication](#3-jwt-authentication)
5. [Rate Limiter](#4-rate-limiter)
6. [Circuit Breaker](#5-circuit-breaker)
7. [Logging](#6-logging)
8. [Actuator / Management](#7-actuator--management)
9. [Quick-Start Configuration](#quick-start-configuration)

---

## Features

| # | Feature | Status |
|---|---------|--------|
| 1 | Internal user repo (MySQL + Hibernate) or External User API | Configurable |
| 2 | Dynamic route forwarding / proxying | Configurable |
| 3 | JWT Auth — STRING / filesystem / AWS S3 / Azure Blob keys | Configurable |
| 4 | Redis-based sliding-window rate limiter | Optional |
| 5 | Resilience4j circuit breaker | Optional |
| 6 | Structured logging with MDC (userId, correlationId) | Always on |
| 7 | Spring Boot Actuator health & management endpoints | Always on |

---

## 1. User Repository — Internal vs External

The gateway can manage users from its own MySQL database (via Hibernate/JPA) **or** delegate all user lookups to an external User microservice.

### Internal repo (MySQL + Hibernate)

Enable Hibernate and configure the datasource:

```yaml
spring:
  datasource:
    url: jdbc:mysql://<db-host>:3306/<mydb>
    username: root
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: validate        # use 'validate' or 'none' in production
    show-sql: false
```

Pass the JVM flag at startup:

```bash
java -Dhibernate.enabled=true -jar api-gateway.jar
```

```bash
docker run  -p 8080:8080 --env-file .env -e JDK_JAVA_OPTIONS="-Dhibernate.enabled=true" --name emc2fma/emc2fma.com-api-gateway
```

A default admin account is seeded on first startup:

| Property | Default | Notes |
|----------|---------|-------|
| `user.default.userName` | `admin` | Change immediately |
| `user.default.pwd` | `admin123` | Change immediately |
| `user.default.roleName` | `admin` | |
| `user.default.startDate` | `2026-01-19` | `yyyy-MM-dd` |
| `user.default.endDate` | `2028-01-19` | `yyyy-MM-dd` |

> **Production:** change `user.default.pwd` before go-live and set `ddl-auto: validate`.

### External User API

Disable Hibernate and point the gateway at your User microservice:

```bash
java -Dhibernate.enabled=false -jar output/api-gateway-0.0.1-spring-boot.jar
```
```bash
docker run  -p 8080:8080 --env-file .env -e JDK_JAVA_OPTIONS="-Dhibernate.enabled=false" --name emc2fma/emc2fma.com-api-gateway
```

```yaml
user:
  service:
    api:
      url: <http://user-service:4001/user>
      path:
        name: </by-name>        # optional — path segment for name lookups
        id:   </by-id >           # optional — path segment for id lookups
      query:   ""               # optional default query string
      headers: ""               # optional forwarded headers (key=value)
```

---

## 2. Service Discovery & Route Configuration

The gateway acts as a reverse proxy. Routes are declared as a map where **each key is an Ant-style path pattern** and the value is the **target upstream URL**.

```yaml
routes:
  apis:
    /order/**:   http://order-service:8081
    /inventory/**: http://inventory-service:8082
    /payment/**:  http://payment-service:8083
```

All requests matching `/order/**` are forwarded to `http://order-service:8081`, preserving the path suffix.

### JWT whitelist (unauthenticated paths)

Paths that should bypass JWT verification (e.g. docs, public health):

```yaml
security:
  white:
    paths: /sample/v1/docs/**, /actuator/health, /public/**
```

---

## 3. JWT Authentication

The gateway validates JWTs on every non-whitelisted request. Signing keys can be sourced , by setting env variables, `security.jwt.public.type` and `security.jwt.private.type` (must be the same value).

| Type | Description |
|------|-------------|
| `STRING` | Inline HMAC secret in config |
| `X509_JKS_FILE` | RSA key pair from local filesystem (JKS / CER) |

For `X509_JKS_FILE` the storage types is selected by env variables, `security.jwt.public.store` and `security.jwt.private.store`:

| `FILE` | RSA keys fetched from local file disk |
| `AWS` | RSA keys fetched from Amazon S3 |
| `AZURE` | RSA keys fetched from Azure Blob Storage |

Token expiry is controlled by `security.jwt.token.expiration` (default: **10 minutes**).

### STRING (HMAC)

```yaml
security:
  jwt:
    public:  { type: STRING }
    private: { type: STRING }
    string:
      public.Key:  your-public-hmac-secret
      private.Key: your-private-hmac-secret
```

### FS (filesystem RSA)

```yaml
security:
  jwt:
    public:  { type: FS }
    private: { type: FS }
    fs:
      public.path:  /keys/public.cer
      private.path: /keys/private.jks
    rsa:
      fs:
        storepass: changeit
        keypass:   changeit
        alias:     my-key-alias
```

### AWS S3

```yaml
security:
  jwt:
    public:  { type: AWS }
    private: { type: AWS }
    aws:
      public:
        region: <aws-region>
        access: <aws-access>
        secret: <aws-secret>
        bucket: <bucket>
        path:   <path-to-public-cer>
      private:
        region: <aws-region>
        access: <aws-access>
        secret: <aws-secret>
        bucket: <bucket>
        path:   <path-to-private-jks>
```

### Azure Blob Storage

```yaml
security:
  jwt:
    public:  { type: AZURE }
    private: { type: AZURE }
    azure:
      public:
        connectionString: <"DefaultEndpointsProtocol=https;AccountName=...">
        containerName: <container>
        blobName: <certificate-file>
      private:
        connectionString: <"DefaultEndpointsProtocol=https;AccountName=...">
        containerName: <container>
        blobName: <jks-file>
```

---

## 4. Rate Limiter

Sliding-window rate limiting backed by **Redis**.

```yaml
security:
  rl:
    enabled:       true
    burstRate:     100          # max requests per window
    windowSeconds: 1            # window size in seconds
    header.key:    X-Client-Id  # header used to identify the caller
    redis:
      host:    <redis-host>
      port:    <redis-port - default 6379>
      dbindex: 1
      uname:   <redisuser>
      pwd:     <redispassword>
```

> Set `enabled: false` to remove the Redis startup dependency entirely.

---

## 5. Circuit Breaker

Resilience4j circuit breaker wraps all upstream proxy calls. Disabled by default.

```yaml
spring:
  cloud:
    circuitbreaker:
      resilience4j:
        enabled: true

resilience4j:
  circuitbreaker:
    instances:
      default:
        minimumNumberOfCalls:                        5
        failureRateThreshold:                        50      # %
        slidingWindowType:                           COUNT_BASED
        slidingWindowSize:                           10
        waitDurationInOpenState:                     10000   # ms
        permittedNumberOfCallsInHalfOpenState:       3
        automaticTransitionFromOpenToHalfOpenEnabled: true
```

States: **CLOSED** (normal) → **OPEN** (failing, blocks calls) → **HALF_OPEN** (probe) → back to CLOSED or OPEN.

---

## 6. Logging

Log levels follow Logback conventions. Every request is decorated with **MDC context** (`userId`, `correlationId`) automatically.

```yaml
logging:
  level:
    root:                  INFO
    org.springframework:   INFO
    org.springdoc:         INFO
  pattern:
    console: "%d{HH:mm:ss} [%X{correlationId}] [%X{userId}] %-5level %logger{36} - %msg%n"
  file:
    path: </var/log/api-gateway>   # omit to disable file logging
```

Available levels: `TRACE` / `DEBUG` / `INFO` / `WARN` / `ERROR`

---

## 7. Actuator / Management

Spring Boot Actuator is enabled by default. Sensitive endpoints (`env`, `beans`) are excluded.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
        exclude: env, beans
  endpoint:
    health:
      show-details: always
```

Key endpoints:

| Endpoint | Purpose |
|----------|---------|
| `GET /actuator/health` | Liveness / readiness probe |
| `GET /actuator/info` | App metadata |
| `GET /actuator/metrics` | Micrometer metrics |
| `GET /actuator/loggers` | Runtime log-level inspection |

---

## Quick-Start Configuration

Minimal `application.yml` for local development:

```yaml
spring:
  application:
    name: api-gateway
  main:
    web-application-type: reactive
  datasource:
    url: jdbc:mysql://mysql-db-server:3306/gatewaydb
    username: root
    password: secret
  jpa:
    hibernate:
      ddl-auto: update

server:
  port: 8080

security:
  white:
    paths: /actuator/health, /v3/api-docs/**, /swagger-ui/**
  cors:
    enable: true
    allowed:
      origins: "*"
      methods: "*"
      headers: "*"
    creds: false
    maxAge: 5
  jwt:
    token:
      expiration: 10
    public:
      type: STRING
    private:
      type: STRING
    string:
      public.Key:  dev-public-key
      private.Key: dev-private-key

routes:
  apis:
    </product/**>:  <http://product-service:4001/product>

springdoc:
  swagger-ui:
    path: /swagger-ui.html
    enabled: true
  api-docs:
    path: /v3/api-docs
    enabled: true
```

Start:

```bash
java -Dhibernate.enabled=true -jar api-gateway.jar
```
```bash
docker run  -p 8080:8080 --env-file .env -e JDK_JAVA_OPTIONS="-Dhibernate.enabled=true" --name emc2fma/emc2fma.com-api-gateway
```

---
