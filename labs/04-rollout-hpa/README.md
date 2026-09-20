# Lab 04 — RollingUpdate and HPA

## Цель

Понять:
- ReplicaSet revisions;
- maxSurge/maxUnavailable;
- broken rollout;
- rollback;
- HPA relation to CPU request.

## 1. Deployment

Создайте 3 replicas и:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Наблюдайте:

```bash
kubectl get rs,pod -w
```

## 2. Update

```bash
kubectl set image deploy/spring-app   app=<valid-new-image>

kubectl rollout status deploy/spring-app
kubectl rollout history deploy/spring-app
```

Нарисуйте sequence старых/новых Pods.

## 3. Broken rollout

```bash
kubectl set image deploy/spring-app   app=example.invalid/image:nope
```

Проверить:

```bash
kubectl get rs,pod
kubectl describe pod <new-pod>
kubectl rollout status deploy/spring-app
```

Rollback:

```bash
kubectl rollout undo deploy/spring-app
```

Объясните, почему это не rollback DB.

## 4. HPA prerequisites

Проверьте:

```bash
kubectl top pod
```

Если Resource Metrics API отсутствует, HPA CPU lab не заработает: сначала нужен Metrics Server/совместимый provider.

## 5. HPA

```bash
kubectl apply -f ../../examples/spring-boot/autoscaling/hpa.yaml
kubectl get hpa -w
```

Создайте нагрузку.

Наблюдайте:
- current CPU;
- target;
- desired replicas.

## 6. Измените CPU request

Сравните HPA behavior при разных requests.

Главный вопрос:

> Почему одинаковый actual CPU usage даёт разный utilization percentage?

Потому что utilization вычисляется относительно request.

## 7. Downscale

После остановки нагрузки наблюдайте, что scale down не обязательно происходит мгновенно. Default stabilization window предназначена уменьшать flapping.

## Cleanup

Удалите созданные resources/namespace.
