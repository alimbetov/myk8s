# Manifest Reference — HorizontalPodAutoscaler

## Учебная схема объекта

### HPA как feedback loop

```text
metrics
  |
  v
HPA
  |
  | desired replicas
  v
Deployment
  |
  v
Pods
  |
  v
new metrics
```

### Annotated fragment

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

### Ключевая скрытая связь

```text
CPU usage
   /
Deployment requests.cpu
   =
HPA utilization
```

И ещё:

```text
maxReplicas
 ×
Hikari pool / consumer concurrency
 =
downstream pressure
```

Практика: [HPA + PostgreSQL](../../../showcases/03-rest-postgres-hpa-networkpolicy/annotated.yaml).

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Назначение

`HorizontalPodAutoscaler` управляет replica count target resource через scale subresource по metrics.

```text
metrics
 -> HPA controller
 -> desired replicas
 -> target scale
 -> new/removed Pods
```

## 2. API

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
```

## 3. Example

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

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
```

## 4. scaleTargetRef

Required reference к scalable resource.

Fields:
- apiVersion
- kind
- name

Target должен поддерживать `scale` subresource.

## 5. minReplicas

**Default: 1.**

Обычный lower bound.

В Kubernetes 1.37 scale-to-zero доступен в beta при:
- `minReplicas: 0`;
- feature gate enabled;
- как минимум Object или External metric;
- resource metrics вроде CPU недостаточны, потому что при 0 Pods их невозможно измерить.

## 6. maxReplicas

Required.

Upper replica bound.

Не может быть меньше minReplicas.

Это capacity/security boundary и должно учитывать downstream systems.

## 7. metrics default

Если `metrics` не задан, default target — **80% average CPU utilization**.

Но CPU utilization требует корректных CPU requests и metrics pipeline.

## 8. Resource metric

```yaml
type: Resource
resource:
  name: cpu
  target:
    type: Utilization
    averageUtilization: 70
```

CPU utilization relative to requested CPU.

## 9. CPU calculation

Упрощённо:

```text
current CPU / requested CPU
```

Pod 350m usage, request 500m -> ~70%.

Если CPU request missing для relevant container, CPU utilization metric calculation может быть unavailable for that Pod/HPA decision.

## 10. Memory resource metric

Можно использовать memory resource.

Для JVM memory может быть плохим downscale signal: heap/RSS behavior не всегда уменьшается пропорционально traffic.

## 11. ContainerResource metric

Позволяет масштабировать по resource одного named container внутри Pod.

Полезно при sidecars, когда aggregate Pod usage искажает application signal.

## 12. Pods metric

Custom per-Pod metric.

Нужен custom metrics API adapter.

## 13. Object metric

Metric конкретного Kubernetes object.

Может использовать Value/AverageValue depending on spec.

## 14. External metric

Metric outside Kubernetes object model, например queue backlog через external metrics adapter.

Подходит queue consumers / scale-to-zero scenarios.

## 15. Multiple metrics

Если несколько metrics дают разные desired replica counts, HPA использует максимальную recommendation, с additional safeguards при metric errors.

Это conservative scale-up behavior.

## 16. Target types

- Utilization
- AverageValue
- Value

Выбор зависит от metric source.

## 17. behavior.scaleUp / scaleDown

Раздельные scaling policies.

Позволяют настроить:
- velocity;
- stabilization;
- selection policy.

## 18. stabilizationWindowSeconds

Defaults:
- scaleUp: **0**
- scaleDown: **300**

Scale-down window выбирает безопасную recommendation из recent history, уменьшая flapping.

## 19. policies

Policy types:
- Percent
- Pods

Current API defaults позволяют scale-up по более агрессивному из roughly doubling / adding fixed Pods per control period, а scale-down — значительное уменьшение с stabilization.

Явно задавать policy полезно, если workload sensitivity требует controlled velocity.

## 20. selectPolicy

Values:
- Max
- Min
- Disabled

**Default: Max** при policies.

## 21. tolerance

Modern HPA API позволяет per-direction tolerance в актуальных versions/features; cluster-wide default tolerance traditionally около 10%, если field не задан.

Не опираться на tiny metric changes как на guaranteed scaling trigger.

## 22. Initial readiness / CPU initialization

HPA controller учитывает not-yet-ready Pods conservatively.

Controller flags имеют defaults, включая initial readiness delay и CPU initialization period, чтобы startup CPU spikes не искажали decision.

Это platform-level HPA controller behavior, а не HPA object field.

## 23. Metrics Server

Resource CPU/memory HPA обычно требует Resource Metrics API provider, часто Metrics Server.

Проверить:

```bash
kubectl top pod
kubectl get apiservice | grep metrics
```

## 24. Custom metrics adapters

Pods/Object/External metrics требуют соответствующей metrics API integration/adapters.

Manifest сам не создаёт metrics pipeline.

## 25. HPA vs spec.replicas

Если GitOps постоянно возвращает Deployment `replicas` к фиксированному значению, а HPA одновременно масштабирует workload — controllers могут конфликтовать.

Нужно определить ownership replica count.

## 26. HPA + rollout

HPA меняет desired replicas, Deployment одновременно управляет old/new ReplicaSets.

Capacity planning должен учитывать maxSurge + autoscaling.

## 27. HPA + DB pools

```text
maxReplicas=50
Hikari maxPoolSize=10
=> up to ~500 app DB connections
```

Max replicas не может задаваться только по CPU.

## 28. Failure — unknown metrics

```bash
kubectl get hpa
kubectl describe hpa orders
```

Symptoms:
- current metric unknown;
- no scale;
- conditions show metric retrieval errors.

Check metrics server/adapter.

## 29. Failure — missing CPU requests

CPU utilization target задан, Pods не имеют usable CPU requests.

HPA cannot calculate expected utilization correctly.

Fix resource requests.

## 30. Failure — maxReplicas reached

HPA 20/20, load still high.

Это saturation/capacity event.

Need analyze:
- downstream;
- cluster nodes;
- target metric;
- code bottleneck.

## 31. Failure — flapping

Replica count oscillates due noisy metrics / aggressive policy.

Use:
- stabilization;
- better metric;
- appropriate tolerance/policies.

## 32. Day-2

```bash
kubectl get hpa
kubectl describe hpa orders
kubectl top pod
kubectl get deploy orders
kubectl get events --sort-by=.lastTimestamp
```

## 33. Security / reliability

- custom metrics endpoints authenticated appropriately;
- maxReplicas protects downstream/cost;
- autoscaling not a substitute for rate limits/backpressure;
- metrics pipeline is production dependency.

## 34. Anti-patterns

- HPA without requests;
- arbitrary maxReplicas;
- CPU metric for queue workload without correlation;
- autoscale memory leak;
- ignore startup latency;
- HPA + GitOps replica tug-of-war;
- scale up app beyond DB capacity.

## 35. CKAD

Know:
- autoscaling/v2 structure;
- scaleTargetRef;
- min/max;
- Resource CPU metric;
- describe HPA;
- kubectl autoscale.

## Production-like examples

- [HPA + CPU request + DB capacity](../../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)

Каждый showcase содержит `WALKTHROUGH.md` с разбором mapping'ов и `all.yaml` с полной композицией.

## Sources

- https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/
- https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/
