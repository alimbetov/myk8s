# Showcase 11 — RabbitMQ worker with Cluster Operator

Проверено: 2026-09-20.

## Схема

```text
producer
   |
   | AMQP
   v
Service rabbit-main:5672
   |
   v
RabbitMQ Cluster Operator managed cluster
   |
   v
worker Deployment
```

## Prerequisite

RabbitMQ Cluster Operator CRD:

```text
rabbitmqclusters.rabbitmq.com
```

должен быть установлен.

## RabbitmqCluster

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbit-main
spec:
  replicas: 3
```

Operator создаёт StatefulSet, Services, Secrets и supporting resources.

RabbitMQ рекомендует odd replica counts; production cluster обычно 3 nodes, а не 2.

## Services

Для cluster `rabbit-main` operator создаёт:
- `rabbit-main` — client Service, включая AMQP 5672;
- `rabbit-main-nodes` — headless peer discovery Service.

Spring client подключается к:

```text
rabbit-main:5672
```

а не к `rabbit-main-server-0`.

## Persistence

```yaml
persistence:
  storage: 20Gi
```

Если StorageClass не указан, используется cluster default.

Если default StorageClass отсутствует, RabbitMQ Pods могут остаться Pending.

## Credentials

Cluster Operator создаёт default user Secret. Для production authentication lifecycle следует проектировать отдельно; application credentials не нужно hardcode в ConfigMap.

В стенде application Secret показан отдельно, чтобы connection contract был очевиден.

## Queue semantics

Для durable replicated workloads production обычно рассматривает quorum queues.

Queue topology можно:
- создавать application code;
- definitions;
- Messaging Topology Operator.

Это отдельная responsibility от Cluster Operator.

## Worker scaling

```text
queue depth
 -> consumers
 -> DB/external capacity
```

CPU HPA не всегда лучший сигнал. Для queue workers часто полезнее backlog/external metric.

## Failure simulations

1. Wrong Service name -> DNS failure.
2. Secret wrong -> AMQP authentication failure.
3. No default StorageClass -> cluster Pods Pending.
4. Scale worker aggressively -> downstream overload.
5. Rabbit disk/memory alarm -> broker flow control despite Pods Running.

## Проверка

```bash
kubectl get rabbitmqcluster
kubectl get statefulset
kubectl get svc | grep rabbit-main
kubectl get secret | grep rabbit-main
kubectl get pvc
```

## Sources

- https://www.rabbitmq.com/kubernetes/operator/operator-overview
- https://www.rabbitmq.com/kubernetes/operator/using-operator
