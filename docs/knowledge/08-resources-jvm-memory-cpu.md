# 08 — Resources, JVM memory and CPU

## Учебная карта темы

### Где resources находятся в системе

```text
Node capacity
   |
   v
Scheduler
   |
   | uses requests
   v
Pod
   |
   +--> CPU request/limit
   +--> Memory request/limit
   |
   v
cgroups
   |
   v
JVM
   |
   +--> heap
   +--> metaspace
   +--> thread stacks
   +--> direct buffers
   +--> code cache
```

### Главное различие

```text
request = сколько ресурсов резервируем / как scheduler считает
limit   = runtime ceiling
```

Для JVM:

```text
memory limit
  !=
-Xmx
```

Потому что RSS JVM включает не только heap.

### Annotated fragment

```yaml
resources:
  requests:
    cpu: 500m       # scheduler + HPA denominator
    memory: 512Mi   # requested node capacity
  limits:
    memory: 768Mi   # cgroup ceiling; превышение может дать OOMKilled
```

### Важная системная связь

```text
CPU request
   ↓
HPA Utilization

memory limit
   ↓
JVM sizing

replicas
   ↓
total cluster + DB/client pressure
```

Практика: [REST + HPA + PostgreSQL](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md).

## Для аналитика, разработчика и тестировщика

### Аналитик

Переводит требования по нагрузке и SLA в measurable capacity assumptions: expected concurrency, response time, scale range и ограничения downstream.

### Разработчик

Понимает JVM memory budget и измеряет requests/limits вместо случайных значений. Связывает CPU request с HPA и replica count с connection pools.

### Тестировщик

Проверяет:
- Pending из-за ресурсов;
- OOMKilled;
- CPU throttling;
- startup под limit;
- поведение при HPA scale;
- capacity degradation before hard failure.

### Перед следующей главой

Нужно уметь объяснить разницу:

```text
CPU request
CPU limit
memory request
memory limit
JVM heap
process RSS
```

Проверено: 2026-09-20.

Resources — это договор между приложением, scheduler и Linux cgroups. Для Java это особенно важно: JVM heap — лишь часть container memory.

## 1. Четыре основных значения

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 768Mi
```

## 2. CPU request

`500m` = половина CPU.

Scheduler использует request при placement:

```text
node allocatable = 4 CPU
A request = 1
B request = 1
C request = 2
D request = 0.5 -> Pending
```

Даже если A/B/C сейчас почти idle.

Request — scheduling promise, не текущий usage.

## 3. Memory request

Memory request тоже участвует в scheduling. Это не заранее выделенный Java heap.

```yaml
requests:
  memory: 512Mi
```

говорит scheduler, сколько memory учитывать при размещении.

## 4. CPU limit

CPU limit реализуется через cgroup throttling.

```text
usage wants > limit
 -> CPU time throttled
 -> process жив
 -> latency/throughput ухудшаются
```

Слишком низкий CPU limit способен проявляться как:
- медленный startup;
- probe timeout;
- долгий GC;
- высокий p99.

## 5. Memory limit

Memory limit — жёстче.

```text
container exceeds usable cgroup memory
 -> kernel/container runtime pressure
 -> process may be OOMKilled
```

Проверить:

```bash
kubectl describe pod <pod>
kubectl logs <pod> --previous
```

Ищите `OOMKilled`.

## 6. CPU и memory нельзя объяснять одинаково

```text
CPU limit    -> throttling
Memory limit -> possible process kill
```

Это важная operational разница.

## 7. JVM memory budget

```text
container limit
  = heap
  + metaspace
  + code cache
  + stacks
  + direct buffers
  + native libs
  + JVM native memory
```

Поэтому:

```text
-Xmx768m
memory limit=768Mi
```

— плохой baseline.

## 8. Пример budget

Это иллюстрация, не универсальная настройка:

```text
container limit  1024 Mi
heap              650 Mi
metaspace         100 Mi
threads/native    120 Mi
direct/code       100 Mi
headroom           54 Mi
```

Реальные числа подтверждаются profiling/load tests.

## 9. Connection pools и memory/CPU

Увеличение replicas/resources не означает, что downstream выдержит рост.

```text
30 Pods
× Hikari 10
= 300 DB connections
```

Resources Pod и capacity dependencies проектируются вместе.

## 10. QoS

Классические QoS classes:
- BestEffort;
- Burstable;
- Guaranteed.

BestEffort возникает без requests/limits и обычно плох для критичного backend.

Guaranteed требует согласованного requests=limits для CPU/memory containers.

QoS влияет на поведение при node pressure/evictions.

## 11. CPU limit — trade-off

Нет универсального правила «CPU limit всегда нужен» или «никогда не нужен».

С limit:
- защита multi-tenant cluster;
- предсказуемый ceiling;
- возможен throttling.

Без CPU limit:
- Pod может использовать spare CPU;
- лучше burst;
- нужен platform governance.

Memory limit обычно более важен как safety boundary.

## 12. Metrics перед sizing

Нужны минимум:
- CPU usage;
- throttling;
- working set/RSS;
- JVM heap used/max;
- GC;
- thread count;
- request latency;
- OOM/restarts.

Не sizing по «кажется, 512Mi хватит».

## 13. Requests и HPA

CPU HPA обычно рассчитывает utilization относительно CPU request.

Если request слишком маленький, HPA может считать utilization огромным и масштабировать слишком агрессивно.

Если request слишком большой — недомасштабировать.

Поэтому request — ещё и denominator autoscaling.

## 14. Limit без request

Если limit задан, а request нет, Kubernetes может использовать limit как request для этого resource, если admission не установил другое значение.

Поэтому requests нужно задавать явно.

## 15. Новое: in-place Pod resize

Исторически изменение CPU/memory resource values требовало replacement Pod.

Современный Kubernetes поддерживает in-place resize container CPU/memory; feature стабильна с v1.35.

Модель:

```text
раньше:
change resources
 -> replace Pod

сейчас:
resize subresource
 -> kubelet меняет cgroup allocation
 -> restart зависит от resizePolicy/resource
```

Но для обучения и обычных Deployment manifests классическая модель `resources.requests/limits` остаётся основной.

## 16. Pod-level resources

Современные Kubernetes версии развивают resource budget на уровне всего Pod. В 1.37 связанные Pod-level resource manager возможности находятся в beta и требуют feature configuration.

Это advanced topic. Не подменяйте им понимание container requests/limits.

## 17. Failure practice: Pending

Поставьте request больше доступного node:

```yaml
requests:
  cpu: "100"
```

Диагностика:

```bash
kubectl get pod
kubectl describe pod <pod>
```

Ищите `Insufficient cpu`.

## 18. Failure practice: OOMKilled

Создайте controlled memory load в lab workload с низким limit.

Проверить:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl logs <pod> --previous
```

Root cause формулируется не как «CrashLoopBackOff», а «process OOMKilled после превышения memory budget».

## 19. Failure practice: CPU throttling

Под нагрузкой сравните:
- CPU usage;
- throttling metrics;
- request latency;
- limit.

Kubernetes status может выглядеть Healthy, пока пользователи видят высокий p99.

## 20. Requests vs capacity

RollingUpdate:

```yaml
replicas: 10
maxSurge: 2
```

В момент rollout scheduler должен разместить до 12 Pods.

Значит requests влияют и на способность выполнить zero-downtime rollout.

## 21. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| JVM profiling | ✓ | | |
| request/limit proposal | ✓ | | ✓ |
| node sizing | | ✓ | |
| quotas/LimitRange | | ✓ | |
| HPA inputs | | | ✓ |
| OOM analysis | ✓ | ✓ | ✓ |

## 22. CKAD

Нужно быстро писать resources и диагностировать Pending/OOM. Production глубже: cgroups, JVM native memory, throttling, HPA interaction, in-place resize.

## 23. Checklist

- requests заданы;
- memory limit осознан;
- JVM имеет native headroom;
- CPU throttling наблюдается;
- HPA denominator корректен;
- rollout surge помещается;
- pool/concurrency соответствуют downstream;
- OOM runbook есть.

## Связанные production-like примеры

- [REST + HPA — CPU request as autoscaling denominator](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)
- [Sidecar — Pod resource footprint](../../showcases/17-native-sidecar/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/
- https://kubernetes.io/docs/concepts/resource-management/pod-level-resource-managers/

Проверено: **2026-09-20**.
