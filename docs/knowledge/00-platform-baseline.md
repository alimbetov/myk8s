# 00 — Platform baseline: Spring Boot application in Kubernetes

Проверено: 2026-09-20.

## 1. Mental model

Kubernetes не «запускает jar». Он поддерживает желаемое состояние набора API-объектов.

Для обычного Spring Boot backend:

```text
Deployment
  -> ReplicaSet
    -> Pod
      -> container(java -jar app.jar)

Service
  -> EndpointSlice
    -> Ready Pods

ConfigMap/Secret
  -> env/files
    -> Spring Environment

Ingress/Gateway
  -> Service
    -> Pod

PVC
  -> PV/StorageClass
    -> mounted storage
```

Pod — расходная единица. Нельзя проектировать приложение так, будто имя Pod, IP Pod или локальная файловая система долговечны.

## 2. Spring Boot view

Базовый production contract:

```yaml
spring:
  application:
    name: orders-service
  lifecycle:
    timeout-per-shutdown-phase: 20s

server:
  shutdown: graceful

management:
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true
  endpoints:
    web:
      exposure:
        include: health,prometheus
```

Приложение должно:
- брать environment-specific values извне image;
- иметь конечные connect/read/request timeouts;
- не делать external dependency частью liveness;
- завершаться gracefully;
- писать логи в stdout/stderr;
- не хранить runtime-state на container filesystem.

## 3. Kubernetes view

Минимум:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 2
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
          image: registry.example/orders:1.0.0
          ports:
            - containerPort: 8080
```

Production добавляет requests/limits, probes, securityContext, ServiceAccount, topology rules, rollout settings, termination grace period, ConfigMap/Secret refs и observability.

## 4. Security layers

Не смешивать:

1. **NetworkPolicy** — кто может установить L3/L4 соединение.
2. **TLS/mTLS** — защита канала и, при mTLS, machine identity.
3. **Application authentication** — JWT/OAuth2/session/API key.
4. **Authorization** — что authenticated principal имеет право сделать.
5. **RBAC Kubernetes** — доступ workload/operator к Kubernetes API.

Открытый TCP 8080 внутри cluster network не означает, что endpoint должен доверять любому caller.

## 5. Stateful implications

Stateless Spring Boot deployment должен считать Pod disposable. PostgreSQL/Kafka/RabbitMQ/object storage — отдельные stateful systems с собственными replication/quorum/backup/upgrade semantics.

## 6. Day-2

Базовые команды:

```bash
kubectl get deploy,rs,pod,svc
kubectl describe pod <pod>
kubectl logs <pod> --all-containers
kubectl logs <pod> --previous
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status deploy/orders
kubectl rollout history deploy/orders
kubectl rollout undo deploy/orders
```

## 7. Failure scenarios

- Pod crash: controller создаёт/перезапускает workload, но причина остаётся в logs/events.
- Readiness failed: Pod жив, но Service перестаёт направлять к нему traffic.
- DNS failure: приложение может быть healthy, но dependency calls ломаются.
- Wrong Secret: обычно startup failure или authentication errors.
- Node drain: реплики должны пережить voluntary disruption.
- Incompatible DB migration: Kubernetes rollback приложения не откатывает schema автоматически.

## 8. Responsibility

| Область | Developer | Platform | Shared |
|---|---|---|---|
| application config contract | ✓ |  |  |
| image/runtime | ✓ |  | ✓ |
| cluster/DNS/CNI/CSI |  | ✓ |  |
| probes | ✓ |  | ✓ |
| RBAC/NetworkPolicy |  | ✓ | ✓ |
| secrets lifecycle |  | ✓ | ✓ |
| DB migration compatibility | ✓ |  | ✓ |
| SLO/alerts |  |  | ✓ |

## 9. CKAD

CKAD проверяет application primitives и troubleshooting; production дополнительно требует HA, secret lifecycle, TLS PKI, operators, backup/restore, SLO и capacity planning.

## 10. Lab

Начать с [Lab 01](../../labs/01-spring-boot-baseline/README.md).

## Sources

- https://kubernetes.io/docs/concepts/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/concepts/services-networking/
- https://docs.spring.io/spring-boot/reference/actuator/endpoints.html

Проверено: **2026-09-20**.
