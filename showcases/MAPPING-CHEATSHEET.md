# Manifest Mapping Cheat Sheet

Проверено: 2026-09-20.

Эта страница показывает **какие поля разных Kubernetes manifests реально связаны между собой**.

---

## 1. Deployment selector -> Pod labels

```yaml
# Deployment
spec:
  selector:
    matchLabels:
      app: orders

  template:
    metadata:
      labels:
        app: orders
```

Связь должна совпадать:

```text
Deployment.spec.selector.matchLabels
                  =
Deployment.spec.template.metadata.labels
```

Если не совпадает при creation, Deployment validation fails.

---

## 2. Service selector -> Pod labels

```yaml
# Service
spec:
  selector:
    app: orders
```

```yaml
# Deployment Pod template
metadata:
  labels:
    app: orders
```

```text
Service.selector
       ↓
Pod labels
       ↓
EndpointSlice
```

Это **label mapping**, а не mapping по имени Deployment.

Service:

```text
orders-service
```

может выбирать Pods Deployment:

```text
orders-backend-v2
```

если labels совпадают.

---

## 3. Service targetPort -> container port

Named mapping:

```yaml
# Deployment
ports:
  - name: http
    containerPort: 8080
```

```yaml
# Service
ports:
  - port: 80
    targetPort: http
```

```text
caller -> Service:80
              |
              | targetPort=http
              v
         Pod:8080
```

`Service.port` и `containerPort` не обязаны быть одинаковыми.

---

## 4. Ingress backend -> Service

```yaml
backend:
  service:
    name: orders
    port:
      name: http
```

должно соответствовать:

```yaml
kind: Service
metadata:
  name: orders
spec:
  ports:
    - name: http
```

Здесь mapping по **object name + Service port name/number**.

---

## 5. HPA -> Deployment

```yaml
scaleTargetRef:
  apiVersion: apps/v1
  kind: Deployment
  name: orders
```

Здесь exact object reference.

---

## 6. HPA utilization -> Deployment requests

```yaml
# Deployment
resources:
  requests:
    cpu: 500m
```

```yaml
# HPA
target:
  type: Utilization
  averageUtilization: 70
```

Logical/runtime mapping:

```text
actual CPU
     /
CPU request
     =
utilization %
```

Это не name reference, но configuration одного manifest определяет semantics другого.

---

## 7. ConfigMap -> envFrom -> Spring property

```yaml
# ConfigMap
data:
  PAYMENT_API_URL: http://payment-api:8080
```

```yaml
# Deployment
envFrom:
  - configMapRef:
      name: orders-config
```

```yaml
# Spring application.yaml
app:
  payment-api:
    base-url: ${PAYMENT_API_URL}
```

Mapping:

```text
ConfigMap key
 -> environment variable
 -> Spring Environment
 -> application property
 -> @ConfigurationProperties
```

---

## 8. Secret -> explicit env mapping

Secret:

```yaml
stringData:
  password: ...
```

Deployment:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: orders-db
        key: password
```

Spring:

```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}
```

Здесь Kubernetes key `password` и process env `DB_PASSWORD` могут называться по-разному.

---

## 9. Volume -> volumeMount

```yaml
volumes:
  - name: work
    persistentVolumeClaim:
      claimName: work-data
```

```yaml
volumeMounts:
  - name: work
    mountPath: /data/work
```

Обязательная связь:

```text
volumeMount.name
       =
volume.name
```

А затем:

```text
volume.persistentVolumeClaim.claimName
       =
PVC.metadata.name
```

---

## 10. Pod -> ServiceAccount

```yaml
spec:
  serviceAccountName: orders
```

ссылается на:

```yaml
kind: ServiceAccount
metadata:
  name: orders
```

---

## 11. RoleBinding -> ServiceAccount + Role

```yaml
subjects:
  - kind: ServiceAccount
    name: orders
roleRef:
  kind: Role
  name: orders-reader
```

Это identity-to-permission mapping.

---

## 12. NetworkPolicy -> Pod labels

Target:

```yaml
podSelector:
  matchLabels:
    app: payment-api
```

Caller:

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: orders-api
```

NetworkPolicy не ссылается на Service name. Она выбирает Pods по labels/IP/namespaces.

---

## 13. Probe -> named container port

```yaml
ports:
  - name: http
    containerPort: 8080

readinessProbe:
  httpGet:
    path: /readyz
    port: http
```

Named port обеспечивает internal mapping внутри Pod spec.

---

## 14. Readiness -> EndpointSlice

Это runtime mapping, а не YAML reference:

```text
readinessProbe succeeds
 -> Pod Ready=True
 -> EndpointSlice ready endpoint
 -> Service can route normal traffic
```

---

## 15. CronJob -> Job -> Pod template

```text
CronJob.spec.jobTemplate
      ↓
new Job.spec
      ↓
Job.spec.template
      ↓
Pod
```

Поэтому retry settings Job находятся внутри `jobTemplate.spec`, а schedule/concurrency — на CronJob.

---

## 16. Ingress TLS -> Secret

```yaml
tls:
  - secretName: api-tls
```

ссылается на:

```yaml
kind: Secret
metadata:
  name: api-tls
type: kubernetes.io/tls
```

---

## 17. PVC -> StorageClass

```yaml
storageClassName: fast-csi
```

ссылается на:

```yaml
kind: StorageClass
metadata:
  name: fast-csi
```

Если `storageClassName` omitted, cluster default StorageClass может примениться.

---

# Mapping types

Полезно различать четыре типа связи.

### A. Exact object reference

```text
HPA -> Deployment name
Pod -> ServiceAccount name
Ingress -> Service name
PVC -> StorageClass name
```

### B. Label selector

```text
Service -> Pods
NetworkPolicy -> Pods
Deployment -> Pods
```

### C. Named port mapping

```text
Service.targetPort -> container port name
probe.port -> container port name
Ingress backend port -> Service port name
```

### D. Runtime/application configuration

```text
ConfigMap -> env -> Spring property
Secret -> env/file -> Spring property
readiness -> EndpointSlice readiness
requests.cpu -> HPA utilization
replicas × connection pool -> DB capacity
```

Именно тип связи определяет, **что нужно диагностировать**, когда два объекта «не видят» друг друга.
