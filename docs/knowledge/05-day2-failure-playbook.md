# 05 — Day-2 operations and failure playbook

Проверено: 2026-09-20.

Эта глава отвечает на вопрос:

> Мы уже понимаем configuration, networking, rollout и stateful dependencies. **Что делать, когда production-система ведёт себя неправильно, а причина неизвестна?**

Цель главы — научить не командам `kubectl` как таковым, а **дисциплине расследования**:

```text
symptom
   ↓
layer
   ↓
evidence
   ↓
hypothesis
   ↓
verification
   ↓
repair
   ↓
post-check
   ↓
prevention
```

Главное правило:

> **Не лечите симптом раньше, чем поняли, на каком слое возникло первое нарушение ожидаемого поведения.**

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь:

1. Отличать symptom от root cause.
2. Не начинать расследование с restart.
3. Собирать evidence до изменения системы.
4. Определять failure layer.
5. Читать Pod status, conditions, events и restart count.
6. Понимать `Pending`.
7. Понимать `ImagePullBackOff`.
8. Понимать `CrashLoopBackOff`.
9. Понимать `Running but NotReady`.
10. Диагностировать Service без endpoints.
11. Отличать NXDOMAIN от DNS timeout.
12. Отличать connection refused от timeout.
13. Отличать TLS error, 401 и 403.
14. Диагностировать missing/wrong Secret.
15. Диагностировать OOMKilled и CPU/resource pressure.
16. Диагностировать PVC Pending и disk-full.
17. Читать stalled Deployment rollout.
18. Проверять HPA problems.
19. Отличать healthy Kafka cluster от consumer lag.
20. Проверять PostgreSQL failover.
21. Проверять RabbitMQ queue/consumer problems.
22. Понимать, когда node drain блокируется корректно.
23. Делать rollback только там, где он действительно безопасен.
24. Формулировать incident conclusion доказуемо, а не “после restart заработало”.

---

# 2. Day-1 и Day-2

## Day-1

```text
build
deploy
configure
make healthy
```

## Day-2

```text
months of operation
      |
      +--> failures
      +--> rotations
      +--> rollouts
      +--> drains
      +--> resource pressure
      +--> dependency outages
      +--> restore
      +--> upgrades
      +--> incidents
```

Production competence начинается именно на Day-2.

---

# 3. Самая важная мысль: symptom != root cause

Пример:

```text
HTTP 503
```

Это symptom.

Возможные root causes:

```text
no Ready endpoints
wrong Service selector
failed rollout
DB unavailable
application overloaded
Gateway route broken
dependency timeout
```

Другой пример:

```text
CrashLoopBackOff
```

Это тоже не root cause.

Root cause может быть:

```text
invalid Spring property
missing Secret
Liquibase error
OOM
certificate parse error
process exit
```

---

# 4. Incident loop

Используйте один и тот же цикл.

```text
1 Observe symptom
      ↓
2 Identify layer
      ↓
3 Collect evidence
      ↓
4 Form hypothesis
      ↓
5 Test hypothesis
      ↓
6 Apply minimal repair
      ↓
7 Verify recovery
      ↓
8 Verify no hidden damage
      ↓
9 Record prevention/action
```

Это лучше random YAML editing.

---

# 5. Почему restart — плохой первый шаг

```bash
kubectl delete pod <pod>
```

может:

- стереть transient state;
- уничтожить previous logs;
- изменить IP/node;
- скрыть timing problem;
- “починить” race condition;
- создать reconnect storm;
- лишить вас evidence.

Restart полезен как **осознанное remediation action**, но не как diagnostic reflex.

---

# 6. Что нужно сохранить до изменения

Минимальный evidence pack:

```bash
kubectl get deploy,rs,pod -o wide
kubectl get svc,endpointslice
kubectl get pvc
kubectl get events --sort-by=.lastTimestamp
kubectl describe pod <pod>
kubectl logs <pod> --all-containers
kubectl logs <pod> --previous
```

Дополнительно:

- deployment revision;
- image digest/tag;
- ConfigMap/Secret references;
- node;
- restartCount;
- Pod conditions;
- timestamps;
- application metrics;
- dependency metrics;
- trace/correlation IDs.

---

# 7. Общая диагностическая лестница

```text
Desired state correct?
       ↓
Object created?
       ↓
Pod scheduled?
       ↓
Image pulled?
       ↓
Container started?
       ↓
Spring started?
       ↓
startup probe passed?
       ↓
readiness passed?
       ↓
Service has ready endpoints?
       ↓
DNS works?
       ↓
TCP path works?
       ↓
TLS works?
       ↓
Authentication works?
       ↓
Authorization works?
       ↓
Dependency works?
       ↓
Business operation works?
```

Не перепрыгивайте сразу в середину без причины.

---

# 8. Layer map

```text
Layer 1  desired state / manifests
Layer 2  scheduling
Layer 3  image/container runtime
Layer 4  application startup
Layer 5  probes/readiness
Layer 6  Service/EndpointSlice
Layer 7  DNS/network
Layer 8  TLS/authentication/authorization
Layer 9  dependency
Layer 10 storage/stateful product
Layer 11 rollout/version compatibility
Layer 12 capacity/observability
```

---

# 9. Pod Pending

Symptom:

```text
STATUS=Pending
```

Первый вопрос:

> Container вообще запускался?

Часто ответ: нет.

Decision tree:

```text
Pod Pending
   |
   v
kubectl describe pod
   |
   +--> FailedScheduling?
   |       |
   |       +--> insufficient CPU/memory
   |       +--> affinity impossible
   |       +--> taint/toleration
   |       +--> topology constraint
   |
   +--> PVC Pending?
           |
           +--> StorageClass
           +--> CSI
           +--> access mode
           +--> capacity
```

Проверка:

```bash
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
kubectl get pvc
```

Spring logs здесь часто бесполезны, потому что JVM не стартовала.

---

# 10. ImagePullBackOff

Flow:

```text
Pod scheduled
   ↓
kubelet requests image
   ↓
pull fails
   ↓
retry with backoff
```

Проверить:

- image name/tag;
- registry availability;
- imagePullSecret;
- auth;
- DNS/egress to registry;
- architecture mismatch where relevant.

```bash
kubectl describe pod <pod>
```

Events обычно дают наиболее полезную причину.

---

# 11. CrashLoopBackOff

Flow:

```text
container starts
   ↓
process exits
   ↓
kubelet restarts
   ↓
process exits again
   ↓
backoff grows
```

Главное:

```text
CrashLoopBackOff
!=
root cause
```

Проверить:

```bash
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

---

# 12. Почему logs --previous критичны

Текущий container уже новый.

```text
container instance #1
   ↓ crash
logs contain root cause
   ↓
container instance #2 starts
```

Обычный `kubectl logs` может показывать уже второй process.

```bash
kubectl logs <pod> --previous
```

возвращает logs предыдущего terminated container instance, если они ещё доступны.

---

# 13. Типичные Spring Boot причины CrashLoopBackOff

```text
Spring configuration binding error
missing environment variable
Secret key missing
Liquibase/Flyway failure
port bind/startup error
invalid certificate/key
OutOfMemory / JVM abort
uncaught startup exception
```

Хорошая практика:

> Startup exception должна быть понятной и указывать конкретный configuration/dependency contract.

---

# 14. Running, но NotReady

Symptom:

```text
STATUS=Running
READY=0/1
```

Это означает:

```text
process exists
but
traffic readiness is false
```

Decision tree:

```text
Running 0/1
   |
   v
describe Pod
   |
   v
readiness probe reason
   |
   +--> wrong path/port
   +--> app not initialized
   +--> dependency policy
   +--> overloaded app
```

Проверка:

```bash
kubectl describe pod <pod>
kubectl get endpointslice
kubectl logs <pod>
```

---

# 15. Service существует, но endpoints нет

```text
Service object exists
DNS exists
but request fails
```

Проверяем:

```bash
kubectl get svc customer-api
kubectl get pod --show-labels
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api
```

Hypotheses:

```text
selector mismatch
no matching Pods
matching Pods NotReady
```

Это не DNS problem.

---

# 16. DNS: NXDOMAIN vs timeout

## NXDOMAIN

DNS server ответил:

> такого имени нет.

Проверять:

- Service name;
- namespace;
- short-name scope;
- Service existence.

## Timeout

DNS server не ответил вовремя.

Проверять:

- NetworkPolicy egress;
- DNS Service;
- cluster DNS Pods;
- CNI/network path.

Это принципиально разные симптомы.

---

# 17. UnknownHostException

Spring symptom:

```text
java.net.UnknownHostException: customer-api
```

Проверка:

```bash
kubectl exec <debug-pod> -- cat /etc/resolv.conf
kubectl exec <debug-pod> -- nslookup customer-api
kubectl get svc customer-api
```

Если default-deny egress включён — проверить DNS allow.

---

# 18. Connection refused

Conceptually:

```text
DNS succeeded
network reached destination
but
nothing accepted connection on target socket
```

Проверять:

- targetPort;
- container listening port;
- wrong protocol/port;
- process not listening.

Это отличается от timeout.

---

# 19. Connect timeout

```text
connection attempt
   ↓
no usable response within timeout
```

Possible layers:

- NetworkPolicy;
- routing;
- remote dependency unavailable;
- firewall;
- overloaded dependency;
- path blackhole.

Проверять network path прежде, чем менять JWT.

---

# 20. TLS handshake error

Если TCP есть, но TLS ломается:

Проверять:

- certificate expired;
- trust chain;
- hostname/SAN;
- wrong protocol HTTP vs HTTPS;
- mTLS client cert;
- clock/time;
- rotated cert not reloaded.

```text
TCP success
TLS failure
```

уже сильно сужает investigation.

---

# 21. 401 vs 403

## 401

```text
authentication failed
```

Примеры:

- token missing;
- token invalid;
- expired;
- issuer/signature mismatch.

## 403

```text
authentication succeeded
authorization denied
```

Примеры:

- authority missing;
- scope missing;
- policy denies action.

Не диагностируйте 401 через Service selector.

---

# 22. Wrong Secret

Симптом зависит от application lifecycle.

## Startup dependency

```text
wrong DB password
   ↓
Spring startup fails
   ↓
CrashLoopBackOff
```

## Lazy/reconnect dependency

```text
Pod Ready
   ↓
later DB reconnect
   ↓
authentication failures
```

Проверять:

- Secret object exists;
- expected key exists;
- Deployment reference;
- Pod revision;
- env-based Secret requires new Pod;
- credential valid on dependency side.

Не печатать secret value.

---

# 23. Secret rotated, но Pod использует старый env

```text
Secret=P2
running Pod environment=P1
```

Это ожидаемо для env-based injection.

Проверка:

- когда Pod создан;
- после какого Secret revision;
- был ли rollout.

Repair:

```text
coordinated rollout
```

а не “Secret не работает”.

---

# 24. Expired certificate

Symptoms:

- PKIX validation failed;
- certificate expired;
- hostname mismatch;
- unknown CA;
- client certificate rejected.

Incident questions:

```text
Which side owns cert?
Who issued it?
When did it expire?
Was Secret updated?
Did application reload it?
```

Restart со старым certificate ничего не исправляет.

---

# 25. OOMKilled

Pod status/history может показать:

```text
Reason=OOMKilled
```

Mental model:

```text
container memory usage
      >
cgroup memory limit
      ↓
container killed
```

Для JVM считать нужно не только heap:

```text
heap
+ metaspace
+ threads
+ direct memory
+ code cache
+ native
```

Проверять:

- container limit;
- JVM heap sizing;
- memory metrics;
- thread count;
- direct buffers;
- leak/load pattern.

---

# 26. CPU throttling / saturation

Pod может быть:

```text
Running
Ready
not restarting
```

но latency растёт.

Possible cause:

```text
CPU demand > available CPU / quota behavior
```

Проверять:

- CPU usage;
- request/limit;
- throttling metrics where available;
- HPA;
- downstream latency;
- GC.

Not every production incident has a red Pod status.

---

# 27. PVC Pending

Decision tree:

```text
PVC Pending
   |
   +--> StorageClass exists?
   |
   +--> default class expected?
   |
   +--> CSI provisioner healthy?
   |
   +--> access mode supported?
   |
   +--> capacity available?
   |
   +--> WaitForFirstConsumer?
   |
   +--> topology mismatch?
```

Commands:

```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get storageclass
kubectl get events --sort-by=.lastTimestamp
```

---

# 28. Disk full

Distinguish:

```text
PVC full
vs
node filesystem full
```

PVC full can cause:

- PostgreSQL write failures;
- RabbitMQ disk alarm;
- Kafka broker/storage distress;
- file worker failures.

Node full can affect:

- kubelet;
- image/runtime storage;
- logs;
- node pressure.

Restart не создаёт free space.

---

# 29. Deployment rollout stalled

Decision tree:

```text
Deployment not progressing
      |
      v
new ReplicaSet exists?
   /       \
 no         yes
 |           |
template?    v
         new Pod created?
            /      \
           no      yes
           |        |
        quota/      v
        scale    scheduled?
                 /     \
               no      yes
               |        |
           resources/    v
           PVC        started?
                       /   \
                     no    yes
                     |      |
                   image/   v
                   security Ready?
                           /   \
                         no    yes
                         |      |
                      probes/  minReady/
                      app      deadline
```

Commands:

```bash
kubectl rollout status deploy/orders
kubectl describe deploy orders
kubectl get rs
kubectl get pod -o wide
kubectl get events --sort-by=.lastTimestamp
```

---

# 30. Technically successful rollout, functionally broken

Очень важный class.

```text
all Pods Ready
rollout Complete
HTTP 200
```

но:

- wrong price calculation;
- DB schema logic bug;
- events malformed;
- latency doubled;
- Kafka lag grows;
- error rate in downstream rises.

Поэтому rollout verification включает:

```text
Kubernetes health
+
technical SLI
+
business KPI
```

---

# 31. HPA “не масштабируется”

Проверять:

```text
HPA object exists?
metrics available?
resource requests present?
current utilization?
min/max bounds?
target reachable?
```

Commands:

```bash
kubectl get hpa
kubectl describe hpa <name>
kubectl top pod
kubectl get deploy
```

Частая conceptual ошибка:

```text
CPU utilization target
depends on CPU requests
```

Если requests отсутствуют/неадекватны — HPA semantics могут быть не теми, что ожидали.

---

# 32. HPA масштабирует, но система хуже

```text
load rises
   ↓
HPA adds Pods
   ↓
more Hikari pools
   ↓
DB connections spike
   ↓
PostgreSQL saturates
   ↓
latency/errors rise
```

Root cause:

> scaling app exceeded downstream capacity.

Не всегда правильный repair — “ещё больше maxReplicas”.

---

# 33. PostgreSQL failover incident

Сценарий:

```text
primary dies
```

Проверять:

1. operator cluster status;
2. current primary;
3. `-rw` Service endpoints;
4. replica health;
5. failover duration;
6. application reconnect;
7. transaction errors;
8. actual RPO/RTO.

Application symptom может быть transient:

```text
broken connections
SQL exceptions
pool reconnect
```

Не hardcode primary Pod.

---

# 34. PostgreSQL auth failure vs network failure

```text
DNS works
TCP 5432 works
server responds
login rejected
```

Это credential/auth issue.

Если:

```text
connect timeout
```

это ещё не password problem.

Layering экономит время расследования.

---

# 35. PostgreSQL pool exhaustion

Spring/Hikari symptom:

```text
connection acquisition timeout
```

Possible causes:

- queries too slow;
- leaked/long transactions;
- pool too small;
- DB saturated;
- too many app replicas;
- network stall.

Проверять:

```text
Hikari active
idle
pending
acquisition time
DB active connections
query latency
locks
```

---

# 36. Kafka: cluster healthy, business unhealthy

```text
Kafka brokers Ready
topic exists
producer works
```

но:

```text
consumer lag growing
```

Root hypotheses:

- consumer too slow;
- downstream DB slow;
- partitions too few;
- consumer errors/retries;
- rebalance churn;
- poison message;
- processing blocked.

Consumer lag — business processing signal, а не Pod lifecycle status.

---

# 37. Kafka consumer group imbalance

Если topic:

```text
6 partitions
```

а consumer group:

```text
10 Pods
```

часть consumers idle.

Если load high, adding Pods beyond partition count не увеличит useful parallelism.

Incident repair может требовать:

```text
partitioning/capacity redesign
```

а не ещё replicas.

---

# 38. RabbitMQ queue depth grows

Possible chain:

```text
producer rate > consumer rate
      ↓
queue depth grows
      ↓
latency grows
```

Проверять:

- ready messages;
- unacked;
- consumer count;
- consumer processing time;
- downstream DB/API;
- broker memory/disk alarms.

Не всегда broker — bottleneck.

---

# 39. Rabbit consumer stuck / unacked

Большой `unacked` может означать:

- worker получил message и долго обрабатывает;
- downstream завис;
- transaction hung;
- ack semantics wrong.

Сравнивать:

```text
ready
unacked
consumer count
processing latency
```

---

# 40. Node NotReady

Pod problems могут быть следствием node failure.

```bash
kubectl get node
kubectl describe node <node>
kubectl get pod -o wide
```

Проверять:

- Ready condition;
- pressure conditions;
- kubelet connectivity;
- disk/memory;
- network;
- affected Pods.

Не анализируйте каждый Pod как независимый application incident, если все они на одном broken node.

---

# 41. Node drain

Before:

```bash
kubectl get pod -o wide
kubectl get pdb
```

Then:

```bash
kubectl cordon <node>
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Но перед drain проверить:

- PDB;
- stateful operator health;
- quorum;
- local storage;
- target node capacity;
- topology.

---

# 42. Drain blocked by PDB

Scenario:

```text
Deployment replicas=1
PDB minAvailable=1
```

Drain tries eviction:

```text
eviction would violate PDB
      ↓
drain waits/fails
```

Это не “Kubernetes завис”.

Availability policy и maintenance intent конфликтуют.

Repair — изменить system design/maintenance plan, а не force-delete blindly.

---

# 43. Stateful maintenance и quorum

Даже если generic PDB разрешает eviction, product может считать node quorum-critical.

```text
Kubernetes budget OK
       |
       X
Rabbit/Kafka/Postgres topology unsafe
```

Поэтому stateful maintenance требует:

```text
Kubernetes state
+
product state
```

---

# 44. Rollback

```bash
kubectl rollout undo deployment/orders
```

Это rollback Pod template.

Не rollback:

- DB migration;
- Kafka event already published;
- Rabbit message processed;
- S3 object written;
- revoked credential;
- external API side effect.

До rollback спросите:

> Совместима ли старая application version с текущим persistent state?

---

# 45. Когда rollback опасен

Например:

```text
v2 migration removes column
      ↓
v2 deployed
      ↓
rollback to v1
      ↓
v1 expects removed column
      ↓
rollback makes incident worse
```

Иногда правильная strategy:

```text
roll forward
```

а не rollback.

---

# 46. Secret rotation incident

Runbook:

```text
new credential issued
      ↓
Secret updated
      ↓
workload rollout/reload
      ↓
new connections verified
      ↓
old credential revoked
```

Incident if order reversed:

```text
old credential revoked
      ↓
existing Pods still have old env
      ↓
reconnects fail
```

Проверять Pod age/revision и actual dependency authentication.

---

# 47. Certificate rotation incident

```text
Secret updated
   ↓
mounted file may update
   ↓
application may still use old certificate
```

Questions:

- file changed?
- process reloads automatically?
- TLS context rebuilt?
- rollout required?
- old cert expired already?

---

# 48. Observability: RED

Для HTTP:

```text
Rate
Errors
Duration
```

Если user reports “service slow”, первым графиком часто полезнее:

```text
latency p95/p99
+
error rate
+
request rate
```

чем CPU.

---

# 49. JVM signals

Минимум:

- heap usage;
- non-heap;
- GC pauses;
- thread count;
- CPU;
- process/container RSS;
- allocation rate where available.

Пример:

```text
latency spike
   ↓
GC pause spike
   ↓
memory pressure
```

или:

```text
thread pool saturated
   ↓
latency grows
while CPU moderate
```

---

# 50. Dependency signals

## PostgreSQL

- active connections;
- locks;
- query latency;
- errors;
- replica lag;
- failover state.

## Kafka

- consumer lag;
- producer errors;
- ISR;
- broker/storage status.

## RabbitMQ

- queue depth;
- unacked;
- consumers;
- memory/disk alarms.

## S3/Object storage

- request latency;
- errors;
- object count/storage;
- failed multipart uploads where relevant.

---

# 51. Kubernetes signals

Useful:

- restart count;
- OOMKilled;
- Pending;
- probe failures;
- unavailable replicas;
- node pressure;
- PVC state;
- rollout conditions;
- HPA conditions.

Но Kubernetes metrics не заменяют application metrics.

---

# 52. Logs: что должно быть полезно для incident

Хорошие fields:

```text
timestamp
service
version
pod
traceId/correlationId
request path
safe business identifier
dependency
error class
latency
```

Не логировать:

- password;
- access token;
- Authorization header;
- private key;
- raw Secret;
- sensitive customer payload.

---

# 53. Trace/correlation thinking

Один user request может пройти:

```text
Gateway
  ↓
orders-api
  ↓
payment-api
  ↓
PostgreSQL
  ↓
Kafka
```

Без correlation:

```text
5 systems
5 unrelated logs
```

С trace/correlation ID:

```text
one causal chain
```

Это резко улучшает incident diagnosis.

---

# 54. Evidence quality levels

Слабое:

> После restart заработало.

Среднее:

> Pod был NotReady.

Сильное:

> Readiness endpoint возвращал 503 с 14:03:11 до 14:07:42; EndpointSlice помечал все three endpoints not-ready; root cause — exhausted DB pool из-за long transactions. После cleanup active pool normalized, readiness recovered.

Цель incident culture — переходить к третьему уровню.

---

# 55. Hypothesis-driven debugging

Плохой процесс:

```text
может DNS?
поменяем policy
может Java?
рестарт
может Service?
поменяем selector
```

Хороший:

```text
Evidence:
DNS resolves
EndpointSlice has ready endpoint
TCP connect timeout

Hypothesis:
egress/ingress policy blocks path

Test:
inspect NetworkPolicy + controlled connection test
```

---

# 56. Minimal repair principle

Repair должен менять минимально необходимую часть.

Если root cause:

```text
wrong Service selector
```

repair:

```text
fix selector
```

не:

```text
restart cluster
delete Deployment
recreate namespace
```

Чем крупнее remediation, тем больше новых variables вы добавляете.

---

# 57. Verify after repair

“Command succeeded” не равно recovery.

После fix проверить:

```text
object state correct?
Pod Ready?
EndpointSlice correct?
request works?
error rate normal?
latency normal?
dependency normal?
business KPI normal?
```

Для stateful:

```text
data intact?
replication healthy?
lag normal?
backup/recovery state unaffected?
```

---

# 58. Incident timeline

Хороший incident record:

```text
13:58 deployment v1.4.3 started
14:01 first elevated 5xx
14:03 all new Pods NotReady
14:05 root cause narrowed to DB auth
14:08 discovered Secret v2 but Pods env v1
14:10 controlled rollout
14:13 error rate recovered
14:20 old credential revoked
```

Timeline помогает отличить cause от coincidence.

---

# 59. Change correlation

Всегда спросите:

```text
Что изменилось перед incident?
```

Возможные changes:

- application release;
- ConfigMap;
- Secret rotation;
- NetworkPolicy;
- certificate;
- node maintenance;
- DB migration;
- operator upgrade;
- HPA change;
- StorageClass;
- external dependency release.

Это не доказательство root cause, но сильный investigation hint.

---

# 60. Failure matrix

| Symptom | Первый слой | Evidence | Частые причины |
|---|---|---|---|
| Pending | scheduling/storage | events/describe | resources/PVC/affinity |
| ImagePullBackOff | image/registry | events | tag/auth/network |
| CrashLoopBackOff | process | previous logs | config/startup/OOM |
| Running 0/1 | readiness | describe/endpoints | probe/app/dependency |
| no endpoints | Service/readiness | labels/EndpointSlice | selector/NotReady |
| NXDOMAIN | DNS naming | nslookup/Service | name/namespace |
| DNS timeout | DNS reachability | resolver/policy | NetworkPolicy/CNI |
| connection refused | socket/backend | port/listener | targetPort/process |
| connect timeout | network/path | policy/connect test | deny/routing/load |
| TLS error | PKI | cert details | trust/expiry/SAN |
| 401 | authn | app logs/token claims | token invalid |
| 403 | authz | authorities/policy | permission missing |
| OOMKilled | memory | status/metrics | limit/leak/load |
| rollout stalled | rollout | deploy/RS/Pod/events | image/probes/capacity |
| PVC Pending | storage | PVC/events/SC | CSI/access/capacity |
| DB auth fail | dependency auth | logs/DB | Secret/rotation |
| DB pool timeout | DB capacity | Hikari/DB metrics | slow/leak/saturation |
| Kafka lag | processing | consumer metrics | slow consumer/partitions |
| Rabbit queue growth | messaging | queue/consumer metrics | consumers/downstream |
| HPA no scale | autoscaling | HPA conditions/metrics | metrics/requests/bounds |

---

# 61. Для аналитика

Аналитик во время incident помогает не командами kubectl, а system context.

Нужно уметь ответить:

- какой user flow нарушен;
- какой business impact;
- какие dependencies участвуют;
- какой expected fallback;
- какие RPO/RTO/SLO;
- какие recent changes;
- кто owner каждого слоя;
- можно ли rollback;
- какие irreversible side effects уже произошли.

Хорошее incident requirement:

```text
Payment creation unavailable.
Read-only order search remains available.
No committed payment data loss acceptable.
RTO < 15 min.
```

Это помогает технической команде выбирать recovery.

---

# 62. Для разработчика

Developer responsibility:

- meaningful startup errors;
- probes с правильной semantics;
- finite timeouts;
- bounded retries;
- structured logs;
- metrics;
- correlation IDs;
- graceful shutdown;
- dependency-specific error classification;
- no secret leakage;
- correct exit behavior;
- rollback/schema compatibility awareness.

Диагностируемость — часть application quality.

---

# 63. Для тестировщика

Tester/SDET должен уметь intentionally inject failures.

Минимальная practice matrix:

```text
wrong image
missing ConfigMap
wrong Secret
broken readiness
broken liveness
selector mismatch
wrong targetPort
DNS deny
network deny
invalid JWT
missing authority
OOM pressure
PVC Pending
disk full
failed rollout
DB primary loss
Kafka lag
Rabbit consumer slowdown
```

Для каждого заранее записать:

```text
Expected symptom
Expected evidence
Expected recovery
```

---

# 64. Для Platform/SRE

Platform responsibility:

- cluster/node visibility;
- DNS/CNI/CSI health;
- scheduling capacity;
- registry connectivity;
- certificates/platform secrets;
- operator health;
- storage health;
- metrics/logging stack;
- drain/upgrade procedures;
- escalation/runbooks.

Platform не должна автоматически обвинять application, если root cause — CNI/CSI/node.

---

# 65. Incident severity thinking

Severity должна отражать impact, не “страшность” stack trace.

Пример:

```text
one canary Pod CrashLoop
no user impact
!=
all payment writes failing
```

Полезные axes:

- users affected;
- criticality;
- data integrity;
- duration;
- workaround;
- regulatory/business impact.

---

# 66. Когда нужно остановить изменения

Во время incident опасно одновременно менять:

```text
Deployment
ConfigMap
NetworkPolicy
DB password
HPA
```

Тогда невозможно понять, что реально помогло.

Если ситуация позволяет:

> one hypothesis → one controlled change → observe result.

---

# 67. Roll forward vs rollback

## Rollback

Хорошо, если:

- previous version compatible;
- no irreversible schema/data change;
- problem clearly tied to new release.

## Roll forward

Лучше, если:

- rollback incompatible with current schema;
- data already transformed;
- external side effects occurred;
- fixed version can be deployed safely faster.

Incident runbook должен знать оба пути.

---

# 68. Post-incident questions

После recovery:

1. Почему monitoring обнаружил/не обнаружил incident?
2. Почему system allowed failure to impact users?
3. Почему recovery занял столько времени?
4. Какой evidence отсутствовал?
5. Что можно автоматизировать?
6. Нужен ли new alert?
7. Нужен ли test?
8. Нужен ли architecture change?
9. Нужно ли обновить runbook?
10. Был ли фактический RPO/RTO в пределах requirement?

---

# 69. Практический incident drill: readiness failure

## Step 1

Deploy healthy application.

## Step 2

Сломать readiness path.

## Step 3

Не рестартовать.

Собрать:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice
kubectl logs <pod>
```

## Step 4

Сформулировать root cause:

```text
readiness endpoint returns 404
therefore Pod Running but Ready=False
therefore EndpointSlice endpoint not ready
therefore Service has no normal ready backend
```

## Step 5

Fix path.

## Step 6

Verify:

- Ready;
- endpoints;
- request;
- error rate.

---

# 70. Практический incident drill: DNS deny

## Break

Apply default deny egress without DNS allow.

## Evidence

```bash
kubectl exec <orders-pod> -- nslookup customer-api
kubectl get networkpolicy
kubectl get svc customer-api
```

Hypothesis:

```text
Service exists
but DNS resolver unreachable
```

Fix explicit DNS egress based on actual cluster DNS labels/service.

---

# 71. Практический incident drill: bad DB password

## Break

Rotate Secret incorrectly.

## Observe

```text
DNS works
TCP works
DB rejects authentication
```

## Evidence

- app logs without printing password;
- Pod revision;
- Secret reference;
- DB auth logs where available.

## Recovery

Correct credential + controlled rollout.

---

# 72. Практический incident drill: stalled rollout

Break new image or readiness.

Then:

```bash
kubectl rollout status deploy/orders
kubectl describe deploy orders
kubectl get rs
kubectl get pod
kubectl describe pod <new-pod>
kubectl logs <new-pod>
```

Goal:

> identify exact layer where progress stops.

---

# 73. Практический incident drill: PostgreSQL failover

Using operator-managed lab:

```text
record primary
record -rw Service endpoint
delete/fail primary
observe promotion
observe application errors
verify reconnect
measure RTO
```

Do not stop at:

> new primary exists.

Verify business write.

---

# 74. Практический incident drill: Kafka lag

Generate messages faster than consumer processes.

Observe:

```text
broker healthy
consumer Pods healthy
lag grows
```

Then determine bottleneck:

- partitions;
- consumer CPU;
- DB;
- external API;
- retry loop.

---

# 75. Практический incident drill: disk full

Fill lab PVC or simulate threshold.

Observe:

- product-specific write errors;
- storage metrics;
- application symptoms.

Recovery:

- cleanup/retention;
- expansion where supported;
- capacity change.

Not:

```text
restart until it works
```

---

# 76. Связанные failure-стенды

Используйте:

- [Internal REST service](../../showcases/01-internal-rest-service/README.md) — selector, targetPort, readiness, Secret;
- [REST + PostgreSQL + HPA + NetworkPolicy](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md) — DNS, egress, HPA, DB pool;
- [Service-to-service security](../../showcases/06-service-to-service-security/README.md) — DNS → TCP → JWT → 401/403;
- [Blue/Green](../../showcases/08-blue-green/README.md) — cutover/rollback;
- [Canary](../../showcases/09-canary/README.md) — version-specific failures;
- [Kafka](../../showcases/10-kafka-producer-consumer/README.md) — consumer lag;
- [RabbitMQ](../../showcases/11-rabbitmq-worker/README.md) — queue/consumer;
- [CloudNativePG](../../showcases/12-cloudnativepg-primary-replicas/README.md) — primary failover;
- [StatefulSet + headless](../../showcases/18-statefulset-headless-service/README.md) — identity/PVC.

Practice loop:

```text
predict symptom
   ↓
break
   ↓
collect evidence
   ↓
form hypothesis
   ↓
verify
   ↓
repair
   ↓
verify recovery
```

---

# 77. Control questions

## Method

1. Чем symptom отличается от root cause?
2. Почему restart плохой первый diagnostic action?
3. Что собрать до изменения?
4. Что такое hypothesis-driven debugging?
5. Почему минимальный repair лучше большого?

## Pods

6. Что означает Pending?
7. Что означает ImagePullBackOff?
8. Почему CrashLoopBackOff не root cause?
9. Когда использовать logs --previous?
10. Что означает Running 0/1?

## Network

11. Что означает NXDOMAIN?
12. Чем DNS timeout отличается от NXDOMAIN?
13. Чем connection refused отличается от timeout?
14. Что проверять при TLS error?
15. Чем 401 отличается от 403?

## Configuration/resources

16. Почему Secret update не меняет env running Pod?
17. Что означает OOMKilled?
18. Почему JVM RSS не равен heap?
19. Может ли CPU incident происходить без Pod restart?

## Storage/stateful

20. Что проверить при PVC Pending?
21. Почему restart не лечит full PVC?
22. Как проверить PostgreSQL failover?
23. Что такое pool exhaustion?
24. Почему consumer lag важнее broker Running?
25. Почему Rabbit queue depth может расти при healthy brokers?

## Rollout/autoscaling

26. Как диагностировать stalled rollout?
27. Почему technically successful rollout может быть functionally broken?
28. Почему HPA может не scale?
29. Почему successful scale-up может ухудшить DB?
30. Когда rollback опасен?

## Incident practice

31. Что должно быть в incident timeline?
32. Почему change correlation полезен, но не является доказательством?
33. Что проверить после repair?
34. Что такое roll forward?
35. Какие вопросы задавать после incident?

---

# 78. Что изучать дальше

Следующая глава:

- [06 — CKAD map](06-ckad-map.md)

Логика перехода:

```text
Глава 05:
мы научились рассуждать о failures системно

Следующий вопрос:
как превратить эти mental models
в быстрые практические действия
с Kubernetes primitives
и подготовиться к CKAD?

       ↓

Глава 06
```

---

# 79. Sources

- https://kubernetes.io/docs/tasks/debug/debug-application/
- https://kubernetes.io/docs/tasks/debug/debug-cluster/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/workloads/pods/disruptions/

Главный operational принцип главы:

```text
Do not ask first:
"what command should I run?"

Ask:
"what layer is violating the expected contract,
and what evidence proves it?"
```
