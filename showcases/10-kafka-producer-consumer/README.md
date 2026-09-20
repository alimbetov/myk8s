# Showcase 10 — Kafka producer/consumer with Strimzi

Проверено: 2026-09-20.

## Схема

```text
orders-api producer
      |
      | bootstrap: event-bus-kafka-bootstrap:9092
      v
Strimzi Kafka cluster (KRaft)
      |
      | topic orders.created
      v
notification-worker consumer
```

## Почему Operator

Kafka — distributed stateful product. Kubernetes primitives сами не реализуют:
- KRaft controller quorum;
- broker configuration;
- rolling broker operations;
- listener topology;
- topic/user lifecycle;
- partition replication.

В showcase используется Strimzi.

## Prerequisite

До применения CR должны быть установлены Strimzi CRDs и Cluster Operator.

Проверить:

```bash
kubectl api-resources | grep kafka.strimzi.io
kubectl get deployment -A | grep strimzi
```

## KafkaNodePool mapping

```yaml
metadata:
  labels:
    strimzi.io/cluster: event-bus
```

связывает `KafkaNodePool` с:

```yaml
kind: Kafka
metadata:
  name: event-bus
```

## KRaft roles

В ознакомительном стенде 3 nodes имеют обе роли:

```yaml
roles:
  - controller
  - broker
```

Для более серьёзной production topology controller и broker pools могут разделяться.

## Persistent storage

```yaml
storage:
  type: jbod
  volumes:
    - id: 0
      type: persistent-claim
      size: 20Gi
      deleteClaim: false
```

Kafka data lifecycle отделён от Pod lifecycle.

## Client Service

Strimzi создаёт internal bootstrap Service:

```text
event-bus-kafka-bootstrap
```

Spring:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
```

ConfigMap:

```yaml
KAFKA_BOOTSTRAP_SERVERS: event-bus-kafka-bootstrap:9092
```

Приложение не перечисляет broker Pod IP.

## Consumer scaling

Kafka consumer group parallelism ограничено partitions.

```text
topic partitions = 6

1 consumer  -> up to 6 partitions one process
3 consumers -> partitions distributed
6 consumers -> max useful parallel consumers
10 consumers -> ~4 consumers idle
```

HPA/KEDA не должны масштабировать consumers без учёта partition count.

## Delivery semantics

Consumer должен учитывать duplicate processing.

Типичный application design:
- idempotent handler;
- transaction/outbox where needed;
- explicit offset/ack semantics;
- DLQ/retry strategy.

## Failure simulations

1. Kafka bootstrap name wrong -> DNS failure.
2. Kafka unavailable -> producer/consumer retry behavior observable.
3. Consumer replicas > partitions -> idle consumers.
4. `deleteClaim: true` + destructive cluster deletion -> storage lifecycle changes dramatically.
5. replication/min ISR configuration inconsistent with broker count -> availability problems.

## Проверка

```bash
kubectl get kafka
kubectl get kafkanodepool
kubectl get pod
kubectl get svc | grep event-bus
kubectl get pvc
```

## Sources

- https://strimzi.io/documentation/
- Strimzi uses Kafka + KafkaNodePool resources for KRaft/node pools.
