# Walkthrough — Kafka producer/consumer + Strimzi

## Архитектура

```text
orders-api producer
      |
      | event-bus-kafka-bootstrap:9092
      v
 Kafka cluster
      |
      | orders.created
      v
notification-worker consumer group
```

## Файлы

- [README](README.md)
- [Strimzi/Kubernetes manifests](all.yaml)
- [Spring Boot Kafka config](application.yaml)

## Operator boundary

Application не должна знать broker Pod names.

Strimzi управляет Kafka topology и Kubernetes resources.

## KafkaNodePool mapping

```yaml
labels:
  strimzi.io/cluster: event-bus
```

Operator связывает node pool с Kafka CR `event-bus`.

## Roles

Учебный пример:

```yaml
roles:
  - controller
  - broker
```

Три nodes совмещают KRaft controller и broker role.

Production может разделять pools.

## Storage

```yaml
type: persistent-claim
size: 20Gi
deleteClaim: false
```

### Осторожно

Storage lifecycle должен быть частью DR policy.

Нельзя менять `deleteClaim` как косметическую настройку.

## Bootstrap Service

```yaml
KAFKA_BOOTSTRAP_SERVERS: event-bus-kafka-bootstrap:9092
```

Spring:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
```

Client использует bootstrap endpoint, затем Kafka metadata сообщает broker topology.

## Topic

```yaml
partitions: 6
replicas: 3
```

Не путать:

```text
partitions -> parallelism/order units
replicas   -> copies for durability/availability
```

## Consumer group scaling

6 partitions:

```text
1 consumer  -> 6 partitions
3 consumers -> ~2 each
6 consumers -> ~1 each
9 consumers -> ~3 idle
```

Поэтому autoscaling workers выше partition count часто бессмыслен.

## Reliable processing

Типичная логика:

```text
poll
 -> validate
 -> idempotent business action
 -> persist
 -> commit offset
```

Если process crash после side effect, но до offset commit, event может быть delivered повторно.

## Producer acks

```yaml
acks: all
```

Повышает durability ожиданием acknowledgement согласно ISR rules, но влияет на latency.

## Failure scenarios

### Wrong bootstrap
DNS/connect failure.

### Worker slow
Consumer lag растёт, Pods могут быть Running.

### Broker restart
Consumer group rebalance.

### Too many consumers
Extra Pods idle.

## Production metrics

- consumer lag;
- rebalance count;
- produce error/latency;
- ISR;
- under-replicated partitions;
- disk;
- broker/controller health.

## Важно

Не делайте Spring liveness зависимой от Kafka так, чтобы outage Kafka вызвал restart storm всех consumers/producers.
