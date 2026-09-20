# 04 — Stateful dependencies: persistence, replication, HA, backup и operators

Проверено: 2026-09-20.

Эта глава отвечает на следующий вопрос после Deployment lifecycle:

> Мы научились безопасно удалять и заменять Spring Boot Pods. **Но где тогда должны жить данные, что именно сохраняет PVC, зачем нужны replicas, quorum и operators, и почему replication всё ещё не является backup?**

Главная mental model главы:

```text
Pod replacement
      |
      v
persistent storage
      |
      v
product replication
      |
      v
high availability / failover
      |
      v
backup / PITR
      |
      v
disaster recovery
```

Каждый следующий слой решает **другую проблему**.

Нельзя заменить один другим.

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь объяснить:

1. Чем stateless workload отличается от stateful.
2. Почему container filesystem нельзя использовать как durable business storage.
3. Чем Pod volume отличается от PersistentVolume.
4. Что такое PVC, PV, StorageClass и CSI.
5. Что реально означают RWO, RWX и RWOP.
6. Почему RWO не означает strict single-Pod writer.
7. Что делает reclaimPolicy.
8. Зачем volumeBindingMode=WaitForFirstConsumer.
9. Что даёт StatefulSet.
10. Чего StatefulSet **не** даёт.
11. Зачем headless Service.
12. Чем persistence отличается от replication.
13. Чем replication отличается от HA.
14. Что такое quorum.
15. Что такое failover и switchover.
16. Почему backup не равен snapshot/PVC.
17. Почему replication не заменяет backup.
18. Что такое PITR.
19. Что такое RPO и RTO.
20. Почему restore test важнее факта “backup job green”.
21. Зачем product-aware Operator.
22. Как Spring Boot должен подключаться к PostgreSQL через logical Services.
23. Почему read replicas могут возвращать stale data.
24. Почему Hikari pool связан с replica count приложения.
25. Почему Kafka partitions и replicas решают разные задачи.
26. Почему RabbitMQ PVC не делает queue replicated.
27. Когда PVC лучше S3, а когда S3 лучше PVC.
28. Что должен проверять аналитик, разработчик и тестировщик.

---

# 2. Одна схема всех stateful слоёв

Сначала посмотрим на систему сверху.

```text
                     Spring Boot
                         |
        +----------------+-------------------+
        |                |                   |
        v                v                   v
   PostgreSQL           Kafka             RabbitMQ
        |                |                   |
        v                v                   v
 logical Service   bootstrap Service    client Service
        |                |                   |
        v                v                   v
 product topology / roles / replication / quorum
        |                |                   |
        +----------------+-------------------+
                         |
                         v
                  Persistent storage
                         |
                         v
                 PVC -> PV -> CSI
                         |
                         v
                  physical storage

Отдельно:
Spring Boot -> S3 API -> Object Storage
```

Главный принцип:

> **Kubernetes storage primitives обеспечивают storage attachment/lifecycle. Product-aware systems обеспечивают semantics данных.**

---

# 3. Stateless и stateful: фундаментальная разница

Stateless Spring Boot Pod:

```text
orders Pod A
   |
   X deleted
   |
   v
orders Pod B

business state remains elsewhere
```

Если приложение корректно спроектировано, replacement не требует восстановления business data из container filesystem.

Stateful workload:

```text
PostgreSQL
Kafka
RabbitMQ
filesystem-based worker
```

имеет durable state, который нельзя потерять вместе с Pod.

---

# 4. Почему container filesystem не является durable storage

Container может писать:

```text
/app/data/order.pdf
```

Но replacement Pod получает новый writable layer.

```text
Pod A
  /app/data/order.pdf
        |
        X Pod deleted

Pod B
  /app/data/
        |
        empty/new filesystem
```

Поэтому container local filesystem подходит для:

- temporary files;
- cache, который можно восстановить;
- scratch workspace;
- immutable packaged files.

Но не для critical business state.

---

# 5. Первый stateful слой: volume

Pod может подключить volume.

```text
container
   |
   | /data
   v
volumeMount
   |
   v
Pod volume
```

Но volume type определяет lifecycle.

Например `emptyDir`:

```text
Pod starts
  ↓
emptyDir created

Pod removed
  ↓
emptyDir data gone
```

То есть:

```text
volume
!=
persistent volume
```

---

# 6. PVC, PV и StorageClass

Persistent storage chain:

```text
Application Pod
      |
      | volumeMount
      v
Pod volume
      |
      | claimName
      v
PVC
      |
      | binds to
      v
PV
      |
      | provisioned by
      v
StorageClass / CSI driver
      |
      v
actual storage backend
```

Это одна из важнейших Kubernetes storage mental models.

---

# 7. PVC: request for storage

Пример:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: orders-work

spec:
  accessModes:
    - ReadWriteOncePod

  resources:
    requests:
      storage: 20Gi
```

PVC говорит примерно:

> Мне нужен volume такого размера и с такими access semantics.

Он не описывает конкретный LUN/device/cloud disk напрямую.

---

# 8. PV: storage resource

PersistentVolume представляет storage resource cluster.

Mental model:

```text
PVC
 = demand/request

PV
 = supply/resource
```

Binding:

```text
PVC Pending
    |
    | matching/provisioning
    v
PVC Bound
    |
    v
PV assigned
```

---

# 9. StorageClass: “какой класс storage мне нужен”

StorageClass описывает provisioning policy.

Например:

```text
fast-ssd
standard
archive
local-path
nfs-rwx
```

Его attributes могут включать:

- provisioner;
- storage parameters;
- reclaimPolicy;
- volumeBindingMode;
- expansion capability;
- topology constraints.

Приложение просит class по имени, а platform решает, что физически за ним стоит.

---

# 10. CSI: кто реально умеет подключать storage

CSI — Container Storage Interface.

Упрощённо:

```text
Kubernetes
    |
    v
CSI driver
    |
    v
storage backend
```

Driver знает, как:

- provision volume;
- attach;
- mount;
- expand;
- detach;
- delete.

Поэтому одинаковый PVC manifest может вести себя по-разному на разных storage backends.

---

# 11. Access modes: RWO, ROX, RWX, RWOP

Основные access modes:

```text
ReadWriteOnce      RWO
ReadOnlyMany       ROX
ReadWriteMany      RWX
ReadWriteOncePod   RWOP
```

Но названия легко понять неправильно.

---

# 12. RWO НЕ означает один Pod

`ReadWriteOnce` означает, что volume может быть mounted read-write на **одном node**.

Это не строгая гарантия:

```text
one Pod only
```

Если два Pods находятся на одном node и storage/driver допускает, они потенциально могут работать с тем же mounted volume.

Поэтому:

```text
RWO
≈ single-node write semantics

не
strict single-Pod writer
```

---

# 13. RWOP — strict single-Pod access

Для strict single-Pod access есть:

```yaml
accessModes:
  - ReadWriteOncePod
```

RWOP stable с Kubernetes 1.29 и поддерживается только для CSI volumes с соответствующей поддержкой driver/sidecars.

Mental model:

```text
RWOP
  |
  v
this PVC can be mounted read-write by only one Pod
```

Перед использованием на k3s/production обязательно проверить actual CSI driver support.

---

# 14. RWX — не “автоматически безопасный concurrent filesystem”

`ReadWriteMany` означает возможность mount read-write несколькими nodes.

Но приложение должно отдельно уметь безопасно работать с shared filesystem.

```text
RWX storage
   |
   X
does not automatically solve
   |
   +--> file locking
   +--> concurrent writes
   +--> consistency
   +--> application-level races
```

Storage capability и application concurrency semantics — разные вещи.

---

# 15. ReclaimPolicy: что будет после удаления claim

Для dynamically provisioned PV StorageClass обычно определяет reclaim policy.

Важные варианты:

```text
Delete
Retain
```

## Delete

После release claim backing storage может быть удалён.

## Retain

PV/data остаются для manual reclamation/recovery.

Критичный вопрос перед production DB:

> Что произойдёт с physical data, если кто-то удалит PVC?

Нельзя узнавать это во время incident.

---

# 16. Default reclaimPolicy для dynamically provisioned storage

Если StorageClass не задаёт reclaimPolicy, для dynamically provisioned volumes default — `Delete`.

Поэтому опасно думать:

> Kubernetes ведь persistent, значит delete PVC не страшно.

Persistent означает “переживает Pod”, а не “никогда не будет удалено”.

---

# 17. volumeBindingMode

Важные варианты:

```text
Immediate
WaitForFirstConsumer
```

## Immediate

Volume provision/binding происходит при создании PVC.

## WaitForFirstConsumer

Binding откладывается, пока не появится Pod, использующий claim.

Зачем?

При topology-aware storage:

```text
Pod must run in zone-b
        |
        v
volume should also be provisioned in zone-b
```

Без ожидания можно заранее создать volume в неподходящей topology и получить unschedulable Pod.

---

# 18. PVC переживает Pod replacement

```text
Pod file-worker-A
      |
      X deleted

PVC work-data
      |
      + remains

replacement Pod
      |
      + mounts work-data
```

Это persistence.

Но пока мы всё ещё не получили:

- replication;
- failover;
- quorum;
- backup.

---

# 19. Persistence != replication

Представьте один PostgreSQL instance:

```text
PostgreSQL Pod
      |
      v
PVC
      |
      v
durable disk
```

Pod умер — data осталось.

Но если сам storage damaged/corrupted:

```text
one copy
   |
   X
storage failure
```

Replication создаёт дополнительные copies на уровне продукта.

---

# 20. Replication != high availability

Допустим:

```text
primary
  |
  +--> replica A
  +--> replica B
```

Copies есть.

Но если primary умер:

> Кто обнаружит failure? Кто выберет новый primary? Кто переключит client endpoint? Как избежать split-brain?

Если ответ “делаем вручную через два часа”, replication есть, а высокий availability — нет.

---

# 21. High availability

HA — способность системы продолжить/быстро восстановить service при expected failures.

Упрощённо:

```text
failure detection
      ↓
decision
      ↓
role transition
      ↓
client routing update
      ↓
reconnect/recovery
```

HA требует не только copies, но и orchestration failure semantics.

---

# 22. Failover vs switchover

## Failover

Незапланированная смена active role после failure.

```text
primary dead
   ↓
replica promoted
```

## Switchover

Плановая смена role, например для maintenance.

```text
healthy primary
   ↓ controlled transition
healthy replica becomes primary
```

Switchover полезен для проверки procedures без аварии.

---

# 23. Quorum

Для distributed consensus systems часто нужен majority.

Три members:

```text
A
B
C
```

majority:

```text
2 of 3
```

Если доступны только один:

```text
1 of 3
   ↓
no majority
   ↓
operations requiring quorum stop
```

Quorum — это product/distributed-system semantics, а не свойство PVC.

---

# 24. Почему нечётное количество members часто встречается

В consensus group:

```text
3 members -> tolerate 1 failure

4 members -> majority 3
             tolerate 1 failure

5 members -> majority 3
             tolerate 2 failures
```

Поэтому четвёртый voting member часто увеличивает cost без увеличения failure tolerance по сравнению с 3.

Но конкретную topology всегда определяет product documentation.

---

# 25. StatefulSet: где он находится

StatefulSet даёт Kubernetes primitives, полезные stateful applications.

```text
Headless Service
      |
      v
StatefulSet
      |
      +--> member-0
      +--> member-1
      +--> member-2
             |
             v
       per-Pod PVCs
```

Он управляет Pods, но сохраняет sticky identity.

---

# 26. Что StatefulSet даёт

Kubernetes StatefulSet обеспечивает:

- stable unique Pod identity;
- stable ordinal;
- stable network identity;
- stable persistent storage association;
- ordered deployment/scaling by default;
- ordered rolling updates.

Пример:

```text
database-0
database-1
database-2
```

После reschedule:

```text
database-1
```

остаётся logical identity `1`, хотя node/IP может измениться.

---

# 27. StatefulSet требует governing headless Service

Для stable network identity StatefulSet использует headless Service.

```text
member-0.cluster-member
member-1.cluster-member
member-2.cluster-member
```

Headless Service:

```yaml
spec:
  clusterIP: None
```

Он нужен для peer identity/discovery, а не для обычного load-balanced business endpoint.

---

# 28. OrderedReady

Default Pod management policy StatefulSet:

```text
OrderedReady
```

Conceptually:

```text
start member-0
wait Ready
   ↓
start member-1
wait Ready
   ↓
start member-2
```

При scale down порядок обратный.

Но не все distributed systems хотят strict ordering — существует `Parallel` policy.

---

# 29. StatefulSet НЕ делает продукт HA

Запомните:

```text
StatefulSet
=
identity
+
ordering
+
storage association

StatefulSet
!=
PostgreSQL replication
!=
Kafka partition replication
!=
Rabbit quorum queue
!=
leader election
!=
backup
!=
PITR
```

Это одна из центральных идей главы.

---

# 30. Почему manual StatefulSet для PostgreSQL опасно воспринимать как готовое решение

Можно написать:

```text
3 PostgreSQL Pods
+
3 PVCs
```

Но остаются вопросы:

- кто primary?
- как replicas получают WAL?
- кто следит за replication lag?
- кто promotes replica?
- как исключить split-brain?
- кто обновляет write endpoint?
- кто делает backup?
- кто выполняет restore?
- как upgrade major version?
- кто reinitializes broken replica?

StatefulSet на эти вопросы не отвечает.

---

# 31. Operator: Kubernetes controller с domain knowledge

Operator использует тот же reconciliation pattern, но понимает продукт.

```text
Custom Resource
      |
      v
Operator
      |
      | domain-aware reconciliation
      v
Services
Pods
PVCs
Secrets
ConfigMaps
product operations
```

Пример:

```text
CloudNativePG Cluster
      |
      v
CloudNativePG Operator
      |
      +--> PostgreSQL instances
      +--> role management
      +--> Services
      +--> recovery lifecycle
```

---

# 32. Operator-owned resources: важный operational rule

Если Operator создал StatefulSet:

плохая идея:

```text
kubectl edit statefulset generated-by-operator
```

как основной способ управления.

Почему:

```text
you change generated child resource
       ↓
Operator reconciliation runs
       ↓
desired state from CR wins / drift corrected
```

Правильный management surface обычно:

```text
Custom Resource
```

---

# 33. PostgreSQL: application не должна знать primary Pod

Плохой JDBC URL:

```text
jdbc:postgresql://orders-db-1:5432/orders
```

Если `orders-db-1` перестал быть primary — application contract сломан.

Лучше:

```text
jdbc:postgresql://orders-db-rw:5432/orders
```

---

# 34. CloudNativePG logical Services

CloudNativePG создаёт role-based Services:

```text
<cluster>-rw
  -> primary

<cluster>-ro
  -> replicas

<cluster>-r
  -> any instance
```

Для cluster `orders-db`:

```text
orders-db-rw
orders-db-ro
orders-db-r
```

Spring write datasource:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://orders-db-rw:5432/orders
```

Operator может менять current primary, а application URL остаётся тем же.

---

# 35. PostgreSQL failover глазами приложения

До failure:

```text
orders-api
   |
   v
orders-db-rw
   |
   v
primary A
```

Primary A lost:

```text
operator detects failure
      ↓
replica B promoted
      ↓
orders-db-rw points to B
      ↓
existing client connections break
      ↓
Hikari reconnects
```

Важно:

> Logical Service скрывает изменение endpoint identity, но не делает existing TCP/DB connections immortal.

Application должна корректно переживать reconnect/transient failure.

---

# 36. Read replicas и stale reads

Reporting application может использовать:

```text
orders-db-ro
```

Но replication может иметь lag.

Сценарий:

```text
T0 write order status=PAID to primary
       ↓
COMMIT
       ↓
T0+20ms read from replica
       ↓
replica has not applied WAL yet
       ↓
status=NEW
```

Поэтому:

```text
read replica
!=
guaranteed read-your-write
```

Использование read replicas — consistency decision, а не просто performance optimization.

---

# 37. Synchronous vs asynchronous PostgreSQL replication

## Async

Primary может commit до подтверждения replica.

Плюсы:

- меньше dependency на replica latency;
- обычно выше write availability/latency characteristics.

Минус:

- при catastrophic primary loss последние not-yet-replicated transactions могут потеряться.

## Sync

Commit зависит от configured synchronous acknowledgement.

Плюсы:

- может улучшить data-loss guarantees.

Минусы:

- replica/network failure может повлиять на write latency/availability.

Это trade-off:

```text
RPO
vs
latency
vs
availability
```

---

# 38. Hikari connection budget

Spring Boot:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
```

Deployment:

```text
replicas = 20
```

Potential:

```text
20 × 10
≈ 200 application connections
```

Плюс:

- migrations;
- DBA sessions;
- operator;
- monitoring;
- reporting;
- admin tools.

Поэтому:

```text
HPA maxReplicas
×
Hikari pool size
```

должно входить в PostgreSQL capacity planning.

---

# 39. Connection storm после failover

После primary failure:

```text
20 application Pods
        |
        X all old DB connections
        |
        v
20 Hikari pools reconnect
        |
        v
new primary
```

Это может создать reconnect burst.

Поэтому нужны:

- bounded pool sizes;
- sensible connection timeouts;
- retry/backoff;
- database connection budget;
- failover tests.

---

# 40. Persistence != backup

Три replicas:

```text
primary
  |
  +--> replica A
  +--> replica B
```

User executes:

```sql
DELETE FROM payments;
```

Replication делает:

```text
DELETE
  ↓
replica A
replica B
```

Теперь три одинаково правильные копии неправильного состояния.

Replication защищает от некоторых infrastructure failures.

Она не защищает от logical corruption/user error.

---

# 41. Backup

Backup — отдельная recoverable copy/state history.

Для PostgreSQL это может быть:

- base backup;
- WAL archive;
- VolumeSnapshot с database-aware procedure;
- operator-managed backup mechanism.

Главный критерий:

```text
backup exists
   X

restore successfully tested
   ✓
```

Пока restore не проверен, backup — только предположение.

---

# 42. PITR

Point-in-Time Recovery отвечает на вопрос:

> Можно ли восстановить database к состоянию до логической ошибки?

Пример:

```text
10:00 base backup
10:00.. continuous WAL archive

14:32:10 accidental DELETE
14:40 incident detected
```

Recovery:

```text
restore base
   ↓
replay WAL
   ↓
stop at 14:32:09
```

Это другой механизм, чем failover.

---

# 43. Snapshot != автоматически application-consistent backup

Storage snapshot может capture disk blocks.

Но database имеет:

- memory buffers;
- WAL;
- filesystem state;
- checkpoint semantics.

Некоторые snapshot mechanisms/operators умеют делать consistent workflows.

Но нельзя автоматически считать:

```text
volume snapshot
=
verified database backup
```

Нужно понимать product-specific recovery semantics.

---

# 44. RPO

Recovery Point Objective:

> Какой объём/возраст данных бизнес допускает потерять?

Пример:

```text
RPO = 5 minutes
```

Это requirement к architecture.

Не:

> backup запускается каждые 24 часа, значит наш RPO 24 часа.

Сначала business requirement, потом technical design.

---

# 45. RTO

Recovery Time Objective:

> За какое время service должен быть восстановлен?

Пример:

```text
RTO = 30 minutes
```

RTO влияет на:

- failover automation;
- restore tooling;
- backup location;
- runbooks;
- staffing/on-call;
- amount of data to restore.

---

# 46. HA и DR — разные вещи

HA обычно относится к expected local failures:

```text
Pod loss
node loss
single instance failure
```

Disaster Recovery может рассматривать:

```text
cluster loss
region/site loss
operator error
mass corruption
ransomware/credential compromise
```

Один HA cluster не является автоматически DR strategy.

---

# 47. Backup должен быть независимо защищён

Плохая схема:

```text
database volume
   |
   +--> backup directory
        on same volume
```

Disk lost:

```text
database gone
backup gone
```

Backup должен учитывать independent failure domain и retention/security policy.

---

# 48. Kafka: state совсем другого типа

Kafka data model:

```text
Topic
  |
  +--> Partition 0
  +--> Partition 1
  +--> Partition 2
```

Partition replicas distributed across brokers.

Kubernetes primitives обеспечивают broker processes/storage.

Kafka itself обеспечивает:

- partitioning;
- replication;
- leader/follower roles;
- ISR;
- controller metadata;
- consumer offsets;
- retention.

---

# 49. Kafka partitions != replicas

Например:

```yaml
partitions: 6
replicas: 3
```

Не путать:

```text
partitions
  = units of ordering + parallelism

replicas
  = copies for durability/availability
```

Если consumer group содержит 10 consumers, а topic имеет 6 partitions:

```text
max useful active consumers ≈ 6
remaining consumers idle
```

---

# 50. KafkaNodePool и Strimzi

Современный Strimzi использует operator-managed resources, включая Kafka/KafkaNodePool.

```text
Kafka CR
   +
KafkaNodePool CR
      |
      v
Strimzi Cluster Operator
      |
      v
Kafka nodes
storage
listeners
certificates
rollouts
```

KafkaNodePool задаёт, среди прочего:

- replicas;
- roles: broker/controller;
- storage;
- resources.

Storage может быть persistent claim/JBOD.

---

# 51. KRaft roles

Node pool может иметь roles:

```text
broker
controller
```

В небольшом учебном cluster они могут быть combined.

В production topology роли могут разделяться.

Важно понимать:

```text
Kubernetes Pod ordinal
!=
Kafka controller/broker semantics
```

Product roles принадлежат Kafka.

---

# 52. Kafka storage lifecycle

Strimzi persistent storage configuration может включать:

```yaml
type: persistent-claim
size: 100Gi
deleteClaim: false
```

`deleteClaim: false` помогает не удалить claim автоматически при удалении Kafka node resource.

Но даже это не заменяет:

- replication;
- retention;
- backup/restore strategy;
- disaster recovery.

---

# 53. Kafka replication != external backup

Если producer publishes bad event:

```text
bad event
  ↓
replicated across brokers
```

Если topic retention deletes historical data, replicas делают то же самое.

Replication обеспечивает broker/data availability, а не immutable business history.

---

# 54. Consumer lag — часть stateful operations

Kafka cluster может быть healthy:

```text
brokers Running
ISR healthy
```

но application:

```text
consumer lag = 2 million
```

Business latency huge.

Поэтому product health и Kubernetes Pod health — разные dimensions.

---

# 55. RabbitMQ: storage + messaging semantics

RabbitMQ включает:

- broker cluster membership;
- exchanges;
- queues;
- bindings;
- messages;
- acknowledgements;
- consumers;
- quorum queues;
- streams;
- metadata;
- memory/disk alarms.

```text
RabbitMQ Pod Running
!=
messaging path healthy
```

---

# 56. RabbitMQ Cluster Operator vs Messaging Topology

RabbitMQ ecosystem разделяет:

```text
Cluster Operator
  -> broker cluster lifecycle

Messaging Topology Operator
  -> vhosts
  -> users/permissions
  -> queues
  -> exchanges
  -> bindings
```

Это важная граница ownership.

---

# 57. PVC != RabbitMQ message replication

Представим три Rabbit nodes:

```text
rabbit-0 -> PVC0
rabbit-1 -> PVC1
rabbit-2 -> PVC2
```

Это только per-node persistence.

Queue durability/replication определяется RabbitMQ queue type и product semantics.

Для quorum queue replicas используют Raft consensus.

```text
PVC
=
node-local durable storage

quorum queue
=
distributed replicated queue semantics
```

---

# 58. RabbitMQ quorum status и maintenance

RabbitMQ Cluster Operator отслеживает quorum-critical nodes, потому что shutdown critical member может лишить quorum определённые queues/streams.

Это показывает важный principle:

> Kubernetes eviction/update нельзя безопасно решать без product topology awareness.

Именно поэтому product operator ценнее generic StatefulSet.

---

# 59. Rabbit consumer scaling влияет на downstream

Spring listener:

```text
3 Pods
×
4 consumer threads
=
12 consumers
```

Если масштабировать до 20 Pods:

```text
20 × 4 = 80 consumers
```

RabbitMQ может справиться, но PostgreSQL/downstream HTTP API — нет.

Stateful capacity planning не заканчивается на broker.

---

# 60. Object storage: другой class state

Files/assets/reports часто лучше хранить через S3-compatible API.

```text
Spring Boot
    |
    | PUT/GET object
    v
S3-compatible object storage
```

Почему это удобно:

- Pods stateless;
- приложение не привязано к node volume;
- objects доступны всем replicas;
- lifecycle/retention/versioning — storage concern;
- удобно direct/presigned upload.

---

# 61. PVC vs S3

## PVC хорошо

Если application semantics требуют filesystem:

- legacy filesystem API;
- working directory;
- local database/application format;
- large temporary processing;
- tool требует mount.

## S3 хорошо

Если data — objects:

- uploads;
- attachments;
- exports;
- reports;
- media;
- generated documents.

Mental model:

```text
filesystem semantics
 -> PVC

object semantics
 -> S3
```

Это не абсолютное правило, но хороший default.

---

# 62. PostgreSQL metadata + S3 object: consistency problem

Пример file service:

```text
PostgreSQL row
  id=42
  status=READY

S3 object
  uploads/42.pdf
```

Но DB transaction и S3 PUT — не одна ACID transaction.

Failure:

```text
S3 upload succeeds
   ↓
DB transaction fails
   ↓
orphan object
```

или:

```text
DB row created
   ↓
S3 upload fails
   ↓
PENDING row
```

Нужна application workflow:

```text
PENDING
   ↓
upload
   ↓
verify checksum
   ↓
READY
```

и cleanup/retry strategy.

---

# 63. Stateful placement: replicas на одном node не дают node HA

Плохая topology:

```text
worker-1:
  postgres-0
  postgres-1
  postgres-2
```

Node dies:

```text
all replicas lost simultaneously
```

Лучше:

```text
worker-1 -> member-0
worker-2 -> member-1
worker-3 -> member-2
```

Но есть ещё storage topology.

---

# 64. Failure domains

Нужно мыслить слоями:

```text
Pod
Node
Rack
Availability Zone
Cluster
Region/Site
Storage backend
Network
Control plane
```

Реплики должны быть распределены по failure domains, которые соответствуют вашим требованиям.

Три Pods в одном failure domain могут быть “3 replicas” только на бумаге.

---

# 65. Anti-affinity и topology spread

Kubernetes может помочь распределить workloads через:

- pod anti-affinity;
- topologySpreadConstraints;
- node/zone labels.

Но product/operator может уже иметь собственные recommended scheduling policies.

Не копируйте affinity templates вслепую.

---

# 66. PDB для stateful system

PDB ограничивает voluntary evictions.

Но для quorum system:

```text
Kubernetes says:
"budget allows eviction"

product says:
"this member is quorum-critical"
```

Нужно учитывать оба слоя.

Operator часто знает product state лучше generic PDB.

---

# 67. Слишком строгий PDB тоже вреден

Например:

```text
minAvailable = all replicas
```

может блокировать:

- node drain;
- cluster upgrade;
- maintenance.

HA design должен позволять безопасную controlled disruption, а не запрещать любую disruption навсегда.

---

# 68. Stateful upgrades

Stateful product upgrade — не:

```bash
kubectl set image statefulset/database ...
```

как универсальная стратегия.

Нужно знать:

- supported upgrade path;
- version compatibility;
- protocol/storage compatibility;
- rolling order;
- quorum;
- backups;
- rollback feasibility;
- schema/catalog changes.

Operator documentation — часть runbook.

---

# 69. Scaling stateful systems тоже product-specific

Stateless Deployment:

```bash
kubectl scale deploy/orders --replicas=10
```

обычно straightforward.

Stateful product:

```text
scale brokers 3 -> 6
```

может потребовать:

- partition rebalance;
- data redistribution;
- quorum/controller considerations;
- storage provisioning;
- topology placement.

Поэтому:

```text
kubectl scale
!=
complete stateful scaling procedure
```

---

# 70. Failure lab — delete stateful Pod

Для учебного StatefulSet:

```bash
kubectl get pod,pvc
kubectl delete pod cluster-member-1
kubectl get pod -w
kubectl get pvc
```

Наблюдать:

```text
Pod disappears
   ↓
same ordinal recreated
   ↓
PVC remains
   ↓
replacement mounts associated storage
```

Это демонстрирует persistence/identity.

Не HA продукта.

---

# 71. Failure lab — PVC Pending

```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get storageclass
kubectl get events --sort-by=.lastTimestamp
```

Hypotheses:

- no/default StorageClass issue;
- unsupported access mode;
- CSI problem;
- no capacity;
- topology mismatch;
- WaitForFirstConsumer waiting for scheduling context.

Важно:

```text
Pod Pending
can be caused by storage
not application code
```

---

# 72. Failure lab — delete PostgreSQL primary

В CloudNativePG lab:

1. найти current primary;
2. зафиксировать `orders-db-rw` endpoints;
3. удалить primary Pod;
4. наблюдать operator;
5. проверить role transition;
6. проверить write Service;
7. выполнить новый DB request;
8. измерить disruption.

Нужно ответить:

```text
Who detected?
Who promoted?
How did Service change?
What happened to connections?
Was committed data lost?
Actual RTO?
```

Это настоящий failover test.

---

# 73. Failure lab — replica stale read

Flow:

1. write value через `-rw`;
2. сразу читать через `-ro`;
3. наблюдать consistency;
4. искусственно увеличить replication lag в lab, если возможно.

Цель:

> увидеть, что read replica — другой consistency contract.

---

# 74. Failure lab — disk full

Симптомы могут различаться:

## PostgreSQL

- writes fail;
- database may enter operational distress;
- checkpoints/WAL issues.

## Kafka

- broker disk pressure;
- partition availability/performance effects.

## RabbitMQ

- disk alarm;
- publishers can be flow-controlled/blocked.

Проверять:

- PVC capacity metrics;
- product-specific disk metrics;
- logs;
- expansion support;
- retention cleanup.

Restart Pod не освобождает persistent volume.

---

# 75. Failure lab — restore, а не backup

Плохой тест:

```text
Backup Job status=Complete
✓
```

Хороший тест:

```text
new isolated target
   ↓
restore backup
   ↓
apply WAL/PITR if required
   ↓
application connects
   ↓
business invariants checked
```

Вопрос:

> Какой фактический RTO мы получили?

---

# 76. Failure lab — Rabbit quorum-critical maintenance

В RabbitMQ operator lab:

- проверить quorum status;
- определить critical node;
- не завершать его blindly;
- сравнить Kubernetes maintenance intent с product state.

Цель:

```text
Pod lifecycle knowledge
+
RabbitMQ quorum knowledge
=
safe operation
```

---

# 77. Failure lab — Kafka consumer lag

Kafka Pods могут быть healthy.

Создать slow consumer.

Observe:

```text
consumer lag grows
       ↓
business event latency grows
```

Это показывает:

```text
Kubernetes workload health
!=
business processing health
```

---

# 78. Что должен фиксировать аналитик

Для каждой stateful dependency нужен system contract.

Пример PostgreSQL:

| Вопрос | Пример ответа |
|---|---|
| Source of truth | PostgreSQL |
| Write endpoint | `orders-db-rw` |
| Read replicas | allowed only for reporting |
| Read-your-write | required for transactional UI |
| Replication | operator-managed |
| Target RPO | < 1 min |
| Target RTO | < 10 min |
| Backup retention | 30 days |
| PITR | required |
| Restore test | monthly |
| Failure domains | 3 nodes / zones where available |
| Credential rotation | coordinated |
| Owner | DBA/platform |

Это гораздо полезнее требования:

> “БД должна быть отказоустойчива.”

---

# 79. Что должен понимать разработчик

Checklist:

- [ ] business state не хранится в Pod local filesystem;
- [ ] используется logical endpoint;
- [ ] application не hardcode primary Pod;
- [ ] Hikari pool рассчитан вместе с replica/HPA count;
- [ ] retries bounded;
- [ ] transient failover errors expected;
- [ ] read replica consistency understood;
- [ ] S3/DB consistency workflow explicit;
- [ ] migrations compatible with HA/rollout;
- [ ] backup/restore не реализуется application improvisation, если platform/operator owns it;
- [ ] product metrics exposed/consumed.

---

# 80. Что должен тестировать QA/SDET

Минимальная matrix:

| Scenario | Что проверяем |
|---|---|
| delete stateful Pod | identity/storage recovery |
| node loss | placement/failover |
| PVC Pending | storage diagnostics |
| disk full | product behavior |
| primary failure | failover + reconnect |
| stale replica | consistency expectations |
| wrong DB credential | auth failure |
| backup restore | recoverability |
| PITR | logical-error recovery |
| Kafka lag | business delay |
| broker loss | replication availability |
| Rabbit quorum critical | maintenance safety |
| S3 upload partial failure | orphan/PENDING cleanup |

---

# 81. Responsibility map

| Область | Developer | QA | Platform | DBA/Product Operator | Analyst |
|---|---:|---:|---:|---:|---:|
| datasource URL contract | ✓ | ✓ | shared | shared | understands |
| Hikari sizing | ✓ | load tests | shared | ✓ | capacity req |
| StorageClass/CSI | awareness | verifies | ✓ |  |  |
| PostgreSQL topology | awareness | tests | shared | ✓ | requirements |
| Kafka topology | awareness | tests | shared | ✓ | requirements |
| Rabbit topology | awareness | tests | shared | ✓ | requirements |
| backup/PITR | awareness | restore test | shared | ✓ | RPO/RTO |
| S3 lifecycle | ✓ | tests | shared | storage owner | retention req |
| failover runbook | input | verifies | ✓ | ✓ | acceptance |
| DR | awareness | drills | ✓ | ✓ | business requirement |

---

# 82. Anti-patterns

1. PostgreSQL `Deployment + emptyDir`.
2. Считать PVC backup.
3. Считать replication backup.
4. Считать StatefulSet HA.
5. Hardcode primary Pod name.
6. Читать critical transactional data через replica без consistency analysis.
7. Не тестировать reconnect после failover.
8. Backup без restore drill.
9. Backup на том же failure domain без анализа.
10. Scale Kafka/Rabbit/PostgreSQL как обычный stateless Deployment.
11. Все replicas на одном node.
12. PDB как единственная quorum protection.
13. Не мониторить disk capacity.
14. Считать `Running` broker/database достаточным health signal.
15. Увеличивать application replicas без DB/broker downstream budget.
16. Хранить uploads на local Pod filesystem.
17. Считать S3 + DB одной transaction.
18. Patch operator-owned child resources вместо CR.

---

# 83. CKAD vs production

CKAD полезно знать:

- volumes;
- PVC;
- StorageClass basics;
- StatefulSet;
- Services;
- probes;
- scheduling diagnostics.

Но production stateful operations добавляют:

```text
replication
quorum
failover
operators
backup
PITR
restore drills
RPO/RTO
topology
capacity
product upgrades
consistency
```

Поэтому CKAD — foundation, не финальная operational competence.

---

# 84. Связь с предыдущей главой

Глава 03:

```text
Pod disposable
rollout replaces Pods
```

Глава 04 добавляет:

```text
business state
must survive independently
```

Получаем:

```text
Deployment / Pods
       |
       | disposable compute
       v
Spring Boot
       |
       +--> PostgreSQL
       +--> Kafka
       +--> RabbitMQ
       +--> S3
       +--> PVC where needed
```

---

# 85. Что изучать дальше

Следующая глава:

- [05 — Day-2 failure playbook](05-day2-failure-playbook.md)

Логика перехода:

```text
Мы уже понимаем:
compute lifecycle
network path
configuration
stateful dependencies

Следующий вопрос:
когда production ломается,
как системно определить,
на каком слое failure,
и не уничтожить evidence бессмысленным restart?

        ↓

Глава 05
```

---

# 86. Production-like examples

## PostgreSQL / CloudNativePG

- [Showcase 12 — CloudNativePG primary/replicas](../../showcases/12-cloudnativepg-primary-replicas/README.md)
- [annotated.yaml](../../showcases/12-cloudnativepg-primary-replicas/annotated.yaml)
- [Spring config](../../showcases/12-cloudnativepg-primary-replicas/application.yaml)

## Kafka / Strimzi

- [Showcase 10 — Kafka producer/consumer](../../showcases/10-kafka-producer-consumer/README.md)
- [annotated.yaml](../../showcases/10-kafka-producer-consumer/annotated.yaml)

## RabbitMQ

- [Showcase 11 — RabbitMQ worker](../../showcases/11-rabbitmq-worker/README.md)
- [annotated.yaml](../../showcases/11-rabbitmq-worker/annotated.yaml)

## Object storage

- [Showcase 13 — S3-compatible object storage](../../showcases/13-object-storage-s3/README.md)
- [annotated.yaml](../../showcases/13-object-storage-s3/annotated.yaml)

## PVC

- [Showcase 05 — File worker + PVC](../../showcases/05-file-worker-pvc/README.md)
- [annotated.yaml](../../showcases/05-file-worker-pvc/annotated.yaml)

## StatefulSet primitive

- [Showcase 18 — StatefulSet + headless Service](../../showcases/18-statefulset-headless-service/README.md)
- [annotated.yaml](../../showcases/18-statefulset-headless-service/annotated.yaml)

---

# 87. Control questions

## Storage primitives

1. Почему container filesystem не durable storage?
2. Чем `emptyDir` отличается от PVC-backed volume?
3. Чем PVC отличается от PV?
4. Что делает StorageClass?
5. Что делает CSI driver?
6. Почему RWO не strict single-Pod?
7. Что даёт RWOP?
8. Что означает RWX?
9. Что делает reclaimPolicy?
10. Почему `Delete` опасно не понимать?
11. Что делает WaitForFirstConsumer?

## StatefulSet

12. Что StatefulSet даёт поверх Deployment?
13. Зачем headless Service?
14. Что такое ordinal?
15. Что делает OrderedReady?
16. Чего StatefulSet не делает?
17. Почему три StatefulSet Pods ещё не DB HA?

## Distributed semantics

18. Чем persistence отличается от replication?
19. Чем replication отличается от HA?
20. Что такое failover?
21. Что такое switchover?
22. Что такое quorum?
23. Почему 3 voting members обычно лучше 4 с точки зрения failure tolerance/cost?

## PostgreSQL

24. Почему Spring не должен знать primary Pod?
25. Что делает `-rw` Service?
26. Что делает `-ro` Service?
27. Почему read replica может быть stale?
28. Чем sync replication отличается от async?
29. Почему failover ломает existing connections?
30. Почему Hikari sizing связан с replicas?

## Backup/DR

31. Почему replication не backup?
32. Что такое PITR?
33. Почему volume snapshot не автоматически DB backup?
34. Что такое RPO?
35. Что такое RTO?
36. Чем HA отличается от DR?
37. Почему backup нужно restore-test?

## Kafka/Rabbit/S3

38. Чем Kafka partition отличается от replica?
39. Что делает Strimzi Operator?
40. Почему Kafka PVC не заменяет replication?
41. Чем Rabbit Cluster Operator отличается от topology management?
42. Почему PVC не делает Rabbit queue replicated?
43. Когда S3 лучше PVC?
44. Почему DB + S3 не одна transaction?

---

# 88. Sources

## Kubernetes

- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/storage/storage-classes/
- https://kubernetes.io/docs/tasks/administer-cluster/change-pv-access-mode-readwriteoncepod/

## PostgreSQL / CloudNativePG

- https://cloudnative-pg.io/docs/
- https://cloudnative-pg.io/docs/1.26/service_management/
- https://cloudnative-pg.io/docs/1.26/backup_recovery/

## Kafka / Strimzi

- https://strimzi.io/documentation/

## RabbitMQ

- https://www.rabbitmq.com/kubernetes/operator/operator-overview
- https://www.rabbitmq.com/kubernetes/operator/quorum-status

Актуальные facts, использованные в главе:

- StatefulSet сохраняет sticky Pod identity и применяется для stable network identity/storage/order; default pod management policy — `OrderedReady`.
- StatefulSet scale-down/delete по умолчанию не означает автоматическое удаление связанных volumes; storage lifecycle нужно проектировать отдельно.
- RWO ограничивает read-write mount одним node, но не гарантирует один Pod; RWOP — strict single-Pod access и stable с Kubernetes 1.29 для CSI.
- StorageClass default reclaimPolicy для dynamically provisioned volumes — `Delete`, а default volumeBindingMode — `Immediate`; `WaitForFirstConsumer` помогает учитывать Pod scheduling topology.
- CloudNativePG использует role-based Services `rw`, `ro` и `r`; application должна подключаться к logical Service, а не primary Pod.
- Strimzi Cluster Operator управляет lifecycle Kafka clusters, а KafkaNodePool описывает replicas, broker/controller roles, storage и resources.
- RabbitMQ Cluster Operator управляет broker cluster lifecycle, а Messaging Topology Operator — messaging objects; quorum health важен для safe maintenance.
