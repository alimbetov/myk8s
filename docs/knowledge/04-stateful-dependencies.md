# 04 — Stateful dependencies: PostgreSQL, Kafka, RabbitMQ and storage

Проверено: 2026-09-20.

## Principle

StatefulSet предоставляет stable identity/storage primitives, но **не реализует** database replication, leader election, quorum, backup, PITR или safe upgrade semantics продукта.

Для production stateful platform предпочтение:
1. managed service, если допустимо;
2. зрелый Kubernetes Operator;
3. ручной StatefulSet — главным образом учебный/узкоспециализированный путь.

## Storage model

```text
Stateful workload
 -> PVC
 -> StorageClass
 -> CSI provisioner
 -> PV
 -> physical/cloud storage
```

StorageClass определяет provisioning policy. `reclaimPolicy` по умолчанию может быть `Delete`; это должно быть осознанным решением.

Для StatefulSet Kubernetes docs рекомендуют `ReadWriteOncePod` для production, когда driver поддерживает.

## PostgreSQL

Production checklist:
- primary/replica topology;
- synchronous/asynchronous replication choice;
- backup to independent object storage;
- restore test;
- WAL archive/PITR;
- anti-affinity/topology spread;
- PDB;
- credential rotation;
- connection pool sizing across all application replicas;
- schema migration compatibility.

Operator candidate for dedicated research: CloudNativePG.

Spring:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres-rw.database.svc:5432/orders
    hikari:
      maximum-pool-size: 10
      connection-timeout: 3000
```

Total connections roughly scale with:
`application replicas × pool size + admin/operator/background connections`.

## Kafka

Kafka availability depends on broker/controller quorum, replication factor, min.insync.replicas, storage durability and rack/zone placement. Do not model Kafka as "one StatefulSet + PVC".

Operator candidate: Strimzi.

Application must configure finite request/delivery timeouts, retry semantics and idempotency appropriate to producer/consumer role.

## RabbitMQ

RabbitMQ requires product-aware clustering, durable queues, quorum queue strategy where appropriate, disk/memory alarms, topology recovery and backup/definitions policy.

Official Kubernetes Cluster Operator exists in RabbitMQ ecosystem.

## Day-2 minimum

Для каждой stateful platform должен существовать runbook:
- scale;
- storage expansion;
- backup;
- restore;
- certificate/credential rotation;
- minor/major upgrade;
- node loss;
- volume loss;
- quorum loss;
- full cluster recovery.

Backup считается существующим только после регулярного restore test.

## RPO/RTO

**RPO** — сколько данных допустимо потерять.  
**RTO** — сколько времени допустимо восстанавливать service.

Эти значения определяют replication, backup frequency, WAL/log retention и DR topology — не наоборот.

## Anti-patterns

- PostgreSQL Deployment с ephemeral disk;
- один PVC, примонтированный несколькими database Pods без понимания access mode;
- backup на тот же volume;
- отсутствие restore drills;
- PDB, блокирующий любой node drain;
- scale stateful cluster через `kubectl scale` без product semantics.

## Sources

- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/storage-classes/
- https://cloudnative-pg.io/documentation/
- https://strimzi.io/documentation/
- https://www.rabbitmq.com/kubernetes/operator/operator-overview

Проверено: **2026-09-20**.
