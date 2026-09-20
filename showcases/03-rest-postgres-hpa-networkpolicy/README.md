# Showcase 03 — REST API + PostgreSQL + HPA + NetworkPolicy

## Как изучать этот стенд

```text
1. Сначала посмотрите архитектурную схему ниже
2. Откройте annotated.yaml и пройдите manifest сверху вниз
3. Сопоставьте связи в WALKTHROUGH.md
4. После понимания используйте чистый all.yaml
5. Затем выполните failure simulations
```

> **Учебный принцип:** сначала понять роль объекта в общей системе, затем его поля, затем runtime behavior. Не начинайте с копирования YAML.

## Место этого стенда в общей системе

```text
request
   |
   v
Service
   |
   v
Deployment
   |
   +--> HPA  ← управляет количеством Pods
   |
   +--> NetworkPolicy ← ограничивает network flows
   |
   +--> ConfigMap/Secret
   |
   v
Spring Boot
   |
   +--> HikariCP
   v
PostgreSQL
```

Здесь впервые видно, что Kubernetes-настройки **переплетаются**: HPA влияет на число JVM, а число JVM — на число DB connections.

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Учебный manifest с подробными комментариями](annotated.yaml)
- [Чистый apply-ready manifest](all.yaml)
- [Spring Boot configuration](application.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

## Схема

```text
Ingress / caller
      |
      v
Service orders-api
      |
      v
Deployment orders-api
      |
      +--> ConfigMap
      +--> Secret
      +--> HikariCP
      |       |
      |       v
      |    Service postgres-rw:5432
      |
      +--> HPA
      |
      +--> NetworkPolicy
```

## Главная идея

Это уже не просто Deployment+Service. Здесь видно, как настройки одного объекта влияют на другой:

```text
Deployment.resources.requests.cpu
        ↓
HPA averageUtilization denominator

Deployment replicas / HPA maxReplicas
        ↓
Hikari maximumPoolSize × replicas
        ↓
PostgreSQL connection budget

NetworkPolicy egress
        ↓
DNS + PostgreSQL должны быть разрешены

Secret DB_PASSWORD
        ↓
environment
        ↓
spring.datasource.password
```

## Mapping 1 — HPA ↔ CPU request

Deployment:

```yaml
requests:
  cpu: 500m
```

HPA:

```yaml
target:
  type: Utilization
  averageUtilization: 70
```

При usage 350m:

```text
350m / 500m ~= 70%
```

Если request изменить на 250m, те же 350m уже означают около 140%.

## Mapping 2 — replicas ↔ Hikari ↔ PostgreSQL

```yaml
SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE: "10"
```

HPA:

```yaml
maxReplicas: 20
```

Потенциальный верхний порядок:

```text
20 Pods × 10 connections = 200 connections
```

Это не означает, что все 200 всегда открыты, но именно такой budget нужно проверить против PostgreSQL.

## Mapping 3 — DB Service

Spring:

```text
jdbc:postgresql://postgres-rw:5432/orders
```

Здесь `postgres-rw` — logical Service endpoint. Application не знает IP primary database.

В этом стенде DB Service показан как architecture boundary; сам PostgreSQL operator/cluster будет отдельным stateful showcase.

## Mapping 4 — NetworkPolicy

После default-deny egress приложение обязано иметь allow на:
- DNS;
- PostgreSQL port 5432.

NetworkPolicy не заменяет DB credentials и TLS.

## Проверка

```bash
kubectl get deploy,svc,hpa
kubectl describe hpa orders-api
kubectl top pod
kubectl get networkpolicy
kubectl get endpointslice   -l kubernetes.io/service-name=orders-api
```

## Failure simulations

1. Удалить CPU request -> HPA CPU utilization становится проблемным.
2. Поставить Hikari=50 и maxReplicas=20 -> theoretical 1000 DB connections.
3. Удалить egress rule к PostgreSQL -> DNS может работать, TCP:5432 нет.
4. Поменять DB password -> Kubernetes healthy, Spring получает DB auth error.
5. PostgreSQL unavailable -> liveness не должна превращать DB outage в restart storm.
