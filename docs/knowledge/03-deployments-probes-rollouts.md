# 03 — Deployment, ReplicaSet, probes, rollout и graceful termination

Проверено: 2026-09-20.

Эта глава отвечает на следующий вопрос после Service/DNS:

> Service умеет направлять traffic только в usable backends. **Кто создаёт эти Pods, как Kubernetes заменяет версию v1 на v2, когда новый Pod становится пригодным для traffic и что происходит, если rollout ломается?**

Главная mental model:

```text
Deployment
   |
   | desired application revision
   v
ReplicaSet
   |
   | desired replica count
   v
Pods
   |
   +--> startup
   +--> readiness
   +--> liveness
   |
   v
Ready endpoints
   |
   v
Service traffic
```

А rollout — это не просто “поменяли image”:

```text
old revision
   |
   | coexistence rules
   v
new revision
   |
   +--> startup
   +--> readiness
   +--> availability
   +--> old Pods termination
```

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь объяснить:

1. Чем Deployment отличается от Pod.
2. Зачем Deployment создаёт ReplicaSet.
3. Что такое Pod template.
4. Какие изменения создают новую rollout revision.
5. Почему изменение `replicas` само по себе не создаёт новую revision.
6. Чем `RollingUpdate` отличается от `Recreate`.
7. Что реально означают `maxUnavailable` и `maxSurge`.
8. Почему rollout требует свободной cluster capacity.
9. Чем Running, Ready и Available отличаются друг от друга.
10. Зачем `startupProbe`.
11. Зачем `livenessProbe`.
12. Зачем `readinessProbe`.
13. Почему внешняя БД не должна определять liveness.
14. Как Spring Boot Actuator связан с Kubernetes probes.
15. Что делает `minReadySeconds`.
16. Что делает `progressDeadlineSeconds`.
17. Что происходит при SIGTERM.
18. Как Kubernetes termination grace period связан со Spring graceful shutdown.
19. Почему application rollback не откатывает DB migration.
20. Как сделать rollout совместимым с Liquibase/Flyway.
21. Что такое PDB и чего он **не** гарантирует.
22. Как диагностировать stalled rollout.
23. Что должен проверять аналитик, разработчик и тестировщик.

---

# 2. Одна схема всего жизненного цикла

Начнём с общей картины.

```text
CI changes image:
orders:1.4.2 -> orders:1.4.3
           |
           v
Deployment.spec.template changes
           |
           v
Deployment controller
           |
           +--> old ReplicaSet v1
           |
           +--> new ReplicaSet v2
                    |
                    v
                 new Pod
                    |
                    v
              container starts
                    |
                    v
               JVM starts
                    |
                    v
              Spring context
                    |
                    v
              startupProbe
                    |
                    v
              readinessProbe
                    |
                    v
               Ready=True
                    |
                    v
          EndpointSlice ready=true
                    |
                    v
             Service traffic
                    |
                    v
         old Pod can be reduced
                    |
                    v
                 SIGTERM
                    |
                    v
        Spring graceful shutdown
                    |
                    v
                Pod exits
```

Это и есть rollout как **runtime process**, а не только YAML.

---

# 3. Как было раньше: systemd deployment

На VM rollout часто выглядел:

```text
Load Balancer
     |
     v
VM
 |
 +--> systemctl stop orders
 +--> cp orders.jar
 +--> systemctl start orders
```

Или при нескольких VM:

```text
remove VM1 from LB
deploy VM1
return VM1

remove VM2 from LB
deploy VM2
return VM2
```

Kubernetes автоматизирует многие части этого процесса, но не отменяет fundamental вопросы:

- когда новая версия действительно готова;
- можно ли одновременно держать v1 и v2;
- хватает ли capacity;
- совместима ли DB schema;
- как завершить старые requests;
- что делать при failed revision.

---

# 4. Deployment не запускает Java напрямую

Очень важное разделение:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pod
    |
    v
container runtime
    |
    v
java
    |
    v
Spring Boot
```

## Deployment

Хранит desired application state:

```text
replicas
Pod template
rollout strategy
availability/progress rules
```

## ReplicaSet

Поддерживает определённое количество Pods одной Pod template revision.

## kubelet

Уже на конкретном node запускает containers и выполняет probes.

---

# 5. Почему вообще нужен ReplicaSet

Можно представить Deployment как manager revisions.

Например:

```text
Deployment orders
   |
   +--> ReplicaSet hash=aaa
   |      image=orders:1.4.2
   |      desired=3
   |
   +--> ReplicaSet hash=bbb
          image=orders:1.4.3
          desired=0
```

Во время rollout:

```text
RS aaa
desired 3 -> 2 -> 1 -> 0

RS bbb
desired 0 -> 1 -> 2 -> 3
```

Deployment orchestrates relationship между ReplicaSets.

---

# 6. Pod template — граница revision

Ключевая часть Deployment:

```yaml
spec:
  template:
    metadata:
      labels:
        app: orders

    spec:
      containers:
        - name: app
          image: registry.example/orders:1.4.2
```

Изменение `spec.template` создаёт новую rollout revision.

Например:

- image;
- env;
- probes;
- resources;
- labels внутри template;
- volumes;
- ServiceAccount;
- container command.

Conceptually:

```text
template hash changed
      ↓
new ReplicaSet
```

---

# 7. Что НЕ создаёт новую revision

Изменение:

```yaml
spec:
  replicas: 3
```

на:

```yaml
spec:
  replicas: 6
```

масштабирует current revision.

Это не новый application revision.

То есть:

```text
replicas change
 -> scale

Pod template change
 -> rollout revision
```

Это важно для понимания HPA: HPA меняет replica count, но не выпускает новую версию приложения.

---

# 8. Минимальный Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders

spec:
  replicas: 3

  selector:
    matchLabels:
      app: orders

  template:
    metadata:
      labels:
        app: orders

    spec:
      containers:
        - name: app
          image: registry.example/orders:1.4.2
```

Главный mapping:

```text
Deployment.selector
      =
Pod template labels
```

Если они не совпадают, manifest некорректен для intended ownership.

---

# 9. Почему selector нельзя считать обычным editable field

Deployment selector задаёт identity набора Pods, которым Deployment управляет.

После creation его нельзя свободно менять как обычную настройку application.

Mental model:

```text
selector
=
ownership boundary
```

Поэтому labels проектируют заранее.

---

# 10. Что происходит после kubectl apply Deployment

```text
kubectl apply
      |
      v
API Server stores desired state
      |
      v
Deployment controller sees Deployment
      |
      v
creates/updates ReplicaSet
      |
      v
ReplicaSet creates Pod objects
      |
      v
Scheduler assigns nodes
      |
      v
kubelet starts containers
      |
      v
probes report state
```

Если удалить Pod вручную:

```bash
kubectl delete pod <orders-pod>
```

ReplicaSet видит:

```text
desired=3
actual=2
```

и создаёт replacement.

---

# 11. RollingUpdate — default deployment strategy

Если strategy не указана, Deployment использует `RollingUpdate`.

```yaml
strategy:
  type: RollingUpdate
```

Default rolling settings:

```text
maxUnavailable = 25%
maxSurge       = 25%
```

Эти defaults полезно знать, но в critical production workload лучше задавать values осознанно.

---

# 12. maxUnavailable: сколько capacity можно потерять

Пример:

```yaml
replicas: 4

strategy:
  rollingUpdate:
    maxUnavailable: 1
```

Conceptually:

```text
desired replicas = 4

во время rollout
минимально допустимо available ~= 3
```

Для percentage Kubernetes округляет percentage вниз при расчёте maxUnavailable.

---

# 13. maxSurge: сколько extra capacity разрешено

```yaml
replicas: 4

strategy:
  rollingUpdate:
    maxSurge: 1
```

Во время rollout может временно существовать:

```text
desired=4
running up to ~5
```

Для percentage surge округляется вверх.

---

# 14. Почему maxUnavailable=0, maxSurge=1 популярен для API

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Для 3 replicas:

```text
BEFORE

v1 A READY
v1 B READY
v1 C READY
```

Создаётся v2:

```text
v1 A READY
v1 B READY
v1 C READY
v2 A STARTING
```

После readiness:

```text
v1 A READY
v1 B READY
v1 C READY
v2 A READY
```

Теперь один v1 можно убрать.

### Но есть цена

Node capacity должна позволять extra Pod.

Если cluster заполнен:

```text
maxSurge=1
      ↓
new Pod Pending
      ↓
rollout cannot progress
```

Zero-downtime YAML без spare capacity не даёт zero downtime.

---

# 15. Recreate

```yaml
strategy:
  type: Recreate
```

Mental model:

```text
v1 v1 v1
 |  |  |
 X  X  X

no old Pods

then

v2 v2 v2
```

Это создаёт downtime window.

Recreate полезен, если две versions действительно не могут coexist.

Но если причиной несовместимости является destructive DB migration, часто лучше исправить migration model, чем просто спрятать проблему за Recreate.

---

# 16. Rollout — это период coexistence versions

При RollingUpdate одновременно существуют:

```text
v1 Pod
v1 Pod
v2 Pod
```

Следовательно, v1 и v2 должны одновременно понимать shared dependencies.

Особенно:

- database schema;
- Kafka event schema;
- Rabbit message format;
- Redis/cache values;
- downstream API contracts.

Это один из ключевых production lessons.

---

# 17. Running, Ready и Available

Три термина нельзя смешивать.

## Running

Container process запущен.

```text
JVM exists
```

## Ready

Pod сейчас пригоден для normal traffic.

```text
readiness conditions satisfied
```

## Available для Deployment

Pod считается available после того, как он был Ready не менее `minReadySeconds`.

Поэтому:

```text
Running
   !=
Ready
   !=
Deployment Available
```

---

# 18. Почему process может быть Running, но application не Ready

```text
java process started
      ↓
Spring context starting
      ↓
configuration binding
      ↓
Liquibase/Flyway
      ↓
HTTP server startup
      ↓
cache initialization
      ↓
application ready
```

Во время части этого flow process существует, но traffic ещё может быть unsafe.

---

# 19. Три probes отвечают на три разных вопроса

```text
startupProbe
  "Приложение уже закончило startup?"

livenessProbe
  "Этот process безнадёжно сломан
   и его нужно restart?"

readinessProbe
  "Можно ли прямо сейчас отправлять
   сюда новый traffic?"
```

Не делайте один endpoint автоматически смыслом всех трёх probes.

---

# 20. startupProbe

Пример:

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http

  periodSeconds: 5
  failureThreshold: 30
```

Примерное startup budget:

```text
5 seconds
×
30 failed probes
=
~150 seconds
```

Пока startup probe не добилась успеха, kubelet не запускает обычные liveness/readiness probe cycles для container.

Это защищает slow-starting JVM от преждевременного liveness restart.

---

# 21. Почему startupProbe лучше огромного initialDelaySeconds

Представьте:

```yaml
livenessProbe:
  initialDelaySeconds: 120
```

Fast startup:

```text
app ready in 8s
but liveness waits 120s
```

Startup probe:

```text
probe every 5s
   ↓
app ready at 10s
   ↓
startup succeeds
   ↓
normal probe lifecycle begins immediately
```

Это более adaptive model.

---

# 22. livenessProbe

Liveness означает:

> restart этого container может помочь?

Хорошие liveness failures:

- deadlock;
- corrupted internal process state;
- event loop cannot progress;
- application AvailabilityState broken.

Плохая liveness dependency:

```text
PostgreSQL unavailable
```

Почему?

```text
DB outage
   ↓
all app Pods report liveness DOWN
   ↓
kubelet restarts all JVMs
   ↓
all Hikari pools reconnect
   ↓
DB receives reconnect storm
   ↓
incident becomes worse
```

Spring Boot docs отдельно рекомендуют не включать external systems в liveness group именно из-за cascading failures.

---

# 23. readinessProbe

Readiness означает:

> Отправлять ли новые requests этому Pod?

```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: http
```

Failure:

```text
Pod process remains Running
       ↓
Ready=False
       ↓
EndpointSlice marks endpoint not ready
       ↓
normal Service traffic stops using it
```

Readiness может быть временным состоянием.

---

# 24. Должна ли readiness зависеть от DB?

Здесь нет одного универсального ответа.

## Вариант A — strict dependency

Если application **вообще не может отвечать полезно** без PostgreSQL:

```text
DB unavailable
  ↓
readiness=false
```

Это может быть разумно.

## Но возможна проблема

Если все replicas зависят от одной DB:

```text
DB down
   ↓
all Pods Ready=False
   ↓
Service has no ready endpoints
```

Client получает hard outage.

Иногда лучше application itself возвращает controlled degraded errors, circuit breaker или read-only mode.

Поэтому readiness — **business availability decision**, а не checkbox.

---

# 25. Spring Boot Actuator probes

Spring Boot Actuator exposes dedicated availability groups:

```text
/actuator/health/liveness
/actuator/health/readiness
```

В Kubernetes environment они интегрируются с Spring `ApplicationAvailability`.

Spring Boot также поддерживает additional probe paths на main server port.

Например:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true
```

Это даёт удобные paths вроде `/livez` и `/readyz` на основном server port.

---

# 26. Почему main server port часто лучше отдельного management port для probes

Представим:

```text
management port = 9090 healthy

main application connector = 8080 broken
```

Если kubelet проверяет только 9090:

```text
probe healthy
but real user path broken
```

Поэтому Spring Boot docs рекомендуют осознанно выбирать probe port; additional paths на main port помогают проверять тот же HTTP connector, через который приходит traffic.

---

# 27. Полный probe example

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 30

livenessProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /readyz
    port: http
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
```

Это **пример**, не universal numbers.

Timings должны основываться на measured:

- JVM startup;
- GC pauses;
- expected endpoint latency;
- dependency behavior;
- rollout SLO.

---

# 28. Probe defaults, которые полезно знать

Если не указано иное:

```text
initialDelaySeconds = 0
periodSeconds       = 10
timeoutSeconds      = 1
failureThreshold    = 3
successThreshold    = 1
```

Но хороший production manifest обычно делает critical timing assumptions явными.

---

# 29. Resources входят в rollout semantics

Deployment может быть идеальным логически, но new Pod не schedule.

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
```

Rollout:

```text
new revision created
   ↓
new Pod requested
   ↓
scheduler searches capacity
   ↓
no node can fit request
   ↓
Pod Pending
   ↓
rollout stalls
```

Поэтому resource requests — часть release engineering.

---

# 30. JVM memory и rollout

Представим:

```text
old Pods memory request = 1Gi
replicas = 10

maxSurge = 2
```

Во время rollout cluster должен временно вместить ещё:

```text
~2Gi requested memory
+
sidecars/init overhead
```

Если capacity нет — rollout зависнет.

---

# 31. minReadySeconds

Default:

```text
minReadySeconds = 0
```

То есть Pod становится available сразу после Ready.

Если:

```yaml
minReadySeconds: 10
```

то Pod должен оставаться Ready 10 секунд без container crash, прежде чем Deployment считает его Available.

Это полезно против:

```text
Pod Ready
  ↓ 2 seconds
crash
```

---

# 32. progressDeadlineSeconds

Default:

```text
progressDeadlineSeconds = 600
```

Он должен быть больше `minReadySeconds`.

Если rollout слишком долго не progresses:

```text
Deployment condition
Progressing=False
reason=ProgressDeadlineExceeded
```

Важно:

> Deployment controller сообщает stalled progress, но не делает автоматический rollback сам по себе.

External deployment automation может использовать эту condition для решения rollback.

---

# 33. Почему rollout может не progress

Типичные причины:

```text
bad image
      -> ImagePullBackOff

bad application config
      -> CrashLoopBackOff

readiness never succeeds
      -> Pods Running but unavailable

insufficient node capacity
      -> Pending

PVC cannot bind
      -> Pending

security/context error
      -> container cannot start
```

То есть “Deployment stuck” — симптом, а не root cause.

---

# 34. Как читать rollout status

```bash
kubectl rollout status deploy/orders
kubectl describe deploy orders
kubectl get rs
kubectl get pod -o wide
```

Mental sequence:

```text
Deployment condition
      ↓
ReplicaSet states
      ↓
new Pod states
      ↓
events
      ↓
container logs
```

Не начинайте с random restart.

---

# 35. Revision history

Deployment сохраняет old ReplicaSets according to `revisionHistoryLimit`.

Default:

```text
revisionHistoryLimit = 10
```

Это позволяет:

```bash
kubectl rollout history deploy/orders
kubectl rollout undo deploy/orders
```

Но history — не Git history и не DB rollback mechanism.

---

# 36. kubectl set image

```bash
kubectl set image deploy/orders \
  app=registry.example/orders:1.4.3
```

Что реально изменилось:

```text
Deployment.spec.template.spec.containers[].image
       ↓
Pod template hash changes
       ↓
new ReplicaSet
       ↓
rollout starts
```

То есть `kubectl set image` — удобная API mutation, а не special runtime deployment engine.

---

# 37. Rollback Deployment

```bash
kubectl rollout undo deploy/orders
```

Это возвращает previous Deployment Pod template revision.

Например:

```text
image 1.4.3
   ↓ rollback
image 1.4.2
```

Но не возвращает автоматически:

- database rows;
- Liquibase changes;
- Kafka events;
- Rabbit messages;
- object storage files;
- external API side effects.

---

# 38. Главная проблема rollback: database schema

Плохой deployment:

```text
Step 1:
DROP COLUMN old_name

Step 2:
rollout v2
```

Пока v1 Pods ещё существуют:

```text
v1 query
SELECT old_name
   ↓
SQL failure
```

Kubernetes rollout может быть идеально healthy.
Application compatibility — сломана.

---

# 39. Expand → migrate → contract

Безопаснее:

## Expand

Добавить новую structure, не ломая old clients.

```text
old_name
new_name
```

## Migrate

v1 и v2 coexist; data постепенно migrated.

## Contract

После того как v1 больше нигде нет:

```text
remove old_name
```

Это позволяет RollingUpdate/Canary/Blue-Green работать с shared DB.

---

# 40. Liquibase/Flyway не решают compatibility автоматически

Migration tool умеет:

```text
version migrations
execute DDL/DML
track applied changes
```

Но он не знает:

> Можно ли этот DROP COLUMN выполнить, пока v1 Pod жив?

Это responsibility application/release design.

---

# 41. Graceful termination: что происходит со старым Pod

Когда Deployment scale-down удаляет old Pod:

```text
Pod termination begins
       ↓
endpoint/readiness transition
       ↓
preStop if configured
       ↓
SIGTERM
       ↓
Spring shutdown
       ↓
in-flight requests drain
       ↓
process exits
       |
       +--> if grace expires: forced termination
```

Точная network propagation может быть distributed/asynchronous, поэтому application должна быть tolerant к short overlap.

---

# 42. terminationGracePeriodSeconds

Default Pod grace period обычно:

```text
30 seconds
```

Example:

```yaml
spec:
  terminationGracePeriodSeconds: 30
```

Spring:

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

Хорошая relationship:

```text
Spring graceful budget
<
Kubernetes hard termination budget
```

---

# 43. Spring Boot graceful shutdown

В актуальном Spring Boot graceful shutdown enabled by default для поддерживаемых embedded web servers.

На shutdown Spring прекращает принимать новые requests и даёт in-flight requests время закончиться в рамках lifecycle timeout.

Но нужно отдельно проверить:

- HTTP requests;
- Kafka listeners;
- RabbitMQ consumers;
- scheduled tasks;
- custom executors;
- DB transactions.

Web graceful shutdown не магически завершает любой custom background workflow.

---

# 44. Нужен ли preStop sleep

Частый pattern:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

Иногда его используют, чтобы дать external load balancers/proxies время увидеть endpoint removal.

Но blindly копировать `sleep 10` — плохая практика.

Нужно понимать:

- какой traffic path;
- кто обновляет endpoints;
- как LB/controller drains;
- сколько занимает propagation;
- какой total termination budget.

`preStop` consumes часть termination grace period.

---

# 45. PDB: где он находится

PodDisruptionBudget:

```text
Deployment
   |
   +--> Pods
          |
          +--> voluntary disruption
                   |
                   v
                  PDB
```

Он ограничивает voluntary evictions.

Например:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orders
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: orders
```

---

# 46. Что PDB защищает

Например node drain:

```bash
kubectl drain worker-2
```

Eviction API учитывает PDB.

Mental model:

```text
admin wants to evict Pods
       ↓
PDB checks disruption budget
       ↓
eviction may wait
```

---

# 47. Что PDB НЕ защищает

PDB не гарантирует availability при:

- node crash;
- power loss;
- OOMKilled;
- application crash;
- NetworkPolicy mistake;
- failed dependency;
- bad rollout;
- storage failure.

То есть:

```text
PDB
!=
HA guarantee
```

---

# 48. Blue/Green и Canary находятся над базовой Deployment mechanics

RollingUpdate:

```text
Deployment itself manages overlap
```

Blue/Green:

```text
Deployment blue
Deployment green
      |
      v
explicit traffic switch
```

Canary:

```text
stable Deployment
canary Deployment
      |
      v
weighted traffic
```

Поэтому понимание обычного Deployment — prerequisite для advanced strategies.

Смотрите:

- [Showcase 08 — Blue/Green](../../showcases/08-blue-green/README.md)
- [Showcase 09 — Canary](../../showcases/09-canary/README.md)

---

# 49. Production-oriented Deployment example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api

spec:
  replicas: 3

  strategy:
    type: RollingUpdate

    rollingUpdate:
      # Не уменьшаем normal available capacity намеренно.
      maxUnavailable: 0

      # Нужна capacity ещё для одного Pod.
      maxSurge: 1

  # Pod должен быть стабильно Ready до Available.
  minReadySeconds: 10

  # Rollout, который не progresses, должен быть видимым.
  progressDeadlineSeconds: 600

  # Старые ReplicaSets нужны для rollout history.
  revisionHistoryLimit: 10

  selector:
    matchLabels:
      app: orders-api

  template:
    metadata:
      labels:
        app: orders-api

    spec:
      automountServiceAccountToken: false

      # Hard outer shutdown budget.
      terminationGracePeriodSeconds: 30

      containers:
        - name: app
          image: registry.example/orders-api:1.4.3

          ports:
            - name: http
              containerPort: 8080

          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              memory: 768Mi

          startupProbe:
            httpGet:
              path: /livez
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 30

          livenessProbe:
            httpGet:
              path: /livez
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
```

Этот manifest — reference, не universal copy/paste.

---

# 50. Mapping таблица

| Откуда | Куда | Зачем |
|---|---|---|
| Deployment selector | Pod labels | ownership ReplicaSet/Pods |
| Service selector | Pod labels | network backend selection |
| Probe `port: http` | container named port | health request |
| Deployment template image | ReplicaSet revision | rollout identity |
| resources.requests | scheduler | placement/capacity |
| readiness | EndpointSlice | traffic eligibility |
| termination grace | kubelet/container | shutdown deadline |

Один Pod label может одновременно участвовать и в Deployment ownership, и в Service selection — но это разные mechanisms.

---

# 51. Failure lab — bad image

Изменить:

```bash
kubectl set image deploy/orders-api \
  app=registry.example/orders-api:does-not-exist
```

Observe:

```bash
kubectl rollout status deploy/orders-api
kubectl get rs
kubectl get pod
kubectl describe pod <new-pod>
```

Expected:

```text
new ReplicaSet exists
       ↓
new Pods ImagePullBackOff
       ↓
new Pods never Ready
       ↓
rollout stalls
```

При conservative RollingUpdate old Ready Pods могут продолжать traffic.

---

# 52. Failure lab — bad startupProbe

Set:

```yaml
startupProbe:
  httpGet:
    path: /never-exists
    port: http
```

Expected:

```text
container starts
   ↓
startup probe repeatedly fails
   ↓
kubelet kills container after threshold
   ↓
restart
```

Symptoms:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

---

# 53. Failure lab — bad readinessProbe

```yaml
readinessProbe:
  httpGet:
    path: /wrong
    port: http
```

Expected:

```text
container Running
       ↓
Ready=False
       ↓
Deployment new revision not Available
       ↓
Service does not use new Pod as normal ready backend
       ↓
rollout may stall
```

Это идеальная демонстрация:

```text
process health
!=
traffic readiness
```

---

# 54. Failure lab — bad livenessProbe

```yaml
livenessProbe:
  httpGet:
    path: /wrong
    port: http
```

Expected:

```text
container starts
   ↓
liveness fails repeatedly
   ↓
kubelet restarts container
   ↓
restartCount grows
```

Use:

```bash
kubectl logs <pod> --previous
```

чтобы не потерять logs предыдущего crashed/restarted container.

---

# 55. Failure lab — no surge capacity

Создайте lab condition, где node capacity недостаточна для extra Pod.

Expected:

```text
old Pods healthy
new Pod Pending
rollout cannot progress
```

Check:

```bash
kubectl describe pod <pending-pod>
kubectl get events --sort-by=.lastTimestamp
```

Это показывает, что rollout strategy связана с cluster capacity.

---

# 56. Failure lab — schema incompatibility

Lab concept:

v1:

```sql
SELECT old_name FROM customer;
```

Migration:

```sql
ALTER TABLE customer DROP COLUMN old_name;
```

v2 rollout begins.

Expected:

```text
v1 Pods still receive traffic
       ↓
SQL fails
```

Kubernetes Pods могут быть Ready — но business flow сломан.

Это важный тест для QA:

> технический readiness != version compatibility.

---

# 57. Failure lab — graceful termination

Создайте endpoint:

```text
GET /slow
takes ~10 seconds
```

Во время request:

```bash
kubectl delete pod <pod>
```

Проверить:

- завершился ли response;
- когда endpoint исчез из Service;
- сколько занял shutdown;
- были ли connection reset;
- что видно в logs.

Это намного полезнее, чем просто проверить, что Pod “когда-нибудь удалился”.

---

# 58. Диагностическая лестница stalled rollout

```text
Deployment not progressing
       |
       v
kubectl describe deployment
       |
       v
which ReplicaSet is new?
       |
       v
new Pod created?
   /          \
 no            yes
 |              |
scheduler/      v
quota/PVC   container started?
             /       \
            no        yes
            |          |
        image/security v
                    Ready?
                   /    \
                 no      yes
                 |        |
              probes/   minReady/
              app       progress
```

Эта схема важнее списка команд.

---

# 59. Что должен фиксировать аналитик

Release requirement “без downtime” недостаточен.

Нужно уточнить:

| Вопрос | Пример |
|---|---|
| Сколько replicas? | 3 |
| Допустим unavailable? | 0 |
| Есть surge capacity? | +1 Pod |
| Max startup time? | 120s |
| Когда считаем ready? | business-ready endpoint |
| Можно ли v1/v2 coexist? | yes |
| DB schema compatible? | expand-contract |
| Rollback allowed? | yes, within 15 min |
| Есть irreversible side effects? | migration/event changes |
| Shutdown max duration? | 20s |
| SLO during rollout? | error rate < 0.1% |

Аналитик тем самым превращает “zero downtime” в проверяемый system contract.

---

# 60. Что должен сделать разработчик

Checklist:

- [ ] immutable image version;
- [ ] startup/liveness/readiness имеют разные semantics;
- [ ] external dependency не включена blindly в liveness;
- [ ] startup time измерен;
- [ ] graceful shutdown работает;
- [ ] HTTP/background workers завершаются корректно;
- [ ] resources measured;
- [ ] DB migration mixed-version compatible;
- [ ] logs идут в stdout/stderr;
- [ ] readiness отражает реальную способность обслуживать traffic;
- [ ] old/new versions совместимы с shared state.

---

# 61. Что должен тестировать QA/SDET

Минимальная matrix:

| Scenario | Expected |
|---|---|
| delete one Pod | replacement created |
| new image valid | gradual rollout |
| image missing | ImagePullBackOff, rollout stalls |
| startup slow | startupProbe protects |
| startup never succeeds | restart |
| readiness fails | Running, NotReady, no normal traffic |
| liveness fails | container restart |
| no surge capacity | new Pod Pending |
| DB migration breaks v1 | business failure during coexistence |
| rollback application | old Pod template returns |
| SIGTERM during request | graceful behavior verified |
| node drain + PDB | voluntary eviction constrained |

---

# 62. Developer vs Platform vs QA

| Область | Developer | QA | Platform/SRE | Shared |
|---|---:|---:|---:|---:|
| probe semantics | ✓ | ✓ |  | ✓ |
| probe timings | ✓ | ✓ | ✓ | ✓ |
| resource sizing | ✓ | tests | ✓ | ✓ |
| node capacity |  |  | ✓ |  |
| rollout strategy | ✓ | ✓ | ✓ | ✓ |
| graceful shutdown | ✓ | ✓ |  | ✓ |
| DB compatibility | ✓ | ✓ | DBA | ✓ |
| PDB | awareness | ✓ | ✓ | ✓ |
| rollout automation | input | verifies | ✓ | ✓ |
| rollback decision | input | evidence | ✓ | ✓ |

---

# 63. Anti-patterns

1. Один endpoint для startup/liveness/readiness без смыслового анализа.
2. DB/Kafka availability в liveness.
3. `maxUnavailable=0` без surge capacity.
4. `maxSurge` без учёта quotas/resources.
5. `-Xmx` равный memory limit.
6. Destructive DB migration до окончания old revision.
7. Rollback application как единственный rollback plan.
8. Blind `preStop sleep 30`.
9. PDB как “гарантия HA”.
10. Restart Pods до чтения events/logs.
11. `latest` как release identity.
12. Проверять только `kubectl get pod Running` и считать deployment healthy.

---

# 64. CKAD mapping

Нужно уверенно использовать:

```bash
kubectl create deployment
kubectl scale deployment
kubectl set image
kubectl rollout status
kubectl rollout history
kubectl rollout undo
kubectl describe deployment
kubectl get rs
kubectl get pod
kubectl describe pod
kubectl logs
kubectl logs --previous
```

Для CKAD важно быстро реализовать объект.

Для production нужно дополнительно понимать:

```text
capacity
JVM
probe semantics
DB compatibility
shutdown
SLO
PDB
rollback boundaries
```

---

# 65. Связь с предыдущей главой

В главе 02 мы получили:

```text
Service
  ↓
EndpointSlice
  ↓
Ready Pods
```

Теперь стало понятно:

```text
Deployment
  ↓
ReplicaSet
  ↓
Pods
  ↓
readiness
  ↓
EndpointSlice
  ↓
Service
```

То есть workload lifecycle и service routing — одна система.

---

# 66. Что изучать дальше

Следующая глава:

- [04 — Stateful dependencies](04-stateful-dependencies.md)

Логика перехода:

```text
Глава 03:
мы научились заменять disposable application Pods

Следующий вопрос:
если Pod disposable,
где тогда живут данные?
как Kubernetes работает с PostgreSQL,
Kafka, RabbitMQ, PVC и object storage?
кто отвечает за replication/failover/backup?

       ↓

Глава 04
```

---

# 67. Production-like examples

## Baseline Deployment

- [Showcase 01 — Internal REST](../../showcases/01-internal-rest-service/README.md)
- [annotated.yaml](../../showcases/01-internal-rest-service/annotated.yaml)

## Blue/Green

- [Showcase 08](../../showcases/08-blue-green/README.md)
- [annotated.yaml](../../showcases/08-blue-green/annotated.yaml)

## Canary

- [Showcase 09](../../showcases/09-canary/README.md)
- [annotated.yaml](../../showcases/09-canary/annotated.yaml)

## HPA + deployment capacity

- [Showcase 03](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)

---

# 68. Control questions

## Deployment

1. Что именно делает Deployment?
2. Зачем нужен ReplicaSet?
3. Что такое Pod template?
4. Что создаёт новую revision?
5. Создаёт ли replica count новую revision?
6. Кто создаёт replacement после delete Pod?
7. Почему selector — ownership boundary?

## Rollout

8. Какая strategy default?
9. Что означает maxUnavailable?
10. Что означает maxSurge?
11. Почему surge требует capacity?
12. Чем Recreate отличается от RollingUpdate?
13. Почему v1/v2 compatibility обязательна?

## Availability

14. Чем Running отличается от Ready?
15. Чем Ready отличается от Deployment Available?
16. Что делает minReadySeconds?
17. Что делает progressDeadlineSeconds?
18. Что означает ProgressDeadlineExceeded?

## Probes

19. Что делает startupProbe?
20. Почему она лучше большого fixed startup delay?
21. Что делает livenessProbe?
22. Почему DB нельзя blindly включать в liveness?
23. Что делает readinessProbe?
24. Может ли DB входить в readiness?
25. Почему это architectural decision?

## Shutdown

26. Что получает process при termination?
27. Что такое terminationGracePeriodSeconds?
28. Как связан Spring shutdown timeout?
29. Что делает preStop?
30. Почему blind sleep плох?

## Data compatibility

31. Почему kubectl rollout undo не откатывает DB?
32. Что такое expand-migrate-contract?
33. Почему Liquibase не решает compatibility автоматически?

## Operations

34. Что проверить при stalled rollout?
35. Почему новый Pod может быть Pending?
36. Когда использовать logs --previous?
37. Что реально защищает PDB?
38. От чего PDB не защищает?

---

# 69. Sources

- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/
- https://kubernetes.io/docs/concepts/workloads/pods/probes/
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
- https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
- https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html

Актуальные facts, использованные в главе:

- `RollingUpdate` — default Deployment strategy.
- `maxUnavailable` default = 25%; percentage округляется вниз.
- `maxSurge` default = 25%; percentage округляется вверх.
- `minReadySeconds` default = 0.
- `progressDeadlineSeconds` default = 600 и должен быть больше `minReadySeconds`.
- startup probe suppresses liveness/readiness until startup success.
- readiness failure removes a Pod from normal Service traffic eligibility without requiring container restart.
- Spring Boot liveness/readiness groups опираются на ApplicationAvailability; external systems не должны blindly входить в liveness.
- Graceful shutdown enabled by default в актуальном Spring Boot для поддерживаемых embedded web servers.
