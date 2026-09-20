# Showcase 07 — Gateway API + several Spring Boot microservices

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
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

Проверено: 2026-09-20.

## Схема

```text
Internet
   |
   v
Gateway
   |
   +--> HTTPRoute /orders    --> Service orders-api   --> Pods
   |
   +--> HTTPRoute /customers --> Service customer-api --> Pods
```

## Почему это отдельный стенд

Ingress обычно представляет routing в одном объекте. Gateway API разделяет ответственность:

```text
Platform team
 -> GatewayClass
 -> Gateway

Application teams
 -> HTTPRoute
 -> Service
 -> Deployment
```

Это особенно удобно, когда один общий edge обслуживает несколько команд/микросервисов.

## Prerequisite

Gateway API CRDs и совместимый controller должны быть установлены.

Проверьте:

```bash
kubectl get gatewayclass
kubectl api-resources | grep -E 'gateway|httproute'
```

В примере `gatewayClassName: replace-with-installed-class` намеренно является placeholder.

## Mapping 1 — Gateway listener -> HTTPRoute parentRefs

Gateway:

```yaml
metadata:
  name: public-api
```

HTTPRoute:

```yaml
parentRefs:
  - name: public-api
```

Это exact object reference.

## Mapping 2 — HTTPRoute -> Service

```yaml
backendRefs:
  - name: orders-api
    port: 8080
```

должно указывать на существующий Service.

## Mapping 3 — Service -> Pods

Дальше обычная цепочка:

```text
HTTPRoute
 -> Service metadata.name
 -> Service selector
 -> Pod labels
 -> EndpointSlice
 -> Ready Pods
```

## Host and path rules

Оба routes используют:

```text
api.example.kz
```

но разные prefixes:

```text
/orders
/customers
```

Так edge routing отделяется от internal Service discovery.

## Что здесь platform-specific

- GatewayClass/controller;
- external load balancer;
- TLS/certificate integration;
- WAF/rate limiting;
- implementation-specific policies.

## Failure simulations

1. Wrong GatewayClass -> Gateway не программируется.
2. HTTPRoute parentRef указывает не на тот Gateway.
3. backend Service missing.
4. Service selector mismatch -> route существует, endpoints нет.
5. Backend NotReady -> Gateway здоров, но usable backend capacity отсутствует.

## Проверка

```bash
kubectl get gatewayclass
kubectl get gateway
kubectl describe gateway public-api
kubectl get httproute
kubectl describe httproute orders-route
kubectl get svc
kubectl get endpointslice
```

## Sources

- https://kubernetes.io/docs/concepts/services-networking/gateway/
- Gateway API 1.5 released in 2026; implementation version/support must be checked separately.
