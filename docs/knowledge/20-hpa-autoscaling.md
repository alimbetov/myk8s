# 20 — HPA / autoscaling

Проверено: 2026-09-20.

Autoscaling — не «если CPU высокий, добавь Pods». Это feedback loop, который должен учитывать requests, startup time, downstream capacity и scale-down safety.

## 1. HPA mental model

```text
metrics
 -> HPA controller
 -> desired replicas
 -> Deployment scale
 -> scheduler
 -> new Pods
 -> readiness
 -> traffic capacity changes
```

Между signal и реальной capacity есть delay.

## 2. Basic HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## 3. CPU utilization и requests

Если target 70%, utilization считается относительно CPU request.

```text
request=500m
actual=350m
=> ~70%
```

Поэтому request — часть autoscaling algorithm.

Плохой request = плохой HPA signal.

## 4. Почему CPU не всегда хороший metric

CPU хорошо коррелирует с capacity для CPU-bound workload.

Но плохо для:
- async queue worker;
- IO-bound service;
- external dependency bottleneck;
- DB-bound API.

Тогда useful metrics:
- queue depth;
- requests/sec;
- concurrency;
- custom/external metrics.

## 5. Memory HPA

Memory часто плохо scale-down signal для JVM: heap может не возвращаться OS быстро после load drop.

Поэтому memory-based HPA требует понимания JVM behavior.

## 6. Scale up latency

```text
load spike
 -> metric collected
 -> HPA reacts
 -> Pod scheduled
 -> image pulled
 -> JVM starts
 -> readiness
 -> capacity available
```

Если это 60 секунд, HPA не спасает мгновенный spike сам по себе.

Нужны:
- baseline replicas;
- queues/backpressure;
- capacity margin;
- fast startup.

## 7. Default stabilization behavior

В autoscaling/v2 default scale-down stabilization window — 300 секунд.

Scale-up по умолчанию без stabilization window.

Это помогает не удалять replicas сразу после короткого падения load.

## 8. Custom behavior

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
      - type: Percent
        value: 25
        periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
      - type: Percent
        value: 100
        periodSeconds: 60
```

Так можно ограничить velocity.

## 9. HPA + DB

```text
10 Pods × pool10 = 100 DB connections
HPA max 50
=> 500 potential connections
```

Если DB max safe application connections 200, `maxReplicas: 50` может быть опасным.

Autoscaling должен учитывать downstream.

## 10. HPA + Kafka/Rabbit workers

Queue metric часто лучше CPU:

```text
backlog grows
 -> add consumers
```

Но число consumers может ограничиваться:
- Kafka partitions;
- DB writes;
- external rate limits.

## 11. HPA and rollout

Во время rollout HPA и Deployment controllers одновременно меняют replica sets/desired count.

Нужно понимать:
- surge capacity;
- metrics new/old Pods;
- startup readiness.

## 12. Metrics Server

CPU/memory Resource metrics обычно требуют Metrics Server/compatible metrics pipeline.

Проверить:

```bash
kubectl top pod
kubectl get hpa
kubectl describe hpa orders
```

## 13. Custom/external metrics

Требуют metrics adapters/platform integration.

Это production platform responsibility, не просто HPA YAML.

## 14. Scale to zero

Kubernetes 1.37 развивает HPA scale-to-zero возможности для workloads с подходящими object/external metrics. Это modern advanced feature и требует внимательной проверки API/feature requirements.

Для обычного HTTP Spring Boot backend `minReplicas >= 1` остаётся простой baseline.

## 15. VPA

Vertical Pod Autoscaler решает другую задачу: изменяет/recommends resource requests.

Не надо одновременно бесконтрольно позволять HPA и VPA управлять одной и той же CPU signal dimension без продуманной policy.

## 16. In-place resize

Современный Kubernetes умеет менять CPU/memory container resources in-place (stable с 1.35), что улучшает vertical scaling possibilities.

Но horizontal и vertical scaling решают разные problems:
- HPA = больше/меньше replicas;
- vertical = больше/меньше resource per Pod.

## 17. Failure practice: wrong CPU request

Поставьте request очень маленьким. Под умеренной нагрузкой HPA увидит high utilization.

Сравните:
- `kubectl top pod`;
- request;
- `kubectl describe hpa`.

## 18. Failure practice: max replicas hits ceiling

Load растёт, HPA уже 20/20.

Это saturation signal.

Следующий шаг — не менять YAML вслепую, а определить bottleneck/capacity.

## 19. Failure practice: scale oscillation

Слишком aggressive policies + noisy metric.

Добавьте stabilization/appropriate metric.

## 20. Autoscaling readiness

Pod не создаёт capacity до Ready. Поэтому startup probe/readiness напрямую влияют на autoscaling response time.

## 21. Anti-patterns

- HPA without requests;
- CPU HPA для queue backlog без анализа;
- maxReplicas arbitrary;
- ignore DB limits;
- autoscale to hide memory leak;
- no stabilization;
- no capacity on nodes.

## 22. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| workload metric | ✓ | | ✓ |
| metrics pipeline | | ✓ | |
| requests | | | ✓ |
| HPA policy | | | ✓ |
| cluster autoscaler | | ✓ | |
| downstream limits | ✓ | ✓ | ✓ |

## 23. CKAD

HPA basics могут быть application operational knowledge; production требует behavior policies, custom metrics, JVM/downstream analysis.

## Связанные production-like примеры

- [HPA + Spring Boot + PostgreSQL capacity](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)

Для каждого стенда откройте `README.md` → `WALKTHROUGH.md` → `all.yaml`.

## Sources

- https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/

Проверено: **2026-09-20**.
