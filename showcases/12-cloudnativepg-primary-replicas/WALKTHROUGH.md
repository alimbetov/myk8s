# Walkthrough — PostgreSQL + CloudNativePG

## Архитектура

```text
orders-api
   |
   | writes
   v
orders-db-rw
   |
   v
PRIMARY
   |
   +--> replica 1
   +--> replica 2

report-api
   |
   | reads
   v
orders-db-ro
   |
   +--> replicas
```

## Файлы

- [README](README.md)
- [Cluster + application manifests](all.yaml)
- [Spring datasource examples](application.yaml)

## Operator responsibility

CloudNativePG управляет role topology.

Application не должна:
- искать primary Pod;
- менять endpoint после failover;
- делать leader election PostgreSQL самостоятельно.

## Logical Services

```text
orders-db-rw -> current primary
orders-db-ro -> replicas
orders-db-r  -> any instance
```

Главная abstraction:

```text
Spring JDBC URL
 -> logical role Service
 -> current database role
 -> Pod
```

## Write workload

```yaml
jdbc:postgresql://orders-db-rw:5432/orders
```

При failover host string не меняется.

## Read replicas

Reporting может читать:

```text
orders-db-ro
```

### Осторожно: replication lag

```text
write primary
 -> COMMIT
 -> immediate read replica
 -> old result possible
```

Если use case требует read-your-write, read replica может быть неправильным endpoint.

## instances=3

Обычно:

```text
1 primary
2 replicas
```

Но это не гарантирует zero data loss при любом failure.

RPO зависит от replication/failure/storage configuration.

## Persistent storage

Pod replacement не означает empty database.

Но:

```text
persistence != backup
replication != backup
```

Logical deletion также реплицируется.

## Backup/PITR

Нужны отдельно:
- backup destination;
- WAL archive;
- retention;
- restore test;
- RPO/RTO.

## Hikari effect

```text
10 application Pods
× Hikari 10
≈ up to 100 client DB connections
```

Failover может вызвать reconnect burst.

## Failure scenarios

### Delete primary Pod
Observe operator and `-rw` Service.

### Wrong password
Database cluster healthy, application auth fails.

### Read stale replica
Demonstrate consistency trade-off.

### Storage failure
Check operator status + PVC/PV events.

## Diagnostics

```bash
kubectl get cluster.postgresql.cnpg.io
kubectl get pod -l cnpg.io/cluster=orders-db
kubectl get svc | grep orders-db
kubectl get endpointslice
kubectl get pvc
```

## Главное operational правило

Сначала operator/product state, потом Pod restart. Blind `kubectl delete pod` может уничтожить полезные incident evidence.
