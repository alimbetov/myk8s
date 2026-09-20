# Showcase 06 — Two Spring Boot services with service-to-service security

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Spring Boot configuration](application.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
orders-api
   |
   | DNS: http://payment-api:8080
   | NetworkPolicy permits TCP
   | OAuth2 bearer token authenticates caller
   v
payment-api
```

## Здесь намеренно три разных security layer

### 1. NetworkPolicy

Отвечает:

```text
может ли packet от orders-api попасть в payment-api:8080?
```

### 2. TLS / mTLS

Не реализован в portable core example, потому что конкретная реализация зависит от:
- service mesh;
- gateway;
- certificate automation;
- Spring TLS setup.

### 3. OAuth2/JWT

Payment API проверяет bearer token.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OIDC_ISSUER_URI}
```

## Mapping NetworkPolicy

Caller Pod:

```yaml
labels:
  app: orders-api
```

Target Pod:

```yaml
labels:
  app: payment-api
```

Policy payment ingress:

```yaml
podSelector:
  matchLabels:
    app: payment-api
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: orders-api
```

Так mapping identity на network layer строится через labels.

Это **не** cryptographic identity: Pod с тем же label и permission создать workload потенциально попадёт под selector. Поэтому JWT/mTLS остаются отдельным security layer.

## Mapping Service

```text
orders application.yaml
PAYMENT_API_URL=http://payment-api:8080
                   |
                   v
Service payment-api
                   |
                   v
payment-api Ready Pods
```

## Failure simulations

1. NetworkPolicy deny -> connect timeout.
2. Wrong Service name -> DNS error.
3. No bearer token -> HTTP 401.
4. Valid token without authority -> HTTP 403.
5. Payment Pod NotReady -> Service loses backend, но orders liveness не должна падать из-за этого.
