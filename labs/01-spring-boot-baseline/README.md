# Lab 01 — Spring Boot Kubernetes baseline

## Цель

Пройти полный путь:

```text
Deployment
 -> Pods
 -> ConfigMap/Secret
 -> probes
 -> Service
 -> EndpointSlice
 -> DNS
 -> controlled failures
 -> diagnosis
 -> repair
```

Лаборатория должна научить не просто выполнить `kubectl apply`, а **объяснить наблюдаемое поведение**.

## Prerequisites

- Kubernetes/k3s cluster;
- `kubectl`;
- отдельный namespace;
- HTTP Spring Boot image с Actuator probes либо собственный image, адаптированный под `/livez` и `/readyz`.

---

## Step 1. Создать namespace

```bash
kubectl create namespace k8s-lab
kubectl config set-context --current --namespace=k8s-lab
```

Проверить:

```bash
kubectl config view --minify
```

**Почему отдельный namespace:** лабораторные NetworkPolicy/Secrets/cleanup не должны затрагивать другие workloads.

---

## Step 2. ConfigMap и runtime Secret

```bash
kubectl apply -f ../../examples/spring-boot/base/configmap.yaml
kubectl apply -f ../../examples/spring-boot/base/serviceaccount.yaml
```

Secret создаём runtime, а не храним real credential в Git:

```bash
kubectl create secret generic spring-app-secret \
  --from-literal=DB_USERNAME=demo \
  --from-literal=DB_PASSWORD=demo
```

Проверить metadata:

```bash
kubectl get configmap
kubectl get secret
```

Не выводите real production secrets в shell history/logs; это lab-only values.

---

## Step 3. Deployment

Сначала адаптируйте image.

```bash
kubectl apply -f ../../examples/spring-boot/base/deployment.yaml
kubectl get deploy,rs,pod -w
```

### Что нужно увидеть и объяснить

```text
Deployment created
 -> ReplicaSet created
 -> Pod created
 -> scheduler selects node
 -> image starts
 -> probes run
 -> Pod becomes Ready
```

Проверить:

```bash
kubectl describe deploy spring-app
kubectl get rs
kubectl describe pod <pod>
```

---

## Step 4. Service

```bash
kubectl apply -f ../../examples/spring-boot/base/service.yaml

kubectl get svc
kubectl get endpointslice \
  -l kubernetes.io/service-name=spring-app
```

**Объясните:** Service существует отдельно от Pod; EndpointSlice связывает logical Service с текущими backends.

---

## Step 5. Проверка изнутри cluster

Debug Pod:

```bash
kubectl run net-debug \
  --image=curlimages/curl \
  -- sleep 3600
```

DNS:

```bash
kubectl exec net-debug -- nslookup spring-app
```

HTTP:

```bash
kubectl exec net-debug -- \
  curl -v http://spring-app:8080/readyz
```

Нужно уметь разделить:
- DNS success;
- TCP connection;
- HTTP status.

---

# Failure 1 — broken readiness

Измените readiness path на:

```text
/broken-readyz
```

```bash
kubectl edit deploy spring-app
kubectl get pod -w
```

Диагностика:

```bash
kubectl describe pod <pod>
kubectl get endpointslice \
  -l kubernetes.io/service-name=spring-app -o yaml
```

### Ожидаем

```text
container Running
Pod NotReady
ready endpoint отсутствует
```

### Главный вывод

**Running != Ready.**

Исправьте path на `/readyz` и проверьте возвращение endpoint.

---

# Failure 2 — missing Secret

```bash
kubectl delete secret spring-app-secret
kubectl rollout restart deploy/spring-app

kubectl get pod
kubectl describe pod <new-pod>
kubectl get events --sort-by=.lastTimestamp
```

Объясните, почему новый Pod не может получить required configuration.

Восстановите:

```bash
kubectl create secret generic spring-app-secret \
  --from-literal=DB_USERNAME=demo \
  --from-literal=DB_PASSWORD=demo

kubectl rollout restart deploy/spring-app
kubectl rollout status deploy/spring-app
```

---

# Failure 3 — Service selector mismatch

Измените selector Service так, чтобы он больше не совпадал с Pod labels.

Проверить:

```bash
kubectl get svc spring-app
kubectl get pod --show-labels
kubectl get endpointslice \
  -l kubernetes.io/service-name=spring-app
```

### Что важно заметить

DNS имя `spring-app` всё ещё может существовать, но ready backend исчез.

Это показывает различие:

```text
DNS record exists
!=
Service has usable endpoints
```

Исправьте selector.

---

# Failure 4 — broken image rollout

```bash
kubectl set image deploy/spring-app \
  app=example.invalid/spring-app:no-such-tag

kubectl rollout status deploy/spring-app
```

В другом terminal:

```bash
kubectl get rs
kubectl get pod
kubectl describe pod <new-pod>
```

Найдите `ImagePullBackOff`.

Исправить:

```bash
kubectl rollout undo deploy/spring-app
kubectl rollout status deploy/spring-app
```

Объясните, почему rollback Deployment не означает rollback database.

---

# Failure 5 — удалить Pod вручную

```bash
kubectl get pod
kubectl delete pod <pod>
kubectl get pod -w
```

Новый Pod появляется.

Объяснение: вы удалили instance, но не desired state Deployment.

---

## CKAD speed round

Повторите без подробного чтения:

```bash
kubectl get deploy,rs,pod,svc
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get endpointslice
kubectl rollout status deploy/spring-app
kubectl rollout history deploy/spring-app
```

Цель — узнавать слой отказа быстро.

---

## Production questions после лаборатории

Вы должны уметь ответить:

1. Кто пересоздал удалённый Pod?
2. Почему Running Pod может не получать traffic?
3. Где хранится stable Service identity?
4. Почему Pod IP нельзя хранить в properties?
5. Почему Secret update через env требует restart/reload?
6. Почему liveness нельзя привязать к PostgreSQL availability?
7. Почему broken rollout не обязательно сразу убивает старые replicas?
8. Почему `rollout undo` не откатывает Liquibase?
9. Как отличить DNS failure от NetworkPolicy failure?
10. Какие данные потеряются при удалении Pod, если они записаны только в container filesystem?

---

## Cleanup

```bash
kubectl delete namespace k8s-lab
```

Проверить:

```bash
kubectl get namespace k8s-lab
```
