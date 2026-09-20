# 19 — Deployment strategies

Проверено: 2026-09-20.

Цель главы — понять, как доставлять новую версию Spring Boot приложения без ненужного downtime и как выбирать между RollingUpdate, Recreate, blue/green и canary.

## 1. Default Kubernetes strategy

Для Deployment default strategy — RollingUpdate.

```yaml
strategy:
  type: RollingUpdate
```

По умолчанию:
- `maxUnavailable: 25%`;
- `maxSurge: 25%`.

Это не всегда подходит конкретному SLO.

## 2. RollingUpdate

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Для 3 replicas процесс примерно:

```text
3 old ready
 -> create 1 new
 -> wait Ready
 -> remove 1 old
 -> repeat
```

Плюс: простой native Kubernetes mechanism.

Минус: во время rollout одновременно живут две версии.

Следствие: API/DB/message compatibility обязательна.

## 3. Resource effect of surge

Deployment docs отмечают важный нюанс: terminating Pods могут временно приводить к фактическому resource consumption выше ожидаемого `replicas + maxSurge`, пока не закончится termination grace period.

Поэтому capacity нужно считать с запасом.

## 4. Recreate

```yaml
strategy:
  type: Recreate
```

```text
stop old
 -> start new
```

Downtime ожидаем.

Подходит:
- lab/dev;
- legacy exclusive workload;
- редкие случаи несовместимости двух versions.

Не default для user-facing REST API.

## 5. Blue/Green

Две полные environments:

```text
blue  = current
green = candidate
```

Traffic switch:

```text
Service selector / Gateway route
blue -> green
```

Плюсы:
- быстрый switch/rollback;
- candidate можно проверить отдельно.

Минусы:
- почти двойная capacity;
- DB всё равно shared/needs compatibility;
- session/cache/external side effects усложняют rollback.

## 6. Canary

Новая версия получает малую часть traffic:

```text
95% -> v1
 5% -> v2
```

Проверяются:
- error rate;
- latency;
- business metrics.

Потом weight увеличивается.

Native Deployment сам по себе не даёт полноценный weighted traffic canary; обычно нужен Gateway/Ingress/service mesh/progressive delivery controller.

## 7. DB schema — главный hidden risk

Rolling/canary/blue-green означают coexistence версий.

Поэтому migration:

```text
expand
 -> compatible deploy
 -> migrate/backfill
 -> contract later
```

Плохой deploy:
- v2 удаляет column;
- v1 ещё работает;
- v1 SQL падает.

## 8. Message compatibility

То же относится к Kafka/Rabbit:
- новый producer не должен сразу публиковать schema, которую old consumer не понимает;
- consumer evolution должна учитывать mixed versions.

## 9. Sessions

Если server-side session хранится local memory, rollout может ломать user session.

Лучше:
- stateless JWT where appropriate;
- external session store;
- deliberate sticky-session design.

## 10. Readiness gate

Новый Pod не должен получать traffic до readiness.

```text
container started
 !=
ready for traffic
```

Это основа safe rollout.

## 11. minReadySeconds

Помогает не считать мгновенно unstable Pod available.

```yaml
minReadySeconds: 10
```

## 12. progressDeadlineSeconds

```yaml
progressDeadlineSeconds: 600
```

Если rollout не progresses, Deployment condition сигнализирует проблему.

CI/CD должен отслеживать `kubectl rollout status`.

## 13. Rollback

```bash
kubectl rollout history deploy/orders
kubectl rollout undo deploy/orders
```

Rollback Deployment не откатывает:
- DB schema/data;
- external APIs;
- emitted events;
- revoked secrets.

## 14. Failure practice: bad image

```bash
kubectl set image deploy/orders   app=registry/orders:not-found
kubectl rollout status deploy/orders
kubectl describe pod <new-pod>
```

Old Pods при подходящей RollingUpdate могут остаться available.

## 15. Failure practice: new version never Ready

Image стартует, readiness fails.

Observe:

```bash
kubectl get rs,pod
kubectl rollout status deploy/orders
kubectl get endpointslice
```

Deployment не должен слепо заменить healthy capacity broken Pods, если strategy настроена разумно.

## 16. Failure practice: schema incompatible

Deploy v2 с breaking migration. Смотрите logs и обе ReplicaSets.

Главный вывод: Kubernetes rollout healthy, application contract unhealthy.

## 17. Choosing strategy

| Scenario | Candidate |
|---|---|
| обычный stateless API | RollingUpdate |
| downtime допустим / exclusive resource | Recreate |
| нужен быстрый switch + capacity есть | Blue/Green |
| нужен постепенный risk exposure | Canary |

Это starting point, не автоматический verdict.

## 18. Deployment pipeline

Хороший pipeline:

```text
build
 -> test
 -> scan
 -> publish immutable image
 -> migrate safely
 -> deploy
 -> rollout status
 -> smoke test
 -> observe
 -> promote/finish
```

## 19. Anti-patterns

- latest tag;
- deploy без readiness;
- breaking DB change;
- rollback plan только `kubectl undo`;
- canary без metrics;
- maxUnavailable copied blindly;
- insufficient surge capacity.

## 20. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| schema/API compatibility | ✓ | | ✓ |
| rollout parameters | | | ✓ |
| delivery controller | | ✓ | |
| smoke/business checks | ✓ | | ✓ |
| rollback plan | | | ✓ |

## 21. CKAD

RollingUpdate, image update, rollout history/undo — CKAD. Blue/green/canary/progressive delivery — production extension.

## Sources

- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/

Проверено: **2026-09-20**.
