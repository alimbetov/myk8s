# 08 — Resources, JVM memory, CPU, cgroups и autoscaling

Проверено: 2026-09-20.

Эта глава отвечает на вопрос:

> JVM уже находится внутри container. **Сколько CPU и memory ей дать, что реально означают requests/limits, почему heap не равен container memory, откуда берётся OOMKilled, что такое CPU throttling и почему одно поле cpu request влияет сразу на scheduler и HPA?**

Главная mental model:

```text
Node capacity
    ↓
Scheduler
    |
    | uses requests
    v
Pod scheduled
    ↓
Linux cgroups
    |
    +--> CPU controls
    +--> Memory controls
    |
    v
JVM
    |
    +--> heap
    +--> metaspace
    +--> thread stacks
    +--> direct buffers
    +--> code cache
    +--> native memory
    |
    v
Spring Boot
    |
    +--> HTTP concurrency
    +--> Hikari pool
    +--> Kafka/Rabbit consumers
```

И отдельно:

```text
CPU request
    ↓
scheduler
    ↓
HPA utilization denominator

memory limit
    ↓
cgroup ceiling
    ↓
JVM total process memory
    ↓
possible OOMKilled
```

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь объяснить:

1. Что такое resource request.
2. Что такое resource limit.
3. Почему request не означает фактическое usage.
4. Как scheduler использует requests.
5. Почему CPU и memory limits ведут себя по-разному.
6. Что такое CPU throttling.
7. Что такое OOMKilled.
8. Почему JVM heap != process RSS.
9. Почему `-Xmx == memory limit` опасно.
10. Что такое metaspace/direct memory/thread stacks.
11. Как оценить memory headroom.
12. Что делает `MaxRAMPercentage`.
13. Как JVM учитывает container/cgroup constraints.
14. Почему слишком маленький CPU request влияет на HPA.
15. Как считается CPU utilization для HPA.
16. Почему HPA scale-out увеличивает Hikari connections.
17. Почему requests влияют на rollout capacity.
18. Что такое QoS classes.
19. Чем BestEffort/Burstable/Guaranteed отличаются концептуально.
20. Когда CPU limit полезен, а когда может вредить latency.
21. Почему memory limit обычно нужен как safety boundary.
22. Как диагностировать Pending из-за ресурсов.
23. Как диагностировать OOMKilled.
24. Как диагностировать CPU saturation без Pod restart.
25. Что такое in-place resource resize.
26. Что такое Pod-level resources.
27. Какие из этих возможностей stable/beta в Kubernetes 1.37.
28. Что должен понимать аналитик, developer, tester и platform engineer.

---

# 2. Четыре базовых значения

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 768Mi
```

```text
requests
= planning / scheduling input

limits
= runtime ceiling/control
```

Но CPU и memory реализуются по-разному.

---

# 3. CPU units

```text
1000m = 1 CPU
500m  = 0.5 CPU
100m  = 0.1 CPU
```

“1 CPU” — это Kubernetes compute unit, обычно соответствующий одному logical CPU/vCPU capacity unit, а не обещание dedicated physical core.

---

# 4. Memory units

Примеры:

```text
128Mi
512Mi
1Gi
```

Не путать binary `Mi/Gi` и decimal `M/G`.

---

# 5. CPU request

```yaml
requests:
  cpu: 500m
```

Scheduler использует request при placement.

```text
Node allocatable = 4 CPU

A=1
B=1
C=1.5
D=1

sum=4.5
```

D может остаться Pending, даже если остальные Pods сейчас idle.

> Scheduler планирует по declared requests, а не по мгновенному usage.

---

# 6. Memory request

```yaml
requests:
  memory: 512Mi
```

Это scheduling/capacity input, а не заранее выделенный Java heap.

---

# 7. Requests как capacity contract

Too low:

```text
scheduler packs too many Pods
        ↓
node pressure risk
```

Too high:

```text
cluster appears full
        ↓
Pods Pending / poor utilization
```

---

# 8. CPU limit

```yaml
limits:
  cpu: "1"
```

Если workload хочет больше CPU time:

```text
demand > quota
    ↓
cgroup throttling
```

Process обычно остаётся жив.

---

# 9. CPU throttling

```text
CPU demand > limit
      ↓
less CPU time
      ↓
latency grows
throughput falls
JIT/GC/startup slow
```

Pod может быть:

```text
Running
Ready
0 restarts
```

и при этом иметь плохой p99.

---

# 10. Memory limit

```yaml
limits:
  memory: 768Mi
```

Если process превышает effective cgroup memory:

```text
memory pressure
   ↓
OOM handling
   ↓
process killed
   ↓
container terminated
```

---

# 11. CPU и memory — разные failure semantics

```text
CPU limit
  -> throttling

Memory limit
  -> possible kill
```

---

# 12. JVM memory model

```text
JVM process RSS
  |
  +--> heap
  +--> metaspace
  +--> code cache
  +--> thread stacks
  +--> direct buffers
  +--> GC/native structures
  +--> JNI/native libraries
```

Следовательно:

```text
heap < total process memory
```

---

# 13. Почему Xmx = limit опасно

```text
memory limit = 768Mi
-Xmx = 768Mi
```

Но остаются:

```text
metaspace
threads
direct memory
code cache
native memory
```

Итого:

```text
RSS > limit
  ↓
OOMKilled
```

---

# 14. Пример memory budget

Иллюстрация:

```text
limit             1024Mi
heap               650Mi
metaspace          100Mi
thread stacks       80Mi
direct buffers      70Mi
code/native         70Mi
headroom            54Mi
```

Числа должны подтверждаться profiling/load test.

---

# 15. MaxRAMPercentage

```text
-XX:MaxRAMPercentage=65
```

Conceptually:

```text
max heap
≈ percentage of JVM-visible RAM
```

Это удобный mechanism, но процент нельзя копировать без измерений.

---

# 16. Fixed Xmx vs percentage

Fixed:

```text
-Xms512m
-Xmx512m
```

Percentage:

```text
-XX:MaxRAMPercentage=65
```

Оба подхода production-valid при корректном измерении native headroom.

---

# 17. Metaspace

Metaspace хранит class metadata.

Он может расти из-за framework/classpath, dynamic classes или classloader leaks. При memory incident нельзя смотреть только heap.

---

# 18. Thread stacks

Sources:

```text
Tomcat/Jetty workers
executors
Kafka listeners
Rabbit consumers
Schedulers
libraries
```

Больше platform threads → больше native stack memory.

---

# 19. Virtual threads

Virtual threads дешевле platform threads, но:

```text
more virtual threads
!= more CPU
!= more Hikari connections
!= more DB capacity
!= more Kafka partitions
```

Concurrency downstream всё равно bounded.

---

# 20. Direct memory

Netty/NIO и другие libraries используют off-heap/direct buffers.

```text
heap looks normal
RSS high
```

может означать native/direct usage.

---

# 21. JVM container awareness

Современная JVM учитывает cgroup constraints при ergonomics.

Это влияет на heap sizing, processor count, GC и runtime heuristics.

Но container awareness не заменяет capacity planning.

---

# 22. CPU resources и JVM

Production performance test должен выполняться под realistic cgroup resources.

```text
Laptop:
12 cores

Production Pod:
request=500m
limit=1 CPU
```

Это разные environments.

---

# 23. CPU request и HPA

Упрощённо:

```text
CPU utilization
=
actual CPU
-----------
CPU request
```

Пример:

```text
actual=350m
request=500m
utilization=70%
```

---

# 24. Один request меняет autoscaling semantics

При actual=350m:

```text
request=500m  -> 70%
request=250m  -> 140%
request=1000m -> 35%
```

CPU request одновременно:

```text
scheduler input
+
HPA denominator
```

---

# 25. Missing request и HPA

Percentage-based CPU resource utilization требует meaningful CPU request. Без него HPA не может нормально вычислять utilization этого container/Pod в обычной модели.

---

# 26. HPA formula mental model

Упрощённо:

```text
desired replicas
≈
current replicas
×
current metric
--------------
target metric
```

Пример:

```text
4 replicas
80% current
40% target

≈ 8 replicas
```

Реальный HPA добавляет tolerance, stabilization и другие rules.

---

# 27. Scale-out увеличивает downstream pressure

```text
Hikari pool=10
replicas=5
=> ~50 connections

replicas=20
=> ~200 connections
```

Поэтому:

```text
HPA maxReplicas
× per-Pod pool/concurrency
=
downstream capacity requirement
```

---

# 28. Autoscaling amplification

```text
DB slow
  ↓
request latency
  ↓
CPU/work grows
  ↓
HPA adds Pods
  ↓
more DB pools
  ↓
DB gets worse
```

Autoscaling может усилить bottleneck.

---

# 29. Requests и rollout capacity

```text
replicas=10
maxSurge=2
cpu request=500m
memory request=512Mi
```

Extra rollout demand:

```text
1 CPU
1Gi memory
```

Без spare capacity new Pods Pending.

---

# 30. Resources и startup time

Low CPU может замедлить:

- JVM startup;
- Spring context;
- JIT;
- Liquibase;
- cache warmup.

Это влияет на startupProbe и rollout.

---

# 31. Probe timeout из-за CPU starvation

```text
CPU throttling
  ↓
/readyz slow
  ↓
probe timeout
  ↓
Ready=False
  ↓
Service removes endpoint
```

Resources могут выглядеть как networking/availability problem.

---

# 32. QoS classes

```text
Guaranteed
Burstable
BestEffort
```

QoS влияет на behavior при node pressure/eviction.

---

# 33. BestEffort

Без CPU/memory requests/limits:

```text
BestEffort
```

Для critical backend обычно плохой baseline.

---

# 34. Guaranteed

Классическая модель:

```text
CPU request = CPU limit
memory request = memory limit
for every container
```

Это даёт Guaranteed QoS, но CPU burst выше limit невозможен.

---

# 35. Burstable

Например:

```yaml
requests:
  cpu: 500m
  memory: 512Mi
limits:
  memory: 768Mi
```

Это типичный Burstable web workload.

---

# 36. CPU limit: trade-off

С limit:

- ceiling;
- multi-tenant governance;
- possible throttling.

Без CPU limit:

- Pod может burst на spare CPU;
- нужен platform governance/noisy-neighbor control.

Нет универсального “всегда ставить” или “никогда не ставить”.

---

# 37. Memory limit как safety boundary

Без limit runaway process может давить node.

С limit blast radius лучше ограничен container, но плохое sizing даст OOMKilled.

---

# 38. Limit without request

Если limit задан, request отсутствует, Kubernetes может default request к limit для этого resource, если admission не задал другое.

Поэтому задавайте requests явно и смотрите admitted Pod spec.

---

# 39. LimitRange

Namespace `LimitRange` может задавать:

- defaults;
- min;
- max;
- default requests/limits.

То есть source YAML без resources может превратиться admission'ом в Pod с resources.

---

# 40. ResourceQuota

Namespace quota может ограничить aggregate requested CPU/memory.

Rollout может не создавать новые Pods из-за quota, даже если node capacity есть.

---

# 41. Pending decision tree

```text
Pod Pending
   |
   v
describe/events
   |
   +--> Insufficient cpu
   +--> Insufficient memory
   +--> quota/admission
   +--> PVC
   +--> affinity/topology
```

---

# 42. OOMKilled

```bash
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml
kubectl logs <pod> --previous
```

Ищите terminated reason:

```text
OOMKilled
```

---

# 43. Java heap OOM vs cgroup OOM

Heap OOM:

```text
java.lang.OutOfMemoryError: Java heap space
```

Cgroup OOM:

```text
external kill
reason=OOMKilled
```

Не всегда Java успевает записать красивую exception.

---

# 44. Native-memory OOM example

```text
heap=400Mi
threads=150Mi
direct=100Mi
metaspace/native=150Mi

total≈800Mi
limit=768Mi
```

Heap не заполнен, а container убит.

---

# 45. Что измерять для Java sizing

Минимум:

- container RSS/working set;
- heap used/max;
- metaspace;
- GC;
- thread count;
- direct/off-heap;
- CPU;
- throttling;
- latency;
- Hikari pool;
- restart/OOM.

---

# 46. Sizing method

```text
representative load
   ↓
steady state
   ↓
peak
   ↓
headroom
   ↓
request/limit proposal
   ↓
load/failure test
   ↓
observe HPA/rollout
```

---

# 47. Memory request selection

Too low:

```text
overpacking
→ node pressure risk
```

Too high:

```text
wasted capacity
→ Pending / larger cluster
```

---

# 48. CPU request selection

CPU request влияет на:

```text
scheduling
QoS
HPA utilization
capacity planning
```

Поэтому выбирать его нужно по измерениям.

---

# 49. Workload profiles

REST API:

- latency;
- bursts;
- DB pool;
- steady memory.

Consumer:

- processing concurrency;
- downstream;
- backlog recovery.

Batch:

- completion time;
- peak CPU/memory;
- scheduling window.

Один resource template для всех workloads плох.

---

# 50. In-place resize

Container-level in-place CPU/memory resize:

```text
stable since Kubernetes 1.35
```

Через Pod `/resize` subresource можно менять desired CPU/memory без обязательного Pod recreation.

---

# 51. resizePolicy

```yaml
resizePolicy:
  - resourceName: cpu
    restartPolicy: NotRequired
  - resourceName: memory
    restartPolicy: RestartContainer
```

Policy задаёт, нужен ли restart container при resize.

---

# 52. Resize conditions

```text
PodResizePending
PodResizeInProgress
```

Resize тоже имеет operational lifecycle.

---

# 53. In-place resize не заменяет Deployment

Для обычного declarative release остаётся стандарт:

```text
Deployment template resources change
  ↓
controller reconciles workload
```

In-place resize — дополнительный runtime capability, а не повод отказаться от desired-state management.

---

# 54. Pod-level resources

Aggregate Pod budget:

```yaml
spec:
  resources:
    requests:
      cpu: "2"
      memory: 2Gi
```

Полезно для Pods с несколькими containers/sidecars.

---

# 55. Status modern features в Kubernetes 1.37

- container-level in-place CPU/memory resize — stable с 1.35;
- Pod-level in-place resize — beta с 1.36;
- Pod-level resource managers — beta с 1.37, disabled by default.

Это advanced platform functionality.

Базовая модель для Spring Boot остаётся container-level `resources.requests/limits`.

---

# 56. Sidecar resource math

```text
main:
500m / 512Mi

sidecar:
100m / 128Mi
```

Pod footprint roughly:

```text
600m CPU
640Mi memory
```

Если считать только main container — capacity math неверна.

---

# 57. HPA и sidecars

Sidecar может влиять на Pod-wide metrics. Для critical autoscaling иногда лучше использовать container-specific metrics или custom workload metrics, если они лучше отражают load.

---

# 58. Memory HPA caveat

Memory может плохо коррелировать с instantaneous load:

```text
load drops
but
heap/cache remains allocated
```

Поэтому memory-based HPA требует особенно аккуратной interpretation.

---

# 59. Better autoscaling signals

Для некоторых workloads лучше:

```text
HTTP RPS
queue depth
Kafka lag
work backlog
concurrency
```

Signal должен быть causal для workload demand.

---

# 60. Saturation diagnosis

High latency может быть:

```text
CPU
GC/memory
Hikari pool
DB locks
Kafka backlog
external API
```

Не делайте вывод по одной metric.

---

# 61. CPU saturation pattern

```text
CPU near limit
throttling rises
p99 rises
Pod Ready
```

Это production incident без crash.

---

# 62. Memory pressure pattern

```text
RSS rises
GC pressure
approach limit
OOMKilled
restart
```

Причина может быть heap или native.

---

# 63. GC как cross-resource effect

```text
tight memory
  ↓
more GC
  ↓
more CPU
  ↓
latency
```

Memory incident превращается в CPU/latency incident.

---

# 64. Startup CPU demand

Startup часто требует больше CPU:

```text
class loading
Spring context
JIT
Liquibase
warmup
```

Поэтому startup budget надо тестировать при realistic resources.

---

# 65. Overcommit thinking

CPU обычно проще overcommit, memory рискованнее.

```text
actual memory > requests across many Pods
  ↓
node memory pressure
  ↓
eviction/OOM risk
```

---

# 66. QoS и eviction

QoS влияет на eviction priority under pressure, но это не HA guarantee.

BestEffort workloads обычно наиболее уязвимы.

---

# 67. LimitRange + HPA surprise

Namespace LimitRange может inject CPU request=100m.

Теперь HPA denominator — 100m.

Developer думает “я request не задавал”, но actual admitted Pod говорит обратное.

---

# 68. Capacity equations

```text
total requested CPU
=
replicas × per-Pod CPU request

total requested memory
=
replicas × per-Pod memory request
```

С sidecars учитывать total Pod footprint.

---

# 69. Downstream equations

```text
DB connections
≈ maxReplicas × Hikari maxPool

Rabbit consumers
≈ replicas × listener concurrency

Kafka useful active consumers
<= partition count
```

---

# 70. Production sizing example

```text
orders-api:
CPU request     500m
memory request  512Mi
memory limit    768Mi
Hikari pool     10
HPA max         12
maxSurge        2
```

Steady max:

```text
CPU requests    6 CPU
memory requests 6Gi
DB connections  ~120
```

During rollout:

```text
14 Pods
7 CPU requested
7Gi requested memory
```

Эта математика должна быть известна до production.

---

# 71. Analyst view

Analyst задаёт measurable load/capacity requirements.

| Requirement | Example |
|---|---|
| peak RPS | 500 |
| latency SLO | p95 < 300ms |
| concurrency | 100 |
| min replicas | 3 |
| max replicas | 12 |
| startup SLO | < 60s |
| service DB connection budget | 120 |
| backlog recovery | < 15 min |

---

# 72. Developer view

Checklist:

- [ ] representative load test;
- [ ] CPU request measured;
- [ ] memory request realistic;
- [ ] memory limit has native headroom;
- [ ] Xmx/MaxRAMPercentage coordinated;
- [ ] threads/direct memory considered;
- [ ] startup under realistic CPU;
- [ ] HPA denominator understood;
- [ ] Hikari/downstream budget calculated;
- [ ] sidecars included;
- [ ] OOM/throttling metrics available.

---

# 73. Tester/SDET view

Tests:

- impossible CPU request -> Pending;
- low memory limit -> OOMKilled;
- low CPU limit -> throttling/latency;
- startup under pressure;
- readiness timeout from starvation;
- HPA scale-up;
- wrong HPA request denominator;
- downstream saturation;
- rollout with no surge capacity;
- sidecar footprint;
- in-place resize where supported.

---

# 74. Platform/SRE view

Platform owns/shared:

- node sizing;
- allocatable;
- LimitRange;
- ResourceQuota;
- metrics pipeline;
- HPA platform;
- CPU policy;
- memory pressure/eviction;
- resize support;
- Pod-level feature gates;
- capacity dashboards.

---

# 75. Failure lab — impossible CPU request

```yaml
requests:
  cpu: "100"
```

Observe:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

Expected:

```text
Pending
FailedScheduling
Insufficient cpu
```

---

# 76. Failure lab — OOMKilled

Low memory limit + controlled load.

Observe:

```bash
kubectl describe pod <pod>
kubectl logs <pod> --previous
```

Compare heap vs RSS.

---

# 77. Failure lab — CPU throttling

```yaml
limits:
  cpu: 200m
```

Under load observe:

- CPU;
- throttling;
- p95/p99;
- readiness;
- GC.

Goal:

```text
healthy Pod status
can coexist with bad performance
```

---

# 78. Failure lab — HPA denominator

Same absolute load, compare:

```text
request=250m
request=1000m
```

Observe reported utilization/scaling.

---

# 79. Failure lab — rollout without capacity

```text
replicas=5
maxSurge=2
memory request=1Gi
```

Make cluster unable to fit extra 2Gi.

Expected:

```text
old Pods healthy
new Pods Pending
rollout stalled
```

---

# 80. Failure lab — DB amplification

```text
HPA max=20
Hikari pool=10
```

Observe scale-out together with DB connection count and DB latency.

---

# 81. Failure lab — in-place resize

On supported cluster:

1. run Pod with explicit CPU/memory;
2. inspect resources;
3. request resize through `/resize` workflow;
4. inspect resize conditions;
5. check restart according to `resizePolicy`.

Goal: resource change no longer always requires Pod replacement.

---

# 82. Anti-patterns

1. Requests omitted because “cluster has enough”.
2. Copy-paste 500m/512Mi everywhere.
3. Xmx = memory limit.
4. Size only by heap.
5. Ignore native/direct/thread memory.
6. CPU limit too low for startup/JIT.
7. CPU request chosen only to manipulate HPA.
8. HPA max without DB capacity math.
9. Scale consumers beyond partition/downstream capacity.
10. Ignore sidecar resources.
11. Guaranteed QoS by habit without latency analysis.
12. No load test.
13. Diagnose latency only by Pod status.
14. Raise memory limit forever instead of finding leak.
15. Add replicas instead of finding bottleneck.
16. Trust source YAML instead of actual admitted Pod.
17. Treat in-place resize as replacement for declarative workload management.

---

# 83. CKAD mapping

Нужно быстро уметь:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

И диагностировать:

```text
Pending
OOMKilled
```

Useful commands:

```bash
kubectl describe pod
kubectl top pod
kubectl get events
```

Production добавляет cgroups, JVM native memory, throttling, HPA/downstream amplification и modern resize.

---

# 84. Связь с предыдущей главой

```text
Chapter 07:
JVM inside container

Chapter 08:
container inside cgroup resource boundary
```

Полная цепочка:

```text
Node
 ↓
Scheduler
 ↓
Pod
 ↓
cgroup
 ↓
JVM
 ↓
Spring
 ↓
downstream dependencies
```

---

# 85. Что изучать дальше

Следующая глава:

- [09 — Spring Boot probes and lifecycle](09-spring-boot-probes-lifecycle.md)

```text
Resources define
how much CPU/memory JVM gets

Next question:
how does Kubernetes know
JVM is starting,
alive,
and ready for traffic?

        ↓

Chapter 09
```

---

# 86. Production-like examples

- [Showcase 03 — REST + PostgreSQL + HPA + NetworkPolicy](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)
- [Showcase 03 annotated.yaml](../../showcases/03-rest-postgres-hpa-networkpolicy/annotated.yaml)
- [Showcase 17 — native sidecar](../../showcases/17-native-sidecar/README.md)
- [Lab 02 — JVM/resources/probes](../../labs/02-jvm-resources-probes/README.md)
- [Lab 04 — rollout/HPA](../../labs/04-rollout-hpa/README.md)

---

# 87. Control questions

1. Что делает CPU request?
2. Что делает memory request?
3. Чем request отличается от usage?
4. Что делает CPU limit?
5. Что делает memory limit?
6. Чем throttling отличается от OOMKilled?
7. Почему heap != RSS?
8. Почему Xmx=limit опасен?
9. Что делает MaxRAMPercentage?
10. Почему threads/direct memory важны?
11. Почему virtual threads не увеличивают DB capacity?
12. Как CPU request связан с HPA?
13. Что будет при 350m usage и 500m request?
14. Что будет при 350m и 250m request?
15. Почему missing request мешает CPU utilization HPA?
16. Почему scale-out может ухудшить DB?
17. Как maxSurge связан с resources?
18. Что такое LimitRange?
19. Что такое ResourceQuota?
20. Чем BestEffort/Burstable/Guaranteed отличаются?
21. Почему Guaranteed не автоматически лучший?
22. Что проверять при Pending?
23. Что означает OOMKilled?
24. Чем heap OOM отличается от cgroup OOM?
25. Может ли Pod быть Ready при throttling?
26. С какой версии container in-place resize stable?
27. Что делает resizePolicy?
28. Какие resize conditions существуют?
29. Что такое Pod-level resources?
30. Каков статус Pod-level resize в 1.37?
31. Каков статус Pod-level resource managers в 1.37?

---

# 88. Sources

## Kubernetes

- https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/
- https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/
- https://kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/
- https://kubernetes.io/docs/concepts/resource-management/pod-level-resource-managers/
- https://kubernetes.io/docs/concepts/policy/limit-range/
- https://kubernetes.io/docs/concepts/policy/resource-quotas/

## JVM

- https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html

Актуальные facts главы:

- Scheduler uses requests for placement.
- CPU limits throttle CPU time; memory limits can result in OOM termination.
- CPU HPA utilization is relative to resource requests.
- Container-level in-place CPU/memory resize is stable since Kubernetes 1.35.
- Pod-level in-place resize is beta since Kubernetes 1.36.
- Pod-level resource managers are beta in Kubernetes 1.37 and disabled by default.
- JVM container awareness does not make heap equal total process memory; native headroom remains necessary.
