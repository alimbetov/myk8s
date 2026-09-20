# 05 — Day-2 operations and failure playbook

## Учебная карта темы

### Как искать проблему сверху вниз

```text
Desired state correct?
       ↓
Pod scheduled?
       ↓
Image/container started?
       ↓
Spring startup succeeded?
       ↓
Probes healthy?
       ↓
Service has endpoints?
       ↓
DNS/network works?
       ↓
Dependency reachable?
       ↓
Business operation works?
```

Это важнее команды `kubectl delete pod`.

### Симптом -> слой

```text
Pending            -> scheduling / PVC / affinity
ImagePullBackOff   -> image / registry
CrashLoopBackOff   -> process startup/runtime
Running NotReady   -> readiness/application dependency
DNS error          -> Service/DNS/NetworkPolicy
connect timeout    -> network / targetPort / dependency
401                -> authentication
403                -> authorization
```

### Базовый incident loop

```bash
kubectl get pod -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl get svc,endpointslice
kubectl get events --sort-by=.lastTimestamp
```

> Сначала сохраняйте evidence, потом restart.

## Для аналитика, разработчика и тестировщика

### Аналитик

Использует failure taxonomy для non-functional requirements и runbooks: какой слой отказал, какой user impact, кто owner и какой recovery target.

### Разработчик

Должен давать платформе диагностируемое приложение:
- structured logs;
- health endpoints;
- metrics;
- meaningful exit/startup failures;
- finite timeouts.

### Тестировщик

Здесь его ключевая роль — intentionally ломать систему:
- wrong selector;
- wrong Secret;
- DNS deny;
- resource shortage;
- bad image;
- dependency outage;
- failed rollout.

### Перед следующей главой

Вы должны уметь начать incident не с restart, а с вопроса:

> На каком слое впервые появилось расхождение между desired и actual behavior?

Проверено: 2026-09-20.

Day-1 — «мы смогли deploy». Day-2 — **как система живёт месяцами после deploy**: failures, upgrades, rotations, drains, incidents, restore и troubleshooting.

## 1. Главный принцип troubleshooting

Не начинайте с:

```bash
kubectl delete pod
```

Restart может временно скрыть symptom и уничтожить полезный evidence.

Начинайте с определения слоя отказа.

```text
1 desired state
2 scheduling
3 image/container
4 application startup
5 probes
6 Service/EndpointSlice
7 DNS/network
8 dependency
9 storage
10 rollout/schema compatibility
```

---

## 2. Базовая цепочка диагностики

```bash
kubectl get deploy,rs,pod -o wide
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
kubectl logs <pod> --all-containers
kubectl logs <pod> --previous
```

### Почему такой порядок

``get`` показывает текущее состояние.

``describe`` показывает conditions/events/configuration.

``events`` показывает scheduler/kubelet/controller failures.

``logs`` показывает application.

``logs --previous`` особенно важен после CrashLoopBackOff: текущий container уже новый, а причина предыдущего crash может быть только в previous logs.

---

## 3. Pod Pending

```text
Pod exists
STATUS Pending
```

Это часто не application bug: container ещё мог вообще не стартовать.

Проверить:

```bash
kubectl describe pod <pod>
```

Причины:
- insufficient CPU/memory;
- PVC not bound;
- nodeSelector/affinity impossible;
- taint without toleration;
- scheduling constraints.

**Практика:** если Pod Pending, сначала смотрите scheduler events, а не Spring logs.

---

## 4. ImagePullBackOff

```text
Pod scheduled
container image cannot be pulled
```

Проверить:

```bash
kubectl describe pod <pod>
```

Причины:
- wrong image/tag;
- registry unavailable;
- imagePullSecret;
- registry authorization;
- DNS/network egress.

---

## 5. CrashLoopBackOff

Это не root cause. Это Kubernetes backoff state после повторяющихся crashes.

```bash
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

Spring причины:
- missing env;
- DB auth failure;
- Liquibase error;
- port/config error;
- OutOfMemory;
- certificate failure.

---

## 6. Running, но service недоступен

Проверяйте:

```bash
kubectl get pod
kubectl get svc
kubectl get endpointslice
```

Возможная цепочка:

```text
Pod Running
readiness false
EndpointSlice has no ready endpoint
Service has nowhere to send traffic
```

То есть restart Service бессмысленен: Service — virtual abstraction, а проблема может быть в readiness приложения.

---

## 7. DNS failure

Spring symptom:

```text
UnknownHostException: customer-api
```

Диагностика:

```bash
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl exec <pod> -- nslookup customer-api
kubectl get svc customer-api
```

Если default-deny egress включён, отдельно проверить DNS allow policy.

---

## 8. Dependency unavailable

Допустим DNS работает, TCP connection timeout.

Нужно различить:
- dependency Pods down;
- Service no endpoints;
- NetworkPolicy;
- dependency overload;
- wrong port;
- TLS handshake.

### Spring side

Metrics должны показывать:
- request latency;
- timeouts;
- pool saturation;
- retry count;
- circuit breaker state, если используется.

**Практика:** dependency outage не должен автоматически превращать liveness в failure.

---

## 9. Wrong Secret

Симптом:
- authentication error;
- application startup failure;
- repeated reconnect.

Проверить:
- Secret существует;
- key name правильный;
- Deployment ссылается на правильный Secret;
- Pod перезапущен после env-based rotation.

Не печатать Secret value в logs.

---

## 10. Expired certificate

Симптомы:
- TLS handshake error;
- PKIX validation;
- certificate expired/not yet valid;
- hostname mismatch.

Диагностика должна определить:
- client или server certificate;
- expiry;
- trust chain;
- SAN/hostname;
- rotation source.

Restart без нового certificate проблему не решает.

---

## 11. Network denied

Полезная модель:

```text
DNS resolves?
  no -> DNS layer

yes
  |
TCP connects?
  no -> NetworkPolicy/CNI/port

yes
  |
TLS succeeds?
  no -> certificate/PKI

yes
  |
HTTP 401?
  -> authentication

HTTP 403?
  -> authorization
```

Это намного эффективнее, чем сразу менять random YAML.

---

## 12. Node drain

Подготовка:

```bash
kubectl get pod -o wide
kubectl get pdb
kubectl cordon <node>
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

### Что происходит

``cordon`` запрещает новые normal Pods на node.

``drain`` пытается корректно evict workloads.

Перед drain проверить:
- PDB;
- replica count;
- stateful operator status;
- local storage;
- topology;
- capacity других nodes.

---

## 13. Почему drain может зависнуть

Например:
- Deployment имеет 1 replica;
- PDB требует minAvailable=1;
- eviction единственного Pod нарушит budget.

Это не «Kubernetes сломался». Политика availability противоречит операции maintenance.

---

## 14. Disk pressure / full disk

Сценарий:

```text
PVC 100%
 -> DB cannot write
 -> application sees SQL errors

node filesystem 100%
 -> kubelet/node pressure
 -> image/log/runtime problems
```

Это разные storage layers.

Проверять:
- PVC metrics;
- node disk;
- logs;
- events;
- retention;
- expansion capability.

Restart не освобождает persistent storage автоматически.

---

## 15. Secret rotation runbook

```text
1 issue/create new credential
2 dependency accepts both/new credential
3 update Secret
4 restart/reload workload
5 verify new connections
6 revoke old credential
7 audit
```

Если dependency не умеет overlap credentials, нужен другой controlled cutover design.

---

## 16. Rollout monitoring

```bash
kubectl rollout status deployment/orders
kubectl rollout history deployment/orders
kubectl get rs
kubectl get pod -w
```

Следить нужно не только за «Pods Running», но и:
- readiness;
- error rate;
- latency;
- DB errors;
- queue lag;
- business KPIs.

Technically successful Deployment может быть functionally broken.

---

## 17. Rollback

```bash
kubectl rollout undo deployment/orders
```

Полезно при bad image/config template, но не откатывает:
- DB migration;
- external data;
- message schema;
- credentials revoked outside cluster.

Поэтому release runbook должен иметь application + data rollback/forward strategy.

---

## 18. Observability minimum

### RED для HTTP

- Rate;
- Errors;
- Duration.

### JVM

- heap/non-heap;
- GC;
- threads;
- CPU;
- process/container memory.

### Dependencies

- DB pool active/pending;
- HTTP client latency/errors;
- Kafka consumer lag;
- Rabbit queue depth;
- storage capacity.

### Kubernetes

- restart count;
- OOMKilled;
- Pending;
- unavailable replicas;
- probe failures;
- node pressure.

---

## 19. Logs

Production logs желательно писать в stdout/stderr.

Полезные поля:
- timestamp;
- service;
- version;
- trace/correlation ID;
- request path;
- safe business identifier;
- error class.

Не писать:
- password;
- access token;
- authorization header;
- private key;
- full sensitive payload.

---

## 20. Failure scenario matrix

| Failure | Что увидим | Первая проверка | Тип исправления |
|---|---|---|---|
| Pod crash | CrashLoopBackOff | previous logs | app/config |
| readiness | Running 0/1 | describe + EndpointSlice | app/dependency |
| DNS | UnknownHost | nslookup | DNS/policy |
| wrong Secret | auth error | refs + logs | rotation/config |
| expired cert | TLS error | cert chain/expiry | PKI |
| network denied | timeout | policy/connectivity | NetworkPolicy |
| node drain | eviction blocked | PDB | availability design |
| disk full | write errors | storage metrics | capacity |
| DB incompatibility | old/new Pods errors | logs + migration | schema evolution |

---

## 21. Практический incident drill

### Шаг 1: deploy healthy app

```bash
kubectl rollout status deploy/orders
```

### Шаг 2: сломать readiness

Изменить path на несуществующий.

### Шаг 3: диагностировать без restart

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice
kubectl logs <pod>
```

### Шаг 4: сформулировать root cause

Не «Pod broken», а:

```text
readiness endpoint returns 404,
therefore Pods are Running but NotReady,
therefore Service has no ready backends.
```

### Шаг 5: исправить и verify

Проверить:
- Pod Ready;
- EndpointSlice populated;
- HTTP works;
- rollout stable.

Это и есть production troubleshooting discipline.

---

## 22. Developer vs Platform

| Incident area | Developer | Platform | Shared |
|---|---:|---:|---:|
| Spring startup | ✓ |  |  |
| probe semantics | ✓ |  | ✓ |
| scheduler/node |  | ✓ |  |
| CNI/DNS |  | ✓ |  |
| dependency timeout/retry | ✓ |  | ✓ |
| storage |  | ✓ | ✓ |
| certificate lifecycle |  | ✓ | ✓ |
| DB migration | ✓ |  | ✓ |
| incident runbook |  |  | ✓ |

---

## 23. CKAD

Нужно быстро:

```bash
kubectl get
kubectl describe
kubectl logs
kubectl logs --previous
kubectl exec
kubectl get events
kubectl rollout status
kubectl rollout undo
```

CKAD проверяет оперативную диагностику primitives; production добавляет metrics, tracing, SLO, incident process, DR и real stateful recovery.

---

## Sources: для проверки

- https://kubernetes.io/docs/tasks/debug/debug-application/ — application troubleshooting.
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/ — Pod/container/probe lifecycle.
- https://kubernetes.io/docs/concepts/workloads/pods/disruptions/ — drain/eviction/PDB.
- https://kubernetes.io/docs/concepts/storage/ — storage concepts.

Проверено: **2026-09-20**.
