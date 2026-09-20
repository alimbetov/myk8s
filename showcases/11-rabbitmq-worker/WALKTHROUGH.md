# Walkthrough — RabbitMQ worker + Cluster Operator

## Архитектура

```text
producer
   |
   | AMQP
   v
rabbit-main:5672
   |
   v
RabbitMQ cluster
   |
   | queue
   v
document-worker Pods
```

## Файлы

- [README](README.md)
- [Manifests](all.yaml)
- [Spring Rabbit config](application.yaml)

## Operator boundary

`RabbitmqCluster` — desired product cluster.

Operator создаёт/управляет:
- StatefulSet;
- Services;
- PVCs;
- cluster bootstrap resources.

Не patch-ить operator-owned StatefulSet как основной management interface.

## Client Service

Spring использует:

```text
rabbit-main:5672
```

а не specific broker Pod.

## Headless Service

`rabbit-main-nodes` нужен broker peer discovery.

Это не тот endpoint, который обычно передают Spring AMQP client.

## Config mapping

ConfigMap:
- RABBITMQ_HOST
- RABBITMQ_PORT
- WORK_QUEUE

Secret:
- username
- password

Spring:

```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST}
    username: ${RABBITMQ_USERNAME}
```

## Queue topology ownership

Cluster Operator != queue topology manager.

Варианты:
- application declaration;
- definitions;
- Messaging Topology Operator.

Нужно выбрать ownership, чтобы queue не создавалась несколькими независимыми системами.

## Quorum queues

Для replicated durable queue важна RabbitMQ quorum semantics.

PVC не делает queue replicated автоматически.

```text
PVC = node persistence
quorum queue = RabbitMQ data replication/consensus
```

## Consumer capacity

```text
worker Pods × listener concurrency = consumers
```

Например:

```text
5 Pods × 4 = 20 consumers
```

Это может нагрузить DB/external API сильнее, чем RabbitMQ.

## Graceful termination

Worker должен корректно:
- stop receiving new deliveries;
- finish or nack current delivery;
- close channel;
- avoid silent message loss.

## Failure scenarios

### Wrong password
TCP connection есть, AMQP auth fails.

### Queue grows
Pods healthy, business latency grows.

### Broker disk alarm
Kubernetes Pod Running, но RabbitMQ flow control меняет поведение.

### Overscale workers
Queue уменьшается, downstream collapses.

## Metrics

- ready messages;
- unacked;
- publish/delivery rate;
- consumers;
- connections;
- disk/memory alarms.

## Главное

Kubernetes health и RabbitMQ product health — не одно и то же.
