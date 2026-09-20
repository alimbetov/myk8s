# Как учиться по myk8s: аналитик, разработчик, тестировщик

Этот учебник не предполагает, что читатель уже является Kubernetes-инженером.

Одна и та же тема рассматривается с трёх сторон:

```text
                 Kubernetes mechanism
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
     Analyst          Developer         Tester
   "зачем и где"    "как реализовать"  "как проверить"
```

## Общая модель обучения

Для каждой главы используйте один и тот же цикл:

```text
1. Понять место механизма во всей системе
2. Понять, кто за него отвечает
3. Понять связи между Kubernetes objects
4. Пройти runtime sequence
5. Прочитать annotated manifest
6. Открыть production-like showcase
7. Сломать один сценарий
8. Диагностировать
9. Объяснить механизм своими словами
```

Не нужно запоминать YAML на первом проходе.

Сначала нужно научиться отвечать:

- зачем существует объект;
- кто его создаёт;
- кто его читает;
- с чем он связан;
- что меняется runtime;
- что увидит пользователь при отказе.

---

# Маршрут аналитика

Аналитику не обязательно уметь писать Deployment с нуля.

Но системный/бизнес-аналитик должен понимать архитектурные последствия требований.

## После учебника аналитик должен уметь читать схему

```text
Internet
   ↓
Gateway
   ↓
Service
   ↓
Spring Boot
   ↓
PostgreSQL

Spring Boot
   ├── Kafka
   ├── RabbitMQ
   └── S3
```

и задавать правильные вопросы:

- сервис внутренний или внешний?
- кто caller?
- какой logical endpoint?
- нужна ли authentication?
- какое expected failure behavior?
- допустим ли stale read?
- что произойдёт при retry?
- можно ли повторить operation?
- что такое successful batch run?
- какие RPO/RTO?
- кто владеет backup/restore?

## Рекомендуемый путь

```text
00 Platform baseline
 ↓
01 Configuration
 ↓
02 Service/DNS/security
 ↓
03 Deployment/probes
 ↓
04 Stateful dependencies
 ↓
11 HTTP integrations
 ↓
12 PostgreSQL
 ↓
14 Service authentication
 ↓
18 External API edge
 ↓
19 Deployment strategies
 ↓
21 Batch
 ↓
22 Observability
```

Остальные главы читать как углубление.

---

# Маршрут разработчика

Разработчику важно увидеть Kubernetes как runtime для Spring Boot.

## Главный contract

```text
Kubernetes provides:
- scheduling
- lifecycle
- service discovery
- configuration delivery
- resource boundaries
- traffic selection

Spring Boot provides:
- application startup
- health semantics
- HTTP/business logic
- dependency clients
- transactions
- graceful shutdown
- application authorization
```

## Разработчик должен уметь объяснить

```text
ConfigMap -> Spring Environment
Secret -> datasource/client
Service -> URL
readiness -> traffic
SIGTERM -> graceful shutdown
resources -> JVM/HPA
replicas -> connection pools
NetworkPolicy -> reachability
JWT -> application identity
```

## Рекомендуемый путь

Читайте строго `00 → 22`, а после каждой главы открывайте связанный showcase и его `annotated.yaml`.

---

# Маршрут тестировщика

Тестировщику Kubernetes особенно полезен как модель failure injection.

Нужно перестать проверять только:

```text
HTTP 200
```

и начать проверять систему:

```text
Pod replacement
Service selection
readiness
DNS
NetworkPolicy
Secret error
dependency outage
retry
rollout
resource pressure
failover
duplicate execution
```

## Базовая тестовая матрица

| Слой | Что сломать | Что наблюдать |
|---|---|---|
| Scheduling | impossible resource request | Pod Pending |
| Image | wrong tag | ImagePullBackOff |
| Application | invalid property | CrashLoopBackOff/startup failure |
| Readiness | broken /readyz | Running, Ready=false |
| Service | wrong selector | empty EndpointSlice |
| Port mapping | wrong targetPort | connection failure |
| DNS | deny DNS egress | UnknownHost |
| Network | deny target egress | connect timeout |
| Authentication | invalid JWT | 401 |
| Authorization | missing role | 403 |
| Secret | wrong DB password | auth/startup/reconnect failure |
| Resources | low memory limit | OOMKilled |
| HPA | missing CPU request | metric/scaling problem |
| Deployment | bad new image | stalled rollout |
| Batch | process exits 1 | Job retries/fails |
| Stateful | delete primary | failover/recovery behavior |

## Рекомендуемый путь

```text
00
 ↓
02 Service/DNS
 ↓
03 Probes/rollouts
 ↓
05 Failure playbook
 ↓
09 Probes
 ↓
10 Shutdown
 ↓
11 HTTP failures
 ↓
15 NetworkPolicy
 ↓
17 Secret rotation
 ↓
19 Release strategies
 ↓
20 HPA
 ↓
21 Jobs
 ↓
22 Observability
```

После этого пройти все labs.

---

# Как читать annotated.yaml

Комментарий рядом с полем отвечает минимум на один вопрос:

```text
WHY
зачем это поле?

MAP
с чем оно связано?

RUNTIME
что оно меняет во время работы?

ATTENTION
где production-риск?

CHECK
как проверить?
```

Пример:

```yaml
resources:
  requests:
    # WHY: scheduler reserve
    # MAP: HPA CPU utilization denominator
    # ATTENTION: изменение request меняет autoscaling semantics
    cpu: 500m
```

---

# Что не нужно делать новичку

Не пытайтесь сразу:
- запоминать все fields;
- учить kubectl flags без mental model;
- копировать production YAML целиком;
- разворачивать PostgreSQL/Kafka вручную через StatefulSet;
- считать Ready и Running одним состоянием;
- считать Secret автоматически безопасным;
- считать NetworkPolicy authentication.

Сначала поймите связи.

---

# Проверка понимания после каждой главы

Попробуйте без текста ответить:

1. **Зачем** существует этот механизм?
2. **Где** он находится в общей архитектуре?
3. **Кто** его исполняет: API server, controller, scheduler, kubelet, application, operator?
4. **С чем** он связан?
5. Какие поля создают эту связь?
6. Что происходит после `kubectl apply`?
7. Что увидит Spring Boot?
8. Что увидит пользователь при failure?
9. Как это проверить?
10. Кто отвечает за исправление: developer, platform, DBA/product operator?

Если ответить трудно — вернитесь к diagram и annotated example, а не к заучиванию API reference.
