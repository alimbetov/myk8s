# 06 — CKAD map vs production engineering

Проверено: 2026-09-20.

Цель CKAD-трека проекта — не выучить отдельный набор экзаменационных трюков, а использовать экзамен как **скоростную практику Kubernetes application primitives**, не смешивая её с полной production architecture.

## 1. Экзаменационная модель

На дату проверки CKAD использует performance-based формат и environment Kubernetes 1.35.

Domains:

| Domain | Weight |
|---|---:|
| Application Design and Build | 20% |
| Application Deployment | 20% |
| Application Observability and Maintenance | 15% |
| Application Environment, Configuration and Security | 25% |
| Services and Networking | 20% |

Практический вывод: configuration/security + networking дают почти половину экзамена.

---

## 2. Почему production material шире CKAD

CKAD может попросить:
- Deployment;
- probe;
- ConfigMap;
- Secret;
- Service;
- NetworkPolicy;
- Job;
- volume;
- troubleshooting.

Но реальная production задача дополнительно включает:

```text
Why this value?
What happens during failure?
How does Spring behave?
How is credential rotated?
How does DB survive?
What is RPO/RTO?
How do we observe and rollback?
```

Поэтому каждая глава в `myk8s` имеет два слоя:
- **CKAD speed layer**;
- **production reasoning layer**.

---

## 3. Application Design and Build

Нужно понимать:
- Pods;
- multi-container patterns;
- init containers;
- Jobs/CronJobs;
- volumes;
- container command/args.

### Практика

```bash
kubectl run
kubectl create job
kubectl create cronjob
```

### Production extension

Для Spring:
- когда Deployment;
- когда Spring Batch Job;
- sidecar lifecycle;
- idempotency Job;
- retries/restart policy;
- data persistence.

---

## 4. Application Deployment

Нужно уметь:
- Deployment;
- replicas;
- image update;
- rollout;
- rollback.

```bash
kubectl set image
kubectl scale
kubectl rollout status
kubectl rollout history
kubectl rollout undo
```

### Production extension

- maxSurge/maxUnavailable;
- capacity;
- graceful shutdown;
- DB schema compatibility;
- canary/blue-green;
- SLO during rollout.

---

## 5. Observability and Maintenance

Нужно быстро:

```bash
kubectl get pod
kubectl describe pod
kubectl logs
kubectl logs --previous
kubectl exec
kubectl get events
```

### Практический алгоритм

```text
Pending?
 -> describe/events

CrashLoopBackOff?
 -> logs --previous

Running 0/1?
 -> probe/events

Service broken?
 -> EndpointSlice/DNS
```

### Production extension

- Prometheus metrics;
- centralized logs;
- tracing;
- alerting;
- SLI/SLO;
- incident runbook.

---

## 6. Configuration and Security

Нужно уметь:
- ConfigMap;
- Secret;
- env;
- volume projection;
- ServiceAccount;
- SecurityContext;
- requests/limits.

```bash
kubectl create configmap
kubectl create secret generic
kubectl auth can-i
```

### Production extension

- external secret management;
- credential rotation;
- etcd encryption;
- Pod Security;
- PKI;
- OAuth2;
- least privilege RBAC.

---

## 7. Services and Networking

Нужно понимать:
- Service;
- port/targetPort;
- DNS;
- NetworkPolicy;
- Ingress.

```bash
kubectl expose deployment
kubectl get svc
kubectl get endpointslice
kubectl exec <pod> -- nslookup <service>
```

### Production extension

- Gateway/API ingress architecture;
- TLS;
- mTLS;
- workload identity;
- egress governance;
- application auth.

---

## 8. Что писать без долгого поиска

Нужно довести до автоматизма skeleton:
- Pod;
- Deployment;
- Service;
- ConfigMap/Secret use;
- probes;
- resources;
- PVC;
- Job/CronJob;
- NetworkPolicy;
- SecurityContext.

Не обязательно помнить каждое редкое поле. Нужно быстро узнавать structure и эффективно использовать официальную документацию там, где это разрешено экзаменом.

---

## 9. Практика: один scenario в двух режимах

### Production mode

Сначала понять:

```text
orders Deployment
 -> ConfigMap/Secret
 -> Service
 -> PostgreSQL
 -> probes
 -> security
 -> failure behavior
```

### CKAD mode

Потом повторить на скорость:

```bash
kubectl create deployment orders --image=...
kubectl expose deployment orders --port=8080
kubectl create configmap ...
kubectl create secret generic ...
```

Так экзаменационная скорость строится поверх понимания.

---

## 10. Failure drill

Намеренно:
- wrong image;
- wrong probe;
- wrong selector;
- missing Secret;
- wrong command;
- insufficient resource request scenario.

Для каждого:
1. symptom;
2. shortest diagnostic command;
3. root cause;
4. minimal repair;
5. verification.

---

## 11. Anti-pattern подготовки

- зубрить YAML без понимания controller behavior;
- учить только imperative commands;
- учить только declarative YAML;
- путать Running и Ready;
- после каждой ошибки удалять Pod;
- готовиться на production Helm charts, не умея написать basic primitive;
- переносить экзаменационный minimal manifest в production без hardening.

---

## 12. Target speed sheet

```bash
kubectl get po
kubectl get po -o wide
kubectl describe po <name>
kubectl logs <name>
kubectl logs <name> --previous
kubectl exec -it <name> -- sh
kubectl get events --sort-by=.lastTimestamp

kubectl create deployment
kubectl scale deployment
kubectl set image deployment
kubectl rollout status deployment
kubectl rollout undo deployment

kubectl create configmap
kubectl create secret generic

kubectl expose deployment
kubectl get svc
kubectl get endpointslice

kubectl create job
kubectl create cronjob

kubectl auth can-i
```

---

## 13. Definition of mastery

Тема считается освоенной, когда вы можете:

```text
объяснить mechanism
+
написать minimal manifest
+
написать production variant
+
сломать его
+
диагностировать
+
исправить
+
объяснить operational trade-off
```

Это сильнее простого прохождения CKAD и непосредственно переносится в работу.

---

## Source

- https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/

Проверено: **2026-09-20**.
