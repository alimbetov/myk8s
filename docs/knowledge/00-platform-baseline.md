# 00 — Platform baseline: Spring Boot application in Kubernetes

## Учебная карта темы

### Где Kubernetes находится относительно Spring Boot

```text
Developer
   |
   | builds image
   v
Container Registry
   |
   v
Kubernetes
   |
   +--> Deployment
   |      |
   |      v
   |     Pods
   |
   +--> Service
   |      |
   |      v
   |   stable DNS
   |
   +--> ConfigMap / Secret
   |
   +--> NetworkPolicy
   |
   +--> PVC / external stateful systems
   |
   v
Spring Boot process
```

Главная смена мышления: **мы больше не администрируем конкретный JVM process как “вечный сервер”**. Мы описываем desired state, а Kubernetes постоянно пытается привести cluster к нему.

### Как читать следующие главы

```text
Image
  ↓
Pod
  ↓
Deployment
  ↓
Service/DNS
  ↓
Config/Secret
  ↓
Security
  ↓
Stateful dependencies
  ↓
Operations
```

### Мини-пример

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 2              # desired state: хотим два экземпляра
  template:
    spec:
      containers:
        - name: app
          image: registry/orders:1.0.0
```

Здесь Kubernetes не “запускает приложение один раз”. Controller постоянно проверяет: **есть ли два подходящих Pod?** Если один исчез — создаётся replacement.

> **Смотрите также:** [Showcase 01 — internal REST service](../../showcases/01-internal-rest-service/README.md) → [annotated manifest](../../showcases/01-internal-rest-service/annotated.yaml).

Проверено: 2026-09-20.

Эта глава строит общую mental model. Если понять её, остальные Kubernetes objects перестают выглядеть как набор несвязанных YAML.

## 1. От монолита на VM к Kubernetes

Привычная модель:

```text
VM
 -> systemd
 -> java -jar app.jar
 -> application.properties
 -> database IP
```

Часто разработчик предполагал:
- server живёт долго;
- IP стабилен;
- local filesystem сохраняется;
- restart делается администратором;
- config лежит рядом с JAR.

Kubernetes меняет эти предположения.

```text
Deployment
 -> ReplicaSet
 -> disposable Pods
 -> containers

Service/DNS
 -> stable logical address

ConfigMap/Secret
 -> runtime configuration

PVC
 -> persistent storage when needed

Ingress/Gateway
 -> external entry

NetworkPolicy/RBAC
 -> isolation/permissions

metrics/logs/traces
 -> observability
```

Главное изменение мышления:

> **Не пытайтесь сделать Pod похожим на вечную VM. Проектируйте приложение так, чтобы Pod можно было безопасно заменить.**

---

## 2. Pod

Pod — минимальная schedulable application unit Kubernetes.

Обычно Spring Boot workload:

```text
Pod
 -> one main application container
```

Иногда рядом есть sidecar.

Pod получает:
- IP;
- volumes;
- ServiceAccount identity;
- resource constraints;
- lifecycle/probes.

### Что не считать постоянным

- Pod name;
- Pod IP;
- container filesystem;
- конкретный node.

---

## 3. Deployment

Deployment нужен для stateless replicated application.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 3
```

Это означает не «создать три процесса один раз», а:

> controller должен постоянно стремиться иметь три соответствующие desired state replicas.

Удалите Pod — ReplicaSet создаст replacement.

---

## 4. StatefulSet

StatefulSet нужен, когда workload требует stable identity/storage/order.

```text
postgres-0
postgres-1
postgres-2
```

Но StatefulSet не реализует product replication или backup. Это только Kubernetes primitive.

---

## 5. Job и CronJob

Не каждый Spring Boot application должен быть Deployment.

Если задача должна выполниться и завершиться:

```text
Job
 -> Pod
 -> task
 -> exit 0
```

Периодическая:

```text
CronJob
 -> Jobs по расписанию
```

Например Spring Batch reconciliation/export/cleanup чаще логичнее моделировать как Job/CronJob, а не вечный Deployment с внутренним scheduler.

---

## 6. Service и DNS

Pods меняются, поэтому caller не должен знать их IP.

```text
orders
 -> http://customer-api:8080
 -> Service
 -> Ready Pods
```

Это заменяет привычку прописывать server IP в properties.

---

## 7. ConfigMap и Secret

```text
application image = одинаковый
environment config = снаружи
```

ConfigMap:
- non-sensitive runtime values.

Secret:
- sensitive values, но Secret object сам по себе не заменяет secret-management lifecycle.

Spring получает значения через Environment и связывает их с configuration properties.

---

## 8. Persistent storage

Container filesystem обычно ephemeral.

Если workload должен сохранить data:

```text
Pod -> PVC -> PV -> StorageClass/CSI -> physical storage
```

Для обычного REST backend лучше хранить business data в database/object storage, а не на local Pod disk.

---

## 9. Ingress и egress

### Ingress

```text
external client
 -> LB / Ingress / Gateway
 -> Service
 -> Pod
```

### Egress

```text
Pod
 -> PostgreSQL
 -> Kafka
 -> external API
```

Egress тоже требует architecture: DNS, firewall/NetworkPolicy, TLS, timeout, proxy policy.

---

## 10. Security layers

Нельзя свести security к одному object.

```text
NetworkPolicy = reachability
TLS           = encrypted transport
mTLS          = workload/channel identity
JWT/OAuth2    = application identity
authorization = allowed action
RBAC          = Kubernetes API permission
Secret        = credential material
```

Например NetworkPolicy может разрешить TCP orders→customer, но customer API всё равно может требовать JWT.

---

## 11. Observability

В VM разработчик иногда искал log file на server.

В Kubernetes Pod может исчезнуть.

Поэтому production model:

```text
stdout/stderr -> centralized logs
metrics       -> monitoring
traces        -> distributed request flow
events        -> Kubernetes control-plane symptoms
```

``kubectl logs`` важен для диагностики, но не является полноценной long-term log platform.

---

## 12. Spring Boot production contract

Приложение должно иметь:
- externalized configuration;
- typed config;
- finite dependency timeouts;
- controlled retry;
- health probes;
- graceful shutdown;
- container-aware memory design;
- logs to stdout/stderr;
- no dependency on Pod identity/local disk.

Пример:

```yaml
spring:
  application:
    name: orders-service
  lifecycle:
    timeout-per-shutdown-phase: 20s

management:
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true
```

---

## 13. Minimal Deployment и production difference

Минимум:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 1
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
```

Это достаточно, чтобы понять primitive, но production обычно требует ещё:
- probes;
- resources;
- rollout strategy;
- securityContext;
- ConfigMap/Secret;
- ServiceAccount;
- topology;
- PDB where justified;
- observability.

**CKAD minimal != production complete.**

---

## 14. Что происходит при deploy

```text
kubectl apply
 -> API Server validates/stores desired state
 -> Deployment controller creates ReplicaSet
 -> ReplicaSet creates Pod objects
 -> Scheduler selects nodes
 -> kubelet pulls image
 -> container runtime starts container
 -> probes run
 -> Ready Pod becomes usable by Service
```

Эта sequence помогает понять, где искать failure.

---

## 15. Failure by layer

### Pending
Scheduler/storage issue вероятнее application.

### ImagePullBackOff
Image/registry issue.

### CrashLoopBackOff
Container/application repeatedly crashes.

### Running but NotReady
Application живо, но не допущено к traffic.

### Service exists but no endpoint
Readiness или selector.

### DNS resolves but timeout
Network/port/backend.

### HTTP 401
Authentication.

### HTTP 403
Authorization.

Такой layer-based подход быстрее random restarts.

---

## 16. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| application code/config contract | ✓ |  |  |
| image | ✓ |  | ✓ |
| cluster/CNI/CSI/DNS |  | ✓ |  |
| probes | ✓ |  | ✓ |
| resources |  |  | ✓ |
| Secret infrastructure |  | ✓ | ✓ |
| application authentication | ✓ |  | ✓ |
| NetworkPolicy |  | ✓ | ✓ |
| DB migration | ✓ |  | ✓ |
| observability/SLO |  |  | ✓ |

---

## 17. Практический baseline checklist

Перед тем как считать Spring Boot service Kubernetes-ready:

1. Image immutable.
2. Config externalized.
3. Secret не хранится в Git.
4. Dependency URL использует Service DNS.
5. Есть connect/read timeout.
6. Pool size рассчитан на replicas.
7. startup/readiness/liveness имеют правильную semantics.
8. graceful shutdown протестирован.
9. Local filesystem не используется как permanent storage.
10. Logs уходят в stdout/stderr.
11. ServiceAccount permissions минимальны.
12. Network/auth layers разделены.
13. Rollout совместим с DB schema.
14. Есть troubleshooting commands/runbook.

---

## 18. Hands-on

Базовая лаборатория: [Lab 01](../../labs/01-spring-boot-baseline/README.md).

---

## Sources: для проверки

- https://kubernetes.io/docs/concepts/ — Kubernetes objects/concepts.
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/ — Pod lifecycle.
- https://kubernetes.io/docs/concepts/services-networking/ — Services/networking.
- https://docs.spring.io/spring-boot/reference/ — Spring Boot production configuration.

Проверено: **2026-09-20**.
