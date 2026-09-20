# 04 — Stateful dependencies: PostgreSQL, Kafka, RabbitMQ and storage

## Учебная карта темы

### Где живёт state

```text
Spring Boot Pod
    |
    +--> PostgreSQL Service --> operator-managed DB cluster
    |
    +--> Kafka bootstrap ----> operator-managed Kafka
    |
    +--> RabbitMQ Service ---> operator-managed RabbitMQ
    |
    +--> S3 endpoint --------> object storage
    |
    +--> PVC ----------------> PV/CSI storage
```

Главная идея: **Pod disposable, business state — нет**.

### StatefulSet и Operator — разные уровни

```text
StatefulSet
  -> stable name
  -> stable storage
  -> ordering

Operator
  -> product replication
  -> role management
  -> failover
  -> upgrades
  -> backup integration
```

Поэтому:

```text
StatefulSet != PostgreSQL HA
StatefulSet != Kafka cluster management
StatefulSet != RabbitMQ quorum
```

Смотрите: [CloudNativePG](../../showcases/12-cloudnativepg-primary-replicas/README.md), [Kafka/Strimzi](../../showcases/10-kafka-producer-consumer/README.md), [RabbitMQ](../../showcases/11-rabbitmq-worker/README.md), [StatefulSet](../../showcases/18-statefulset-headless-service/README.md).

Проверено: 2026-09-20.

Эта глава объясняет, почему «запустить PostgreSQL в Kubernetes» и «эксплуатировать PostgreSQL production-grade» — совершенно разные задачи.

## 1. Stateless и stateful: главное различие

Обычный Spring Boot REST Pod можно удалить:

```text
Pod A deleted
 -> Deployment creates Pod B
 -> application continues
```

Потому что business data находится не в самом Pod.

Для database:

```text
PostgreSQL Pod deleted
 -> новый Pod без данных
 => disaster
```

если данные находились только в container filesystem.

Поэтому stateful system требует отдельного storage lifecycle.

---

## 2. PVC/PV/StorageClass: mental model

```text
application Pod
   |
   | volumeMount
   v
PVC
   |
   | claim
   v
PV
   |
   | provisioned by
   v
StorageClass / CSI driver
   |
   v
real disk / SAN / cloud volume
```

### PVC

Application-side request:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteOncePod
  resources:
    requests:
      storage: 20Gi
```

### PV

Actual cluster storage resource.

### StorageClass

Описывает **как** storage создаётся:
- CSI provisioner;
- storage tier;
- reclaim policy;
- expansion support;
- binding behavior.

---

## 3. Почему Pod volume и persistent volume — не одно и то же

Container filesystem живёт вместе с container/Pod lifecycle.

PVC/PV должен переживать Pod replacement.

```text
Pod postgres-0
     |
     X deleted

PVC data-postgres-0
     |
     + remains

new postgres-0
     |
     + mounts same claim
```

Это storage persistence, но ещё не database HA.

---

## 4. StatefulSet: что он даёт

StatefulSet полезен, когда нужны:
- stable Pod names;
- stable network identity;
- stable per-Pod storage;
- ordered start/stop/update.

Пример identity:

```text
postgres-0
postgres-1
postgres-2
```

В отличие от Deployment:

```text
orders-6cd7d9b5d4-x7m2z
```

### Что StatefulSet НЕ даёт

Сам по себе он не реализует:
- PostgreSQL streaming replication;
- leader election;
- Kafka controller quorum;
- RabbitMQ quorum queue semantics;
- backup;
- PITR;
- automatic DB failover;
- application-aware upgrade.

Это принципиальная граница Kubernetes primitive.

---

## 5. Почему Operator обычно лучше manual StatefulSet

Manual StatefulSet знает Kubernetes lifecycle, но не знает product semantics.

Operator может знать:

```text
PostgreSQL:
  кто primary?
  кто replica?
  как promote replica?
  как archive WAL?
  как выполнить switchover?
  как обновить cluster?

Kafka:
  как менять broker config?
  как обновлять brokers?
  как управлять certificates/users/topics?
```

Поэтому production preference обычно:

```text
managed service
   |
   v
mature Kubernetes Operator
   |
   v
manual StatefulSet only when justified
```

---

## 6. Access modes: важная историческая деталь

Старый распространённый выбор:

```yaml
accessModes:
  - ReadWriteOnce
```

Но ``ReadWriteOnce`` означает по сути single-node mount, а не strict single-Pod writer. Несколько Pods на одном node могут потенциально использовать такой volume.

Для strict single-Pod access при поддерживаемом CSI Kubernetes имеет:

```yaml
accessModes:
  - ReadWriteOncePod
```

Это stable feature и более точная semantic для workloads, которым нужен действительно один writer.

> **Практика:** storage access mode выбирается по semantics приложения и возможностям CSI driver, а не по привычке.

---

## 7. ReclaimPolicy

StorageClass/PV может иметь reclaim policy.

Два важных поведения:

```text
Delete
Retain
```

### Delete

Удаление claim может привести к удалению backing storage.

### Retain

Storage остаётся и требует отдельной administrative cleanup/recovery procedure.

**Почему важно:** production database нельзя разворачивать, не понимая, что произойдёт с physical volume после удаления PVC.

---

## 8. PostgreSQL: connection path

Spring Boot не должен знать Pod primary по IP.

Предпочтительная logical endpoint модель:

```text
orders-service
   |
   v
postgres-rw Service
   |
   v
current writable primary
```

Application:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres-rw.database.svc:5432/orders
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
```

Operator может переключить Service на новый primary после failover.

---

## 9. PostgreSQL HA

Минимальные понятия:

```text
primary
  |
  +--> replica A
  +--> replica B
```

При failure primary нужна корректная promotion/failover procedure.

### Sync vs async replication

**Synchronous** уменьшает потенциальную потерю committed data, но может ухудшать latency/availability.

**Asynchronous** лучше отделяет primary от replica latency, но при catastrophic primary loss может потерять последние ещё не replicated transactions.

Это trade-off RPO vs latency/availability.

---

## 10. RPO и RTO

### RPO

Сколько данных допустимо потерять.

Пример:

```text
RPO = 5 minutes
```

означает, что disaster recovery architecture должна стремиться не потерять больше примерно пяти минут data.

### RTO

Сколько времени допустимо восстанавливать service.

```text
RTO = 30 minutes
```

означает, что restore/failover procedure должна укладываться примерно в это operational objective.

RPO/RTO определяют backup/replication/DR design, а не наоборот.

---

## 11. Backup != копия PVC

Database backup должен быть **application-consistent**.

Для PostgreSQL это может включать:
- base backup;
- WAL archive;
- PITR;
- operator backup mechanism.

Простое копирование filesystem живой DB без понимания database consistency может не дать usable restore.

### Главное правило

> Backup считается существующим только после проверенного restore.

---

## 12. PITR

Point-in-Time Recovery позволяет восстановить database примерно к моменту перед логической ошибкой.

Сценарий:

```text
10:00 good data
10:13 accidental DELETE
10:20 incident detected

restore base backup
+ replay WAL until 10:12:59
```

Это защищает от класса ошибок, от которого replication не спасает: неправильный DELETE реплицируется на replicas тоже.

---

## 13. Connection pool и replicas приложения

Spring Boot pool:

```yaml
maximum-pool-size: 10
```

Deployment:

```yaml
replicas: 20
```

Potential connections:

```text
20 × 10 = 200
```

Это без:
- admin tools;
- migrations;
- reporting;
- operators;
- monitoring.

Следовательно, Horizontal scaling backend может перегрузить PostgreSQL даже при низкой CPU приложения.

---

## 14. Kafka: почему просто StatefulSet недостаточно

Kafka имеет свои понятия:
- brokers;
- controller quorum;
- replication factor;
- ISR;
- partition placement;
- rack/zone awareness;
- retention;
- storage;
- rolling broker updates.

```text
Kubernetes StatefulSet
  -> stable broker Pods/storage

Kafka
  -> data replication/quorum semantics
```

Operator вроде Strimzi добавляет product-aware lifecycle.

---

## 15. RabbitMQ

RabbitMQ production design включает:
- cluster membership;
- quorum queues;
- durable data;
- memory/disk alarms;
- certificates/users;
- topology;
- upgrade semantics.

```text
PVC alone != durable messaging architecture
```

Official Kubernetes operators помогают автоматизировать product lifecycle, но всё равно требуют понимания RabbitMQ semantics.

---

## 16. Object storage

Spring Boot часто лучше хранит files не в local filesystem Pod, а в object storage:

```text
application
 -> S3-compatible API
 -> object storage
```

Преимущества:
- Pod replicas не делят local disk;
- storage lifecycle отделён от application;
- проще retention/versioning/backup policies.

Local mounted PVC нужен только если workload semantics действительно требуют filesystem interface.

---

## 17. PDB и stateful systems

PDB может помочь при voluntary node drain, но опасно задавать его без понимания quorum.

Пример: cluster из 3 nodes требует quorum 2.

Если одновременно потерять слишком много members, service перестанет быть available.

Но слишком строгий PDB способен заблокировать administrative maintenance.

Operator/product docs должны определять правильную disruption strategy.

---

## 18. Affinity / anti-affinity / topology

Плохая HA:

```text
postgres-0 -> worker-1
postgres-1 -> worker-1
postgres-2 -> worker-1
```

Один node failure убивает весь cluster.

Лучше распределять replicas:

```text
postgres-0 -> worker-1 / zone-a
postgres-1 -> worker-2 / zone-b
postgres-2 -> worker-3 / zone-c
```

Но только если underlying storage/network topology также поддерживает такой design.

---

## 19. Failure practice: Pod deleted

Для stateful workload:

```bash
kubectl delete pod postgres-0
kubectl get pod -w
kubectl get pvc
```

Нужно увидеть:
- Pod recreation;
- сохранение PVC;
- повторное attach/mount;
- product recovery.

### Вопрос для проверки

Если новый Pod появился, но database не стартует — проблема уже может быть:
- filesystem;
- WAL;
- ownership;
- corrupted data;
- operator state.

---

## 20. Failure practice: PVC Pending

```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get storageclass
kubectl get events --sort-by=.lastTimestamp
```

Причины:
- нет default StorageClass;
- requested access mode не поддержан;
- CSI provisioner problem;
- topology/binding constraints;
- capacity exhausted.

---

## 21. Failure practice: disk full

Симптомы:
- database writes fail;
- RabbitMQ disk alarm;
- Kafka broker unhealthy;
- Pod/node disk pressure.

Диагностировать:
- PVC usage metrics;
- node filesystem;
- application logs;
- storage expansion support.

**Антипаттерн:** реагировать на 100% disk только restart Pods.

Restart не создаёт место.

---

## 22. Failure practice: primary node lost

Нужно проверить:
1. кто определяет failure;
2. кто promote replica;
3. как меняется write Service;
4. как reconnect приложение;
5. были ли потеряны transactions;
6. какой фактический RPO/RTO.

Это должен быть заранее отрепетированный runbook.

---

## 23. Upgrade

Stateful upgrade — не просто:

```bash
kubectl set image statefulset/postgres ...
```

Нужно учитывать:
- version compatibility;
- upgrade order;
- schema/system catalog changes;
- rollback feasibility;
- backups;
- quorum availability;
- operator-supported path.

---

## 24. Developer vs Platform/DBA

| Область | Developer | Platform/DBA | Shared |
|---|---:|---:|---:|
| JDBC URL contract | ✓ |  | ✓ |
| connection pool | ✓ |  | ✓ |
| DB cluster topology |  | ✓ |  |
| StorageClass/CSI |  | ✓ |  |
| schema migrations | ✓ |  | ✓ |
| backup/restore |  | ✓ | ✓ |
| RPO/RTO |  |  | ✓ |
| failover testing |  |  | ✓ |
| operator lifecycle |  | ✓ |  |

---

## 25. Anti-patterns

- PostgreSQL Deployment + emptyDir;
- считать StatefulSet HA;
- backup на тот же volume;
- никогда не тестировать restore;
- scale Kafka/Rabbit/PostgreSQL вслепую через kubectl scale;
- одинаковая HA replica placement на одном node;
- не мониторить storage capacity;
- считать replication заменой backup;
- считать backup заменой PITR;
- увеличивать Spring replicas, не считая DB connections.

---

## 26. CKAD vs production

CKAD:
- PVC;
- volume mount;
- StorageClass basics;
- StatefulSet basics.

Production:
- operator;
- quorum;
- replication;
- failover;
- backup;
- PITR;
- restore drills;
- capacity;
- DR;
- RPO/RTO.

---

## Sources: для проверки

- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/ — stable identity/storage и StatefulSet limitations.
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/ — PV/PVC access modes.
- https://kubernetes.io/docs/concepts/storage/storage-classes/ — dynamic provisioning/reclaim policy.
- https://cloudnative-pg.io/documentation/ — PostgreSQL operator patterns.
- https://strimzi.io/documentation/ — Kafka on Kubernetes.
- https://www.rabbitmq.com/kubernetes/operator/operator-overview — RabbitMQ operator.

Проверено: **2026-09-20**.
