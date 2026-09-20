# Showcase 12 — PostgreSQL primary/replicas with CloudNativePG

## Файлы стенда

- [Подробный walkthrough](WALKTHROUGH.md)
- [Полный связный manifest](all.yaml)
- [Spring Boot configuration](application.yaml)
- [Общая шпаргалка mapping'ов](../MAPPING-CHEATSHEET.md)

Проверено: 2026-09-20.

## Схема

```text
orders-api writes
      |
      v
orders-db-rw
      |
      v
current PRIMARY

report-api reads
      |
      v
orders-db-ro
      |
      +--> replica
      +--> replica
```

CloudNativePG управляет role changes. Application работает со stable Services.

## Prerequisite

Установлен CloudNativePG Operator и CRD:

```text
clusters.postgresql.cnpg.io
```

## Cluster

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: orders-db
spec:
  instances: 3
  storage:
    size: 20Gi
```

`instances: 3` означает один primary и два replica instances в нормальном состоянии.

## Operator-created Services

CloudNativePG по умолчанию создаёт:

```text
orders-db-rw -> current primary
orders-db-ro -> replicas
orders-db-r  -> any instance
```

## Write application

```text
jdbc:postgresql://orders-db-rw:5432/orders
```

При failover operator перенаправляет `-rw` к promoted replica. Application не меняет host config.

## Read-only application

```text
jdbc:postgresql://orders-db-ro:5432/orders
```

Read replicas могут иметь replication lag. Нельзя автоматически отправлять туда use cases, которым нужен read-your-write consistency.

## Storage

Каждый PostgreSQL instance получает persistent storage.

Pod replacement != data deletion.

Но replication != backup.

## Bootstrap credentials

CR example использует bootstrap initdb database/owner и Secret. Реальный password lifecycle должен управляться безопасно и отдельно.

## Failure simulations

1. Delete primary Pod -> operator promotes/recovers according to cluster state.
2. Application configured to Pod name instead of `-rw` -> failover breaks client.
3. Report reads replica immediately after write -> possible stale result.
4. PVC/storage failure -> HA depends on remaining replicas and topology.
5. Delete/corrupt logical data -> replication copies logical damage; need backup/PITR.

## Проверка

```bash
kubectl get cluster.postgresql.cnpg.io
kubectl get pod -l cnpg.io/cluster=orders-db
kubectl get svc | grep orders-db
kubectl get pvc
```

## Sources

- https://cloudnative-pg.io/docs/1.29/
- CloudNativePG service management: rw, ro, r Services.
