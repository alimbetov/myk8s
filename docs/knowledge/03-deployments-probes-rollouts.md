# 03 — Deployments, probes, resources and rollouts

Проверено: 2026-09-20.

## Spring Boot readiness/liveness

Spring Boot Actuator предоставляет:
- `/actuator/health/liveness`;
- `/actuator/health/readiness`;
- при `management.endpoint.health.probes.add-additional-paths=true`: `/livez`, `/readyz` на main server port.

Критическое правило: **liveness не должна зависеть от БД/Kafka/внешнего API**. Иначе outage зависимости может перезапустить все Pods и усилить cascading failure.

## Production Deployment baseline

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
            httpGet: { path: /livez, port: http }
            failureThreshold: 30
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /livez, port: http }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /readyz, port: http }
            periodSeconds: 5
            failureThreshold: 2
```

## Resources

Requests влияют на scheduling и QoS. Memory limit приводит к OOM kill при превышении. CPU limit может вызвать throttling; его нельзя копировать механически из шаблона без measurements.

Heap sizing JVM должен учитывать container memory. Оставляйте headroom для metaspace, thread stacks, direct buffers, native libs и sidecars.

## Rollout

```bash
kubectl set image deploy/orders app=registry.example/orders:1.4.3
kubectl rollout status deploy/orders
kubectl rollout history deploy/orders
kubectl rollout undo deploy/orders
```

Rollback image не гарантирует rollback DB schema. Database migrations проектируются backward-compatible: expand -> migrate -> contract.

## Graceful shutdown

Kubernetes отправляет SIGTERM и ждёт `terminationGracePeriodSeconds`. Spring Boot должен перестать принимать новую работу и завершить in-flight operations раньше SIGKILL.

Проверять:
- HTTP requests;
- Kafka/Rabbit consumers;
- scheduled jobs;
- connection pool shutdown;
- executor termination.

## PDB

PDB защищает availability при voluntary disruptions (например, drain), но не является защитой от всех failures и не управляет rollout Deployment.

## Sources

- https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

Проверено: **2026-09-20**.
