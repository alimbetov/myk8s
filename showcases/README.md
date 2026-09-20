# Kubernetes Architecture Showcases

Проверено: 2026-09-20.

`showcases/` — набор production-like ознакомительных стендов. Их задача — показать не отдельный Kubernetes object, а **как несколько manifests образуют работающую систему**.

## Чем showcase отличается от examples и labs

- `examples/` — маленькие reusable manifests.
- `docs/knowledge/manifests/` — field-by-field technical reference.
- `showcases/` — законченные архитектурные композиции.
- `labs/` — практические exercises: сломать → диагностировать → исправить.

## Как читать стенды

Для каждого scenario:

```text
README.md
  ↓
architecture + mappings + failure scenarios

all.yaml
  ↓
совместно работающие Kubernetes resources

application.yaml
  ↓
Spring Boot side of the contract, если требуется
```

## Серия A — application fundamentals

| # | Стенд | Что изучаем |
|---|---|---|
| 01 | [Internal REST service](01-internal-rest-service/README.md) | Deployment + Service + ConfigMap + Secret + probes |
| 02 | [Public API / Ingress / TLS](02-public-api-ingress-tls/README.md) | Ingress → Service → Ready Pods |
| 03 | [REST + PostgreSQL + HPA + NetworkPolicy](03-rest-postgres-hpa-networkpolicy/README.md) | resources ↔ HPA, Hikari ↔ replicas, DB egress |
| 04 | [Spring Batch / CronJob](04-batch-cronjob/README.md) | CronJob → Job → Pod, retry/overlap/timezone |
| 05 | [File worker + PVC](05-file-worker-pvc/README.md) | PVC → volume → volumeMount |
| 06 | [Service-to-service security](06-service-to-service-security/README.md) | DNS + Service + NetworkPolicy + OAuth2 layers |

## Серия B — delivery and edge routing

| # | Стенд | Что изучаем |
|---|---|---|
| 07 | [Gateway API + microservices](07-gateway-microservices/README.md) | GatewayClass/Gateway/HTTPRoute/Service responsibility |
| 08 | [Blue/Green](08-blue-green/README.md) | two Deployments + Service selector cutover |
| 09 | [Canary](09-canary/README.md) | weighted HTTPRoute backends, v1/v2 coexistence |

## Серия C — stateful products through operators

| # | Стенд | Что изучаем |
|---|---|---|
| 10 | [Kafka producer/consumer + Strimzi](10-kafka-producer-consumer/README.md) | Kafka/KafkaNodePool, KRaft, bootstrap Service, partitions |
| 11 | [RabbitMQ worker + Operator](11-rabbitmq-worker/README.md) | RabbitmqCluster, client Service, persistence, workers |
| 12 | [PostgreSQL + CloudNativePG](12-cloudnativepg-primary-replicas/README.md) | primary/replicas, -rw/-ro Services, failover identity |

## Серия D — configuration and Pod composition

| # | Стенд | Что изучаем |
|---|---|---|
| 13 | [S3-compatible object storage](13-object-storage-s3/README.md) | endpoint/config/credentials/object metadata consistency |
| 14 | [Mounted application.yaml](14-configmap-mounted-application-yaml/README.md) | ConfigMap file → volume → Spring config location |
| 15 | [Secret configtree](15-secret-configtree/README.md) | Secret files → configtree → Spring Environment |
| 16 | [initContainer + main](16-initcontainer-main/README.md) | sequential initialization + shared emptyDir |
| 17 | [Native sidecar](17-native-sidecar/README.md) | sidecar lifecycle + localhost + resource coupling |
| 18 | [StatefulSet + headless Service](18-statefulset-headless-service/README.md) | stable ordinal DNS + per-Pod PVC identity |

## Mapping cheat sheet

Перед чтением сложных стендов полезно открыть:

- [Manifest Mapping Cheat Sheet](MAPPING-CHEATSHEET.md)

Он разделяет связи на четыре класса:

```text
A. exact object reference
B. label selector
C. named port mapping
D. runtime/application mapping
```

## Основные цепочки, которые повторяются

### Workload routing

```text
Deployment.template.labels
      ↓
Service.selector
      ↓
EndpointSlice
      ↓
Ready Pods
```

### Ports

```text
Ingress/Gateway backend Service port
      ↓
Service.port
      ↓
Service.targetPort
      ↓
containerPort.name / number
```

### Configuration

```text
ConfigMap / Secret
      ↓
env / volume / configtree
      ↓
Spring Environment
      ↓
@ConfigurationProperties
```

### Scaling

```text
resources.requests.cpu
      ↓
HPA utilization

HPA.maxReplicas
      ↓
number of JVMs
      ↓
connection pools / consumers
      ↓
downstream capacity
```

### Stateful products

```text
Application
      ↓
operator-managed logical Service
      ↓
current product role topology
      ↓
stateful Pods + persistent volumes
```

## Важное ограничение

Showcases — **reference stands**, а не universal manifests для прямого production copy/paste.

Platform-specific области всегда проверяются отдельно:

- GatewayClass / IngressClass;
- CNI и NetworkPolicy;
- StorageClass/CSI;
- secret manager;
- metrics provider;
- identity provider;
- Strimzi version/CRDs;
- RabbitMQ Operator version;
- CloudNativePG version;
- object storage implementation.

Главная цель стендов — научиться видеть relationships и runtime behavior до того, как YAML будет адаптирован под конкретную платформу.
