# 03 — Deployments, probes, resources and rollouts

## Учебная карта темы

### Deployment в общей системе

```text
Deployment
    |
    v
ReplicaSet
    |
    +--> Pod v1
    +--> Pod v1
    +--> Pod v1

spec.template changes
    |
    v
new ReplicaSet
    |
    +--> Pod v2
```

Deployment отвечает за **desired replica state и rollout**, а readiness решает, когда новый Pod можно считать usable.

### Rollout + probes

```text
new Pod starts
   ↓
startupProbe passes
   ↓
liveness begins to matter
   ↓
readiness passes
   ↓
Pod enters Service endpoints
   ↓
old Pod can be scaled down
```

### Annotated fragment

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # не теряем available replicas
    maxSurge: 1         # разрешаем один временный extra Pod
```

> **Production:** rollout приложения не равен rollback database schema. Для mixed-version rollout нужна backward-compatible schema.

Смотрите [Blue/Green](../../showcases/08-blue-green/README.md) и [Canary](../../showcases/09-canary/README.md).

## Для аналитика, разработчика и тестировщика

### Аналитик

Должен понимать, что “обновление сервиса” — это период coexistence старой и новой версии. Требования к zero downtime должны учитывать:
- capacity;
- readiness;
- DB/message compatibility;
- rollback boundaries.

### Разработчик

Настраивает:
- startup/liveness/readiness по разным смыслам;
- graceful lifecycle;
- resources;
- rollout-safe schema changes;
- immutable image revisions.

### Тестировщик

Проверяет:
- rollout без потери available replicas;
- bad image;
- readiness failure новой версии;
- rollback;
- slow startup;
- несовместимость schema/version.

### Перед следующей главой

Вы должны объяснить:

```text
change spec.template
 -> new ReplicaSet
 -> new Pods
 -> readiness
 -> traffic
 -> old Pods removed
```

Проверено: 2026-09-20.

Этот раздел объясняет не только **что написать в Deployment**, но и **какой процесс запускает каждое поле, почему оно существует и что в этот момент происходит со Spring Boot приложением**.

## 1. Mental model: Deployment не запускает Java напрямую

На VM привычная модель часто выглядела так:

```text
systemd -> java -jar app.jar
```

При обновлении мы могли остановить процесс, заменить JAR и запустить его снова.

В Kubernetes цепочка другая:

```text
Deployment
  -> ReplicaSet
      -> Pod
          -> container
              -> java -jar app.jar
```

Deployment хранит **desired state**: например, «нужно 3 реплики с image v2». ReplicaSet обеспечивает количество Pods. Kubelet на конкретном node запускает containers и выполняет probes.

**Почему так:** Kubernetes построен вокруг reconciliation. Вы не отдаёте команду «запусти ещё один Java-процесс», а описываете желаемое состояние. Controllers постоянно сравнивают желаемое и фактическое состояние и устраняют расхождение.

### Практический пример

```bash
kubectl get deploy
kubectl get rs
kubectl get pod

kubectl delete pod <orders-pod>
kubectl get pod -w
```

Удалённый Pod появится снова с другим именем, потому что ReplicaSet всё ещё должен поддерживать заданное количество replicas.

> **Практика:** Pod нужно считать расходной единицей. IP Pod, имя Pod и локальную filesystem нельзя использовать как постоянную identity приложения.

---

## 2. Почему Pod не «обновляется на месте»

Когда меняется ``.spec.template`` Deployment, Kubernetes не заходит в существующий Pod и не заменяет JAR. Создаётся новая revision и новый ReplicaSet.

```text
ReplicaSet v1 -> Pod v1 Pod v1 Pod v1
ReplicaSet v2 -> Pod v2 ...
```

Старые Pods постепенно удаляются, новые создаются.

Это даёт immutable deployment model и объясняет, почему application state нельзя держать в container filesystem.

---

## 3. RollingUpdate и Recreate

### Recreate

```yaml
strategy:
  type: Recreate
```

Логика:

```text
v1 v1 v1
 X  X  X

период без приложения

v2 v2 v2
```

Recreate допустим, если две версии приложения вообще не могут существовать одновременно. Для обычного HTTP Spring Boot backend это обычно означает ненужный downtime.

### RollingUpdate

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

По умолчанию Kubernetes использует 25% для ``maxUnavailable`` и 25% для ``maxSurge``.

- ``maxUnavailable`` — сколько desired replicas разрешено потерять во время rollout.
- ``maxSurge`` — сколько временных дополнительных Pods разрешено создать.

### Практический пример: 3 replicas, 0 unavailable, surge 1

Исходно:

```text
v1-A READY
v1-B READY
v1-C READY
```

Kubernetes создаёт дополнительный v2:

```text
v1-A READY
v1-B READY
v1-C READY
v2-A STARTING
```

Когда v2-A становится Ready:

```text
v1-A READY
v1-B READY
v1-C READY
v2-A READY
```

Теперь один старый Pod можно убрать. Процесс повторяется до трёх v2 Pods.

**Почему это удобно:** обычный rollout не должен намеренно опускать число available replicas ниже трёх.

**Цена:** cluster должен иметь capacity для surge Pod. Поэтому нулевой downtime — это не только YAML, но и capacity planning.

---

## 4. Running не означает Ready

Pod может быть:

```text
phase = Running
container = Running
readiness = false
```

JVM работает, но Service не должен посылать туда обычный traffic.

Это важная ментальная модель:

```text
process alive != application ready to serve
```

Например, Spring Boot уже открыл port 8080, но ещё:
- не прогрел cache;
- не завершил initialization;
- не получил обязательную configuration;
- временно не способен безопасно обслуживать запросы.

---

## 5. Startup, liveness и readiness — три разных вопроса

### startupProbe

Вопрос:

> Приложение уже завершило старт или ему ещё нужно время?

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 5
  failureThreshold: 30
```

Окно здесь примерно 150 секунд.

**Почему лучше, чем огромный initialDelay:** startupProbe завершает startup phase сразу после успеха. Быстрое приложение не обязано ждать фиксированные 120 секунд, а медленному можно дать достаточно времени.

### livenessProbe

Вопрос:

> Сам процесс ещё способен продолжать работу?

Если liveness многократно fails, kubelet перезапускает container.

**Нельзя делать PostgreSQL/Kafka/external API частью liveness.**

Иначе возможен каскад:

```text
DB outage
 -> liveness fails у всех Pods
 -> все JVM restart
 -> все connection pools одновременно reconnect
 -> нагрузка на DB возрастает
 -> outage становится тяжелее
```

Liveness должна описывать состояние самого process.

### readinessProbe

Вопрос:

> Можно ли прямо сейчас направить сюда новый traffic?

Readiness failure обычно не перезапускает container. Pod остаётся жив, но перестаёт считаться ready endpoint.

```text
readiness=true  -> Service routes traffic
readiness=false -> Service не должен использовать Pod как ready backend
```

---

## 6. Spring Boot Actuator

Spring Boot Actuator предоставляет:

```text
/actuator/health/liveness
/actuator/health/readiness
```

Удобно добавить probe paths на основной application port:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true
```

Тогда можно использовать ``/livez`` и ``/readyz``.

**Почему основной port полезен:** отдельный management port теоретически может быть healthy, когда основной HTTP connector уже не обслуживает запросы.

---

## 7. Production probe example

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

| Поле | Что делает | Риск плохой настройки |
|---|---|---|
| ``periodSeconds`` | период проверки | слишком часто = шум и лишняя нагрузка |
| ``timeoutSeconds`` | сколько ждать response | слишком мало = false failures |
| ``failureThreshold`` | сколько failures подряд терпеть | слишком мало = flapping/restarts |
| ``initialDelaySeconds`` | задержка первой проверки | для startup часто выразительнее startupProbe |

> **Практика:** значения probe timings должны опираться на измеренный startup/latency, а не копироваться из случайного Helm chart.

---

## 8. Resources: requests и limits

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    memory: 768Mi
```

### Requests

Scheduler использует requests для размещения Pod.

```text
Node allocatable CPU = 2 cores
Pod A requests = 1
Pod B requests = 1
Pod C requests = 1

Pod C может остаться Pending
```

Даже если A и B в данный момент реально используют намного меньше CPU.

### Memory limit

При выходе container за cgroup memory limit возможен OOM kill.

Для Java:

```text
container memory =
  heap
+ metaspace
+ thread stacks
+ direct buffers
+ native libraries
+ JVM native memory
```

Поэтому практика ``-Xmx = весь memory limit`` опасна: JVM нужен native headroom.

### CPU limit

CPU limit обычно вызывает throttling, а не kill. Для latency-sensitive Java сервиса чрезмерно жёсткий CPU limit способен создавать latency spikes.

> **Практика:** requests/limits выбираются по measurements, SLO и cluster policy, а не по шаблону.

---

## 9. Graceful shutdown

При termination упрощённо:

```text
Kubernetes начинает завершение Pod
 -> process получает SIGTERM
 -> идёт terminationGracePeriodSeconds
 -> если process не завершился, возможен forced kill
```

Spring Boot graceful shutdown в современных версиях включён по умолчанию для поддерживаемых embedded web servers. Timeout phase можно настроить:

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

Kubernetes:

```yaml
terminationGracePeriodSeconds: 30
```

**Почему 30 > 20:** Spring получает время корректно завершить работу, а Kubernetes остаётся operational запас до hard deadline.

### Практический пример

Есть HTTP request длительностью 8 секунд.

Без graceful shutdown:

```text
Pod killed -> connection reset -> client error
```

С корректным shutdown текущий request получает шанс завершиться.

Но отдельно нужно тестировать:
- HTTP;
- Kafka listener;
- Rabbit listener;
- scheduled tasks;
- thread pools;
- batch jobs.

---

## 10. Production Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 3

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  minReadySeconds: 10
  progressDeadlineSeconds: 600

  selector:
    matchLabels:
      app: orders

  template:
    metadata:
      labels:
        app: orders
    spec:
      terminationGracePeriodSeconds: 30

      containers:
        - name: app
          image: registry.example/orders:1.4.2

          ports:
            - name: http
              containerPort: 8080

          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              memory: 768Mi

          startupProbe:
            httpGet:
              path: /livez
              port: http
            periodSeconds: 5
            failureThreshold: 30

          livenessProbe:
            httpGet:
              path: /livez
              port: http
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            failureThreshold: 2
```

---

## 11. minReadySeconds и progressDeadlineSeconds

``minReadySeconds: 10`` означает: Pod должен оставаться Ready некоторое время, прежде чем Deployment считает его available.

Это защищает от ситуации:

```text
startup -> Ready -> через 2 секунды crash
```

``progressDeadlineSeconds`` задаёт предел ожидания progress rollout. Broken image, failing readiness, missing config или scheduling failure могут привести к condition вроде ``ProgressDeadlineExceeded``.

---

## 12. Что делает kubectl set image

```bash
kubectl set image deployment/orders \
  app=registry.example/orders:1.4.3

kubectl get rs
kubectl rollout status deploy/orders
kubectl rollout history deploy/orders
```

Команда изменяет Pod template Deployment. Новый template означает новую revision и новый ReplicaSet.

---

## 13. Rollback и его границы

```bash
kubectl rollout undo deployment/orders
```

Rollback возвращает предыдущий Deployment revision, но **не откатывает автоматически**:
- Liquibase/Flyway;
- Kafka event schema;
- Rabbit messages;
- data changes;
- external configuration side effects.

### Production практика: expand → migrate → contract

```text
EXPAND   -> добавить совместимую структуру
MIGRATE  -> обе версии приложения переживают переход
CONTRACT -> удалить legacy структуру позже
```

Плохой rollout:

```sql
ALTER TABLE customer DROP COLUMN old_name;
```

пока старые v1 Pods ещё выполняют:

```sql
SELECT old_name FROM customer;
```

Kubernetes в таком случае работает корректно; ломается application/database compatibility.

---

## 14. PDB: что он реально делает

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

PDB помогает при **voluntary disruptions**, например eviction во время:

```bash
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data
```

Он не предотвращает внезапный node failure, application crash, OOM или network partition.

---

## 15. Как было раньше и как сейчас

### VM

```text
Load Balancer
  -> VM -> systemd -> java -jar
```

Deploy часто был imperative:

```bash
systemctl stop app
cp app.jar
systemctl start app
```

### Kubernetes

```text
Deployment -> ReplicaSet -> Pods
                          -> kubelet probes

Service -> Ready endpoints
```

То есть health, rollout и availability становятся частью declarative application contract.

---

## 16. Failure practice: broken readiness

Намеренно укажите:

```yaml
readinessProbe:
  httpGet:
    path: /does-not-exist
    port: http
```

Диагностика:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice
```

Ожидаем:

```text
STATUS: Running
READY: 0/1
```

Главный вывод: **Running != Ready**.

---

## 17. Failure practice: broken liveness

Сломайте liveness path:

```bash
kubectl get pod -w
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

Увидите рост restart count.

Проверяйте:
1. deadlock приложения;
2. неправильный path;
3. слишком короткий timeout;
4. dependency ошибочно включена в liveness;
5. CPU starvation.

---

## 18. Failure practice: broken rollout

```bash
kubectl set image deploy/orders \
  app=registry.example/orders:does-not-exist

kubectl rollout status deploy/orders
kubectl get pod
kubectl describe pod <new-pod>
```

Типичный результат — ``ImagePullBackOff``.

При корректной RollingUpdate старые Ready Pods могут продолжить обслуживать traffic.

Исправление:

```bash
kubectl rollout undo deploy/orders
```

---

## 19. Developer vs Platform responsibility

| Область | Developer | Platform/SRE | Shared |
|---|---:|---:|---:|
| health semantics | ✓ |  |  |
| probe timings |  |  | ✓ |
| graceful shutdown code | ✓ |  | ✓ |
| node capacity |  | ✓ |  |
| resource sizing |  |  | ✓ |
| rollout strategy |  |  | ✓ |
| PDB |  |  | ✓ |
| DB migration compatibility | ✓ |  | ✓ |
| rollout monitoring |  |  | ✓ |

---

## 20. CKAD mapping

Нужно быстро уметь:

```bash
kubectl create deployment
kubectl scale deployment
kubectl set image
kubectl rollout status
kubectl rollout history
kubectl rollout undo
kubectl describe pod
kubectl logs
kubectl logs --previous
```

Production глубже экзамена в:
- JVM/container memory;
- zero-downtime DB migration;
- capacity;
- PDB semantics;
- SLO;
- canary/blue-green;
- consumer shutdown.

---

## 21. Практическая памятка

Перед production deploy:

1. startup/readiness/liveness отвечают на разные вопросы;
2. liveness не зависит от внешней БД/очереди;
3. requests заданы осознанно;
4. memory limit оставляет JVM native headroom;
5. rollout рассчитан относительно replicas и capacity;
6. graceful shutdown укладывается в termination grace period;
7. DB migration backward-compatible;
8. rollout status контролируется;
9. проблема диагностируется через ReplicaSet/Pod/events/logs;
10. rollback plan учитывает schema/message compatibility.

---

## Sources: для проверки, а не вместо объяснения

Документ специально самодостаточен. Источники нужны для сверки спецификации и дальнейшего углубления.

- Kubernetes Deployment: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/ — ReplicaSet lifecycle, RollingUpdate, ``maxUnavailable``, ``maxSurge``.
- Kubernetes Pod lifecycle: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/ — probes и lifecycle.
- Kubernetes disruptions: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/ — voluntary disruptions и PDB.
- Spring Boot Actuator: https://docs.spring.io/spring-boot/reference/actuator/endpoints.html — liveness/readiness health groups.
- Spring Boot graceful shutdown: https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html — graceful shutdown semantics.

Проверено: **2026-09-20**.
