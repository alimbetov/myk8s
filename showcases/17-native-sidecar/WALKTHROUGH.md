# Walkthrough — Native Kubernetes sidecar

## Архитектура

```text
Pod
 |
 +--> native sidecar telemetry-agent
 |       |
 |       +--> :4318
 |
 +--> Spring Boot app
         |
         +--> localhost:4318
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [Pod reference](../../docs/knowledge/manifests/pod.md)

## Native sidecar syntax

```yaml
initContainers:
  - name: telemetry-agent
    restartPolicy: Always
```

Это special sidecar init container.

В отличие от обычного init container, он продолжает работать весь Pod lifecycle.

## Почему localhost

Все containers Pod разделяют network namespace.

```text
app container
  127.0.0.1:4318
       |
       v
sidecar :4318
```

Service не нужен.

## Ordering

Native sidecar starts in init sequence, затем main containers могут стартовать после sidecar startup semantics.

Это полезнее обычного второго app container, когда нужен defined sidecar lifecycle.

## Resources

Не забывать:

```text
Pod footprint
 =
app resources
+ sidecar resources
+ applicable init resource scheduling semantics
```

Sidecar может стать bottleneck.

## Readiness coupling

Если telemetry best-effort, не делайте business readiness полностью зависимой от telemetry backend.

Если sidecar security-critical proxy, coupling может быть intentional.

## Failure scenarios

### Sidecar image unavailable
Pod startup affected.

### Sidecar repeatedly crashes
App может работать, но telemetry/proxy capability degraded.

### Wrong localhost port
App cannot reach sidecar.

### CPU throttled sidecar
Backlog/latency grows.

## Проверка

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl logs <pod> -c telemetry-agent
kubectl logs <pod> -c app
```

## Главное

Sidecar — часть одного workload unit. Если сервисы должны independently scale/deploy/fail, они не должны быть sidecars друг другу.
