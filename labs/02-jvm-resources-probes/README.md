# Lab 02 — JVM, resources and probes

## Цель

Понять на практике разницу между:
- CPU request;
- CPU limit;
- memory request;
- memory limit;
- Running и Ready;
- startup/readiness/liveness;
- application crash и OOM.

## 1. Подготовка

Используйте отдельный namespace:

```bash
kubectl create namespace jvm-lab
kubectl config set-context --current --namespace=jvm-lab
```

Разверните Spring Boot image с Actuator и endpoints `/livez`, `/readyz`.

## 2. Resources

Добавьте:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Проверить:

```bash
kubectl describe pod <pod>
kubectl top pod
```

Объясните: request участвует в scheduling; limit определяет runtime boundary.

## 3. Failure — Pending из-за request

Временно поставьте:

```yaml
requests:
  cpu: "100"
```

```bash
kubectl get pod
kubectl describe pod <pod>
```

Найдите scheduler event `Insufficient cpu`.

**Вывод:** приложение даже не запускалось.

## 4. Failure — readiness

Сломайте path:

```text
/broken-readyz
```

Проверить:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice
```

Сформулируйте:

```text
JVM работает
Pod Running
readiness fails
Pod NotReady
Service не имеет ready backend
```

## 5. Failure — liveness

Сломайте liveness path и следите:

```bash
kubectl get pod -w
kubectl logs <pod> --previous
```

Объясните рост restart count.

## 6. Failure — memory

На специальном lab workload создайте memory pressure выше limit.

Ищите:

```text
OOMKilled
```

через:

```bash
kubectl describe pod <pod>
```

## 7. Проверка JVM

Сопоставьте:
- container memory;
- JVM heap;
- RSS/working set;
- thread count;
- GC.

Не делайте вывод только по heap.

## 8. Современный Kubernetes

Если cluster >=1.35, отдельно изучите in-place CPU/memory resize, но не смешивайте его с базовым пониманием requests/limits.

## Cleanup

```bash
kubectl delete namespace jvm-lab
```
