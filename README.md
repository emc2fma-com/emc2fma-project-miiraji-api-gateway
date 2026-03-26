# EMC2FMA Project Miiraji : API Gateway

A Spring WebFlux-based API Gateway providing JWT authentication, dynamic route proxying, CORS enforcement, rate limiting, circuit breaking, and pluggable user management — configurable entirely through environment variables or `application.yml`.

Environment variables are available in [.env](https://github.com/emc2fma-com/emc2fma-project-miiraji-api-gateway/blob/main/.env-file) or [application.properties](https://github.com/emc2fma-com/emc2fma-project-miiraji-api-gateway/blob/main/application.properties).

---

## Table of Contents

1. [Features](#features)
2. [User Repository — Internal vs External](#1-user-repository--internal-vs-external)
3. [Service Discovery & Route Configuration](#2-service-discovery--route-configuration)
4. [JWT Authentication](#3-jwt-authentication)
5. [Rate Limiter](#4-rate-limiter)
6. [Circuit Breaker](#5-circuit-breaker)
7. [CORS](#6-cors)
8. [Logging](#7-logging)
9. [Actuator / Management](#8-actuator--management)
10. [API Reference](#9-api-reference)
11. [Quick-Start Configuration](#quick-start-configuration)

---

## Features

| # | Feature | Status |
|---|---------|--------|
| 1 | Internal user repo (MySQL + Hibernate) or External User API | Configurable |
| 2 | Dynamic route forwarding / proxying | Configurable |
| 3 | JWT Auth — STRING / filesystem / AWS S3 / Azure Blob keys | Configurable |
| 4 | Redis-based sliding-window rate limiter | Optional |
| 5 | Resilience4j circuit breaker | Optional |
| 6 | CORS enforcement | Configurable |
| 7 | Structured logging with MDC (userId, correlationId) | Always on |
| 8 | Spring Boot Actuator health & management endpoints | Always on |

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
java -Dhibernate.enabled=true -jar api-gateway-0.0.1-spring-boot.jar
```

```bash
docker run -p 8080:8080 -v <local-folder>:/app/files --env-file .env \
  -e JDK_JAVA_OPTIONS="-Dhibernate.enabled=true" \
  emc2fma/emc2fma.com-api-gateway:<version>
```

A default admin account is seeded on first startup:

| Property | Default | Notes |
|----------|---------|-------|
| `user.default.userName` | `admin` | Change immediately |
| `user.default.pwd` | `${USER_DEFAULT_PWD:admin123}` | **Override via env var before go-live** |
| `user.default.roleName` | `admin` | |
| `user.default.startDate` | `2026-01-19` | `yyyy-MM-dd` |
| `user.default.endDate` | `2028-01-19` | `yyyy-MM-dd` |

> **Production:** set the `USER_DEFAULT_PWD` environment variable before go-live and set `ddl-auto: validate`.

### External User API

Disable Hibernate and point the gateway at your User microservice:

```bash
java -Dhibernate.enabled=false -jar output/api-gateway-0.0.1-spring-boot.jar
```

```bash
docker run -p 8080:8080 -v <local-folder>:/app/files --env-file .env \
  -e JDK_JAVA_OPTIONS="-Dhibernate.enabled=false" \
  emc2fma/emc2fma.com-api-gateway:<version>
```

```yaml
user:
  service:
    api:
      url: <http://user-service:4001/user>      # required
      path:
        name: </by-name>        # optional — path segment for name lookups
        id:   </by-id>          # optional — path segment for id lookups
      query:   ""               # optional default query string
      headers: ""               # optional forwarded headers (key=value)
```

---

## 2. Service Discovery & Route Configuration

The gateway acts as a reverse proxy. Routes are declared as a map where **each key is an Ant-style path pattern** and the value is the **target upstream URL**.

```yaml
routes:
  apis:
    /order/**:      http://order-service:8081
    /inventory/**: http://inventory-service:8082
```

All requests matching `/order/**` are forwarded to `http://order-service:8081`, preserving the path suffix.

### JWT whitelist (unauthenticated paths)

Paths that bypass JWT verification (e.g. login, docs, public health):

```yaml
security:
  white:
    paths: /login, /actuator/health, /v3/api-docs/**, /swagger-ui/**
```

---

## 3. JWT Authentication

### Token issuance

Send credentials to the login endpoint — no JWT is required for this call:

```
POST /login
Content-Type: application/json

{
  "userName": "admin",
  "passWord": "admin123",
  "organizationId": "optional-org-id"
}
```

A successful response returns a signed JWT. Pass it on all subsequent protected requests:

```
Authorization: Bearer <token>
```

### Token validation

The gateway validates the JWT on every non-whitelisted request. Signing keys are sourced by setting `security.jwt.public.type` and `security.jwt.private.type` (must be the same value).

| Type | Description |
|------|-------------|
| `STRING` | Inline HMAC secret in config |
| `X509_JKS_FILE` | RSA key pair (JKS / CER) |

For `X509_JKS_FILE`, the storage location is selected by `security.jwt.public.store` and `security.jwt.private.store`:

| Store | Description |
|-------|-------------|
| `FILE` | RSA keys fetched from local file disk |
| `AWS` | RSA keys fetched from Amazon S3 |
| `AZURE` | RSA keys fetched from Azure Blob Storage |

Token expiry is controlled by `security.jwt.token.expiration` (default: **10 minutes**).

### STRING (HMAC)

```yaml
security:
  jwt:
    token:
      expiration: 10    # minutes
    public:
      type: STRING
    private:
      type: STRING
    string:
      public.Key:  your-public-hmac-secret
      private.Key: your-private-hmac-secret
```

### Filesystem RSA
#### ⚠️ "docker run -v" should point container app to local folder containing path to keys.
```yaml
security:
  jwt:
    token:
      expiration: 10    # minutes
    public:
      type: X509_JKS_FILE
      store: FILE
    private:
      type: X509_JKS_FILE
      store: FILE
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
    token:
      expiration: 10    # minutes
    public:
      type: X509_JKS_FILE
      store: AWS
    private:
      type: X509_JKS_FILE
      store: AWS
    aws:
      public:
        region: <aws-region>
        access: <aws-access-key>
        secret: <aws-secret-key>
        bucket: <bucket-name>
        path:   <path-to-public-cer>
      private:
        region: <aws-region>
        access: <aws-access-key>
        secret: <aws-secret-key>
        bucket: <bucket-name>
        path:   <path-to-private-jks>
```

### Azure Blob Storage

```yaml
security:
  jwt:
    token:
      expiration: 10    # minutes
    public:
      type: X509_JKS_FILE
      store: AZURE
    private:
      type: X509_JKS_FILE
      store: AZURE
    azure:
      public:
        connectionString: <DefaultEndpointsProtocol=https;AccountName=...>
        containerName: <container-name>
        blobName: <certificate-file>
      private:
        connectionString: <DefaultEndpointsProtocol=https;AccountName=...>
        containerName: <container-name>
        blobName: <jks-file>
```

---

## 4. Rate Limiter

Sliding-window rate limiting backed by **Redis**. When the limit is exceeded the gateway returns `429 Too Many Requests`. No `Retry-After` header is sent by default.

```yaml
security:
  rl:
    enabled:       true
    burstRate:     100          # max requests per window
    windowSeconds: 1            # window size in seconds
    header.key:    X-Client-Id  # header used to identify the caller
    redis:
      host:    <redis-host>
      port:    <redis-port>     # default: 6379
      dbindex: 1
      uname:   <redis-username>
      pwd:     <redis-password>
```

> Set `enabled: false` to remove the Redis startup dependency entirely.

---

## 5. Circuit Breaker

Resilience4j circuit breaker wraps all upstream proxy calls. Disabled by default.

The breaker **opens** when the failure rate exceeds `failureRateThreshold`% across the last `slidingWindowSize` calls (at least `minimumNumberOfCalls` must have been made). While open, all calls are blocked immediately. After `waitDurationInOpenState` ms it transitions to **HALF_OPEN**, allows `permittedNumberOfCallsInHalfOpenState` probe calls, then returns to **CLOSED** on success or back to **OPEN** on failure.

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
        minimumNumberOfCalls:                         5
        failureRateThreshold:                         50      # %
        slidingWindowType:                            COUNT_BASED
        slidingWindowSize:                            10
        waitDurationInOpenState:                      10000   # ms
        permittedNumberOfCallsInHalfOpenState:        3
        automaticTransitionFromOpenToHalfOpenEnabled: true
```

States: **CLOSED** (normal) → **OPEN** (failing, blocks calls) → **HALF_OPEN** (probe) → back to CLOSED or OPEN.

---

## 6. CORS

Cross-origin request handling is configurable per environment. For production, replace wildcards with explicit values.

```yaml
security:
  cors:
    enable: true
    allowed:
      origins: "https://app.example.com"   # use "*" for local dev only
      methods: "GET, POST, PUT, DELETE, OPTIONS"
      headers: "Authorization, Content-Type, X-Client-Id"
    creds:  false    # set true only if cookies/auth headers are needed cross-origin
    maxAge: 3600     # seconds to cache preflight response
```

> **Production:** never combine `origins: "*"` with `creds: true` — browsers will reject the combination.

---

## 7. Logging

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
    path: /var/log/api-gateway   # omit this key entirely to disable file logging
```

Available levels: `TRACE` / `DEBUG` / `INFO` / `WARN` / `ERROR`

> **Note:** The `{...}` tokens in the console pattern (e.g. `%d{HH:mm:ss}`, `%X{correlationId}`) are **Logback syntax**, not config placeholders — copy them verbatim.

## 7b. otel/otlp, prometheus and zipkin available for end-to-end tracing with metrics.

---

## 8. Actuator / Management

Spring Boot Actuator is enabled by default. `env` and `beans` are excluded because they expose sensitive environment values and the full bean graph.

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

The Swagger UI (`/swagger-ui/index.html`) and raw API docs (`/v3/api-docs`) are whitelisted by default and do not require a JWT.

---

## 9. API Reference

All endpoints except `POST /login` require a valid JWT:

```
Authorization: Bearer <token>
```

Several write endpoints also accept an optional `X-User-Id` header, recorded as `createdBy` / `updatedBy` in the audit fields.

### Response envelope

Every response — success or failure — uses the same wrapper:

```json
{
  "value": "<payload or null>",
  "error": {
    "message": "human-readable description",
    "code": 400
  }
}
```

`error` is `null` on success; `value` is `null` on error.

---

### Auth

#### `POST /login` — Authenticate and obtain a JWT

> No `Authorization` header required.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `userName` | string | yes | Account username |
| `passWord` | string | yes | Account password |
| `organizationId` | string | no | Optional tenant / org identifier |

```json
{
  "userName": "admin",
  "passWord": "admin123",
  "organizationId": "my-org"
}
```

**Response `200 OK`**

```json
{ "value": true, "error": null }
```

The signed JWT is returned in the `Authorization` response header. Use it as `Authorization: Bearer <token>` on all subsequent calls.

---

### Users

#### `POST /user` — Create a user

**Headers:** `X-User-Id` (optional, recorded as `createdBy`)

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `userName` | string | yes | Unique login name |
| `password` | string | yes | Plain-text password (hashed server-side) |
| `firstName` | string | no | Given name |
| `lastName` | string | no | Family name |
| `dob` | date-time | no | Date of birth — ISO 8601 e.g. `1990-06-15T00:00:00Z` |
| `startDate` | date-time | no | Account activation date |
| `expiredDate` | date-time | no | Account expiry date |
| `userRoles` | string[] | no | Role names to assign (roles must already exist) |

```json
{
  "userName": "alice",
  "password": "s3cr3t",
  "firstName": "Alice",
  "lastName": "Smith",
  "dob": "1990-06-15T00:00:00Z",
  "startDate": "2026-01-01T00:00:00Z",
  "expiredDate": "2028-01-01T00:00:00Z",
  "userRoles": ["admin", "viewer"]
}
```

**Response `200 OK`** — returns the new user's integer ID.

```json
{ "value": 42, "error": null }
```

---

#### `PUT /user/{id}` — Update a user

**Path params:** `id` (integer)

**Headers:** `X-User-Id` (optional, recorded as `updatedBy`)

**Request body** — same fields as `POST /user`. Include only the fields you want to change.

**Response `200 OK`**

```json
{ "value": true, "error": null }
```

---

#### `GET /user/id/{id}` — Get user by ID

**Path params:** `id` (integer)

**Response `200 OK`**

```json
{
  "value": {
    "id": 42,
    "userName": "alice",
    "firstName": "Alice",
    "lastName": "Smith",
    "password": "<hashed>",
    "dob": "1990-06-15T00:00:00Z",
    "startDate": "2026-01-01T00:00:00Z",
    "expiredDate": "2028-01-01T00:00:00Z",
    "userRoles": ["admin"],
    "createdAt": "2026-01-01T10:00:00Z",
    "createdBy": "admin",
    "updatedAt": "2026-03-01T09:00:00Z",
    "updatedBy": "admin"
  },
  "error": null
}
```

---

#### `GET /user/name/{uname}` — Get user by username

**Path params:** `uname` (string)

**Response `200 OK`** — same shape as `GET /user/id/{id}`.

---

#### `POST /user/all` — List users (paginated)

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `page` | integer | yes | Zero-based page number |
| `pageSize` | integer | yes | Results per page |

**Request body**

```json
{
  "sort": {
    "userName": "asc",
    "createdAt": "desc"
  }
}
```

**Response `200 OK`**

```json
{
  "value": [
    {
      "id": 42,
      "userName": "alice",
      "startDate": "2026-01-01T00:00:00Z",
      "expiredDate": "2028-01-01T00:00:00Z",
      "userRoles": ["admin"]
    }
  ],
  "error": null
}
```

---

### Roles

#### `POST /role` — Create a role

**Headers:** `X-User-Id` (optional, recorded as `createdBy`)

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `roleName` | string | yes | Unique role name |

```json
{ "roleName": "viewer" }
```

**Response `200 OK`** — returns the new role's integer ID.

```json
{ "value": 7, "error": null }
```

---

#### `PUT /role/{id}` — Update a role

**Path params:** `id` (integer)

**Headers:** `X-User-Id` (optional, recorded as `updatedBy`)

**Request body**

```json
{ "id": 7, "roleName": "editor" }
```

**Response `200 OK`**

```json
{ "value": true, "error": null }
```

---

#### `GET /role/id/{id}` — Get role by ID

**Path params:** `id` (integer)

**Response `200 OK`**

```json
{
  "value": {
    "id": 7,
    "roleName": "viewer",
    "createdAt": "2026-01-01T10:00:00Z",
    "createdBy": "admin",
    "updatedAt": "2026-01-01T10:00:00Z",
    "updatedBy": "admin"
  },
  "error": null
}
```

---

#### `GET /role/name/{uname}` — Get role by name

**Path params:** `uname` (string)

**Response `200 OK`** — same shape as `GET /role/id/{id}`.

---

#### `POST /role/all` — List roles (paginated)

**Query params:** `page` (integer, required), `pageSize` (integer, required)

**Request body**

```json
{
  "sort": { "roleName": "asc" }
}
```

**Response `200 OK`**

```json
{
  "value": [
    {
      "id": 7,
      "roleName": "viewer",
      "createdAt": "2026-01-01T10:00:00Z",
      "createdBy": "admin",
      "updatedAt": "2026-01-01T10:00:00Z",
      "updatedBy": "admin"
    }
  ],
  "error": null
}
```

---

### HTTP status codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `400` | Bad request — validation failed or missing required fields |
| `401` | Unauthorized — missing or invalid JWT |
| `403` | Forbidden — authenticated but insufficient role |
| `404` | Not found — user or role ID does not exist |
| `429` | Too many requests — rate limit exceeded |
| `500` | Internal server error — check actuator logs |

Error body:

```json
{
  "value": null,
  "error": { "message": "User not found", "code": 404 }
}
```

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
    paths: /login, /actuator/health, /v3/api-docs/**, /swagger-ui/**
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
      expiration: 10    # minutes
    public:
      type: STRING
    private:
      type: STRING
    string:
      public.Key:  dev-public-key
      private.Key: dev-private-key

routes:
  apis:
    /product/**: http://product-service:4001/product

springdoc:
  swagger-ui:
    path: /swagger-ui/index.html
    enabled: true
  api-docs:
    path: /v3/api-docs
    enabled: true
```

Start:

```bash
java -Dhibernate.enabled=true -jar api-gateway-0.0.1-spring-boot.jar
```

```bash
docker run -p 8080:8080 -v <local-folder>:/app/files --env-file .env \
  -e JDK_JAVA_OPTIONS="-Dhibernate.enabled=true" \
  emc2fma/emc2fma.com-api-gateway:<version>
```

---
