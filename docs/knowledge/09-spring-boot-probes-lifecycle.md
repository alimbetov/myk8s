# 09 — Spring Boot probes, ApplicationAvailability и lifecycle

Проверено: 2026-09-20.

Эта глава отвечает на вопрос:

> Kubernetes умеет перезапускать container и убирать Pod из Service traffic. **Но как kubelet понимает, что Spring Boot ещё запускается, что process жив и что application реально готов принимать traffic?**

Главная mental model:

```text
JVM start
  ↓
SpringApplication
  ↓
ApplicationContext
  ↓
ApplicationAvailability
  ├── LivenessState
  └── ReadinessState
  ↓
Actuator health groups
  ↓
/livez /readyz
  ↓
kubelet probes
  ↓
Pod conditions
  ↓
EndpointSlice
  ↓
Service traffic
```

Probe — это не “URL для Kubernetes”. Это **контракт между application lifecycle и platform action**.

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь объяснить:

1. Чем startup, liveness и readiness отличаются по смыслу.
2. Что такое Spring Boot `ApplicationAvailability`.
3. Что такое `LivenessState` и `ReadinessState`.
4. Как Actuator health groups связаны с этими состояниями.
5. Как kubelet использует probe result.
6. Почему startupProbe блокирует liveness/readiness до первого успеха.
7. Почему readiness failure не перезапускает container.
8. Как readiness влияет на Pod Ready condition.
9. Как Ready condition влияет на EndpointSlice.
10. Почему Running != Ready.
11. Почему liveness не должна blindly зависеть от DB/Kafka/external API.
12. Когда dependency может быть частью readiness.
13. Почему отдельный management port может дать false-positive health.
14. Зачем `management.endpoint.health.probes.add-additional-paths=true`.
15. Как выбирать probe timeout/period/failureThreshold.
16. Почему один endpoint для всех probes часто плох.
17. Чем health check отличается от monitoring.
18. Как custom health indicator может навредить.
19. Как диагностировать slow startup.
20. Как диагностировать restart loop из-за liveness.
21. Как диагностировать Running/NotReady.
22. Как тестировать dependency outage без restart storm.
23. Как probe semantics связаны с rollout.
24. Что должен знать analyst/developer/tester/platform engineer.

---

# 2. Вся state machine одной картинкой

```text
container process starts
      ↓
JVM starts
      ↓
Spring Boot starts
      ↓
ApplicationContext builds
      ↓
LivenessState becomes CORRECT
      ↓
web server / runners / initialization
      ↓
ReadinessState becomes ACCEPTING_TRAFFIC
      ↓
startupProbe succeeds
      ↓
readinessProbe returns success
      ↓
Pod Ready=True
      ↓
EndpointSlice endpoint ready=true
      ↓
Service can route traffic
```

А при runtime degradation:

```text
application still alive
      ↓
readiness false
      ↓
Pod Ready=False
      ↓
EndpointSlice not ready
      ↓
traffic removed

BUT

container not restarted
```

---

# 3. Spring Boot lifecycle

Упрощённо:

```text
java main
  ↓
SpringApplication.run
  ↓
Environment prepared
  ↓
ApplicationContext created
  ↓
beans instantiated
  ↓
web server started
  ↓
ApplicationRunner / CommandLineRunner
  ↓
application ready
```

Probe config должна соответствовать **реальному startup behavior**, а не условному “обычно стартуем за 30 секунд”.

---

# 4. ApplicationAvailability

Spring Boot предоставляет abstraction:

```text
ApplicationAvailability
```

Через неё application может читать текущие availability states.

Conceptually:

```text
ApplicationAvailability
  |
  +--> LivenessState
  |
  +--> ReadinessState
```

Это application-level model состояния, которую Actuator может expose через health groups.

---

# 5. LivenessState

Liveness отвечает на вопрос:

> Внутреннее состояние application достаточно здорово, чтобы process мог продолжать работать?

Typical values:

```text
CORRECT
BROKEN
```

Если application стала необратимо broken:

```text
LivenessState=BROKEN
  ↓
liveness health group DOWN
  ↓
livenessProbe fails
  ↓
kubelet restarts container
```

---

# 6. ReadinessState

Readiness отвечает:

> Application сейчас готово принимать новый traffic?

Typical values:

```text
ACCEPTING_TRAFFIC
REFUSING_TRAFFIC
```

Если:

```text
ReadinessState=REFUSING_TRAFFIC
```

то правильное действие обычно:

```text
remove from traffic
not restart
```

---

# 7. Actuator dependency

Maven:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Без Actuator можно реализовать свои endpoints, но для Spring Boot Actuator даёт стандартный integration path.

---

# 8. Probe health groups

Spring Boot exposes dedicated liveness/readiness health groups.

Typical endpoints:

```text
/actuator/health/liveness
/actuator/health/readiness
```

В Kubernetes environment probe support обычно auto-enabled; property всё равно полезно знать:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
```

---

# 9. Additional paths на main server port

Очень полезная настройка:

```yaml
management:
  endpoint:
    health:
      probes:
        add-additional-paths: true
```

Получаем:

```text
/livez
/readyz
```

на main server port.

Spring Boot официально рекомендует этот подход, если management endpoints работают на отдельном management context/port, потому что management connector может быть healthy, когда main application connector уже не способен принимать requests.

---

# 10. Management-port false positive

Плохая mental model:

```text
management port 9090 returns 200
therefore application 8080 healthy
```

Possible failure:

```text
9090 management connector healthy
8080 main connector broken/exhausted
```

Kubelet видит 200 на 9090 и считает Pod healthy.

Поэтому probe через main server path часто лучше отражает real user traffic path.

---

# 11. startupProbe: зачем она существует

StartupProbe отвечает:

> Application уже закончила startup phase?

Пример:

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 5
  failureThreshold: 30
```

Budget:

```text
5s × 30
≈ 150s
```

---

# 12. Startup probe блокирует другие probes

Пока startupProbe не succeeds:

```text
liveness probe not active
readiness probe not active
```

Это защищает slow-starting application от преждевременного liveness restart.

Kubernetes explicitly defines this behavior.

---

# 13. Почему initialDelaySeconds слабее startupProbe

```yaml
livenessProbe:
  initialDelaySeconds: 120
```

Всегда ждёт fixed 120s.

StartupProbe:

```text
app ready at 15s
   ↓
startup succeeds immediately
   ↓
normal lifecycle starts
```

Это adaptive startup gate.

---

# 14. startupProbe endpoint

Часто startupProbe используют тот же low-cost endpoint, что liveness.

Почему:

```text
startup asks:
"application process reached internally alive state?"
```

А не:

> Все external dependencies сейчас доступны?

Startup не должна превращаться в distributed dependency transaction.

---

# 15. livenessProbe semantics

Liveness отвечает:

> Restart container реально может помочь?

Хорошие cases:

- deadlock;
- fatal internal state;
- event loop cannot progress;
- availability state BROKEN.

Плохие cases:

- PostgreSQL temporary outage;
- Kafka broker unavailable;
- payment API 503;
- DNS outage.

---

# 16. Почему DB в liveness опасна

Scenario:

```text
PostgreSQL down
      ↓
all app Pods liveness DOWN
      ↓
kubelet restarts all containers
      ↓
all JVMs recreate pools
      ↓
reconnect storm
      ↓
DB recovery harder
```

Restart Spring Boot не чинит PostgreSQL.

---

# 17. Liveness должна быть локальной

Хороший liveness question:

```text
Can this process make forward progress?
```

Плохой:

```text
Can every dependency in our architecture answer now?
```

Spring Boot docs отдельно предупреждают не включать external systems blindly в liveness health group.

---

# 18. readinessProbe semantics

Readiness отвечает:

> Стоит ли этому Pod давать **новые** requests прямо сейчас?

Failure:

```text
readiness fails
  ↓
Pod Ready=False
  ↓
EndpointSlice ready=false
  ↓
Service stops normal routing to Pod
```

Container остаётся Running.

---

# 19. Readiness работает весь lifecycle

Readiness полезна не только startup.

Например runtime overload:

```text
Pod overloaded
  ↓
readiness false
  ↓
temporarily remove traffic
  ↓
recover
  ↓
readiness true
```

Но такую adaptive readiness нужно проектировать очень осторожно, иначе все replicas могут одновременно disappear from Service.

---

# 20. Dependency в readiness: когда это разумно

Если application **не может дать никакого полезного response** без DB:

```text
DB down
  ↓
readiness=false
```

может быть корректно.

Пример pure transactional API, где каждый endpoint требует DB.

---

# 21. Dependency в readiness: где опасность

Все replicas используют одну DB:

```text
DB outage
  ↓
all Pods NotReady
  ↓
no Service endpoints
```

Теперь даже endpoints, которые могли вернуть controlled degraded response, недоступны.

Поэтому analyst/developer должны решить:

- есть ли degraded mode?
- есть ли cached/read-only path?
- нужна ли circuit breaker?
- должны ли health/status endpoints оставаться reachable?
- какая user experience правильная?

---

# 22. Readiness как business availability contract

Readiness — не чисто technical checkbox.

Она означает:

> Этот instance сейчас соответствует minimum contract для serving traffic.

Например:

```text
orders-api:
DB required
Kafka optional for synchronous request
recommendation-api optional
```

Тогда readiness может зависеть от DB, но не обязана зависеть от recommendation API.

---

# 23. Custom health indicators

Spring позволяет добавлять HealthIndicator.

Но плохой indicator:

```text
probe call
  ↓
DB query
  ↓
Kafka metadata
  ↓
3 external HTTP requests
  ↓
S3 call
```

каждые 5 секунд.

Это превращает health endpoint в distributed load generator.

---

# 24. Health endpoint должен быть cheap

Хорошие свойства:

- fast;
- deterministic;
- cheap;
- bounded timeout;
- low allocation;
- no side effects;
- clear semantics.

Probe endpoint вызывается часто каждым kubelet.

---

# 25. Probe types: HTTP, TCP, exec, gRPC

Kubernetes поддерживает несколько probe mechanisms.

Для typical Spring REST:

```text
HTTP probe
```

обычно наиболее expressive.

TCP:

```text
port accepts connection?
```

но не говорит, что Spring business stack реально готов.

Exec:

```text
run command inside container
```

требует tooling в image.

gRPC probe полезен для gRPC workloads.

---

# 26. Почему HTTP probe обычно лучше TCP для Spring

TCP success:

```text
socket accepts
```

Но application может быть:

- context partially failed;
- dependency unavailable;
- refusing traffic semantically.

HTTP endpoint может выразить application state точнее.

---

# 27. Почему exec probe с curl — плохой default

```yaml
exec:
  command:
    - curl
    - http://localhost:8080/readyz
```

Проблемы:

- curl нужно установить в image;
- shell/tooling увеличивает image;
- kubelet уже умеет HTTP probe.

Лучше:

```yaml
httpGet:
  path: /readyz
  port: http
```

---

# 28. Pod Ready condition

ReadinessProbe влияет на container readiness, а Pod aggregate readiness отражается в condition.

```text
container ready
  ↓
Pod Ready=True
```

При multi-container Pod:

```text
main Ready=true
sidecar Ready=false
  ↓
Pod may remain NotReady
```

Поэтому sidecar lifecycle может неожиданно влиять на Service traffic.

---

# 29. EndpointSlice chain

Official Kubernetes behavior:

```text
readinessProbe fails
      ↓
Pod Ready=False
      ↓
EndpointSlice controller updates endpoint condition
      ↓
ready=false
      ↓
normal Service traffic no longer uses endpoint
```

Это ключевой causal chain этой главы.

---

# 30. EndpointSlice conditions

Современный EndpointSlice имеет conditions:

```text
ready
serving
terminating
```

Это особенно важно для graceful termination.

У terminating endpoint:

```text
terminating=true
ready=false
```

может coexist с `serving=true` для draining semantics.

---

# 31. Running != Ready

```text
Running
=
container process exists

Ready
=
Pod eligible for normal Service traffic
```

Это normal state during:

- startup;
- temporary degradation;
- shutdown;
- dependency failure.

---

# 32. Probe timing fields

Основные:

```text
initialDelaySeconds
periodSeconds
timeoutSeconds
successThreshold
failureThreshold
```

Defaults в Kubernetes важно знать, но production values должны быть measured.

---

# 33. periodSeconds

Как часто probe выполняется.

Например:

```text
period=5s
```

Readiness при NotReady state Kubernetes может проверять чаще configured period, чтобы Pod быстрее вернуть в Ready.

---

# 34. timeoutSeconds

Слишком маленький:

```text
CPU throttling / GC pause
  ↓
false probe timeout
```

Слишком большой:

```text
real failure detection slower
```

---

# 35. failureThreshold

Например:

```text
period=10s
failureThreshold=3
```

rough tolerance:

```text
~30s repeated failure
```

до action.

Для liveness это restart threshold.
Для readiness — transition to NotReady.

---

# 36. successThreshold

Liveness/startup require successThreshold=1.

Readiness может использовать >1, если хотите требовать несколько successes до Ready.

Но слишком высокий threshold увеличит recovery delay.

---

# 37. Probe budget нужно считать

Пример startup:

```text
period=5s
failureThreshold=30
timeout=2s
```

Примерный upper detection/start budget ≠ ровно одна простая формула во всех scheduling details, но mental estimate:

```text
~150s
```

полезен.

---

# 38. Startup SLO должен быть известен

Не:

> Java иногда долго стартует, поставим 10 минут.

А:

```text
p95 startup = 18s
p99 startup = 31s
worst measured under CPU pressure = 45s
```

Тогда startupProbe budget можно выбрать осознанно.

---

# 39. Resources влияют на probes

Связь с главой 08:

```text
low CPU limit
  ↓
slow JVM/startup
  ↓
probe timeout
  ↓
false liveness/readiness failure
```

Probe tuning нельзя делать отдельно от resource sizing.

---

# 40. GC pause и probe timeout

Long stop-the-world pause:

```text
probe request arrives
  ↓
JVM cannot respond in timeout
  ↓
probe failure
```

Если один occasional pause приводит к restart, failureThreshold/timeout могут быть слишком aggressive.

---

# 41. Startup и Liquibase

Spring startup:

```text
ApplicationContext
  ↓
Liquibase/Flyway
  ↓
web ready
```

Если migration занимает 90s, startupProbe должна это учитывать.

Но долгие migrations сами по себе могут быть release architecture problem.

---

# 42. Startup и dependency wait

Плохой startup design:

```text
while DB unavailable:
  retry forever
```

Container:

```text
Running
never Ready
startup never completes
```

Нужны bounded startup policies и понятный failure mode.

---

# 43. Probe endpoints и Spring Security

Health endpoints должны быть доступны kubelet path без невозможной business authentication.

Но:

- не expose sensitive details;
- не открывать весь Actuator наружу;
- security chain должна учитывать additional paths.

Можно разрешить exact probe endpoints и закрыть остальные Actuator endpoints.

---

# 44. show-details

Production baseline:

```text
management.endpoint.health.show-details=never
```

или controlled role-based access.

Kubelet нужен status code, а не DB URL/user/stack detail.

---

# 45. Probe endpoint не monitoring dashboard

Kubelet нужно:

```text
200 / non-200
```

Operations нужны:

- error rate;
- latency;
- saturation;
- dependency health;
- business metrics.

Не перегружайте probe endpoint observability payload.

---

# 46. Monitoring != probe

```text
Probe
  -> local automatic action

Monitoring
  -> operator/system understanding
```

Например:

```text
p99 latency doubled
but readiness still true
```

может быть важным incident.

Не надо превращать любой SLO breach в immediate readiness=false.

---

# 47. Liveness false-positive blast radius

Если wrong liveness deploy на 20 replicas:

```text
all Pods fail liveness
  ↓
all restart
  ↓
zero usable capacity
```

Один неправильный health endpoint может создать outage сильнее исходной проблемы.

---

# 48. Readiness false-positive blast radius

```text
shared DB transient 2s timeout
  ↓
all Pods readiness false
  ↓
Service zero endpoints
```

Даже если приложение могло recover через retry/circuit breaker.

Поэтому readiness dependency policy должна учитывать shared-fate.

---

# 49. Partial degradation

Хорошая application architecture иногда умеет:

```text
GET /catalog -> works from cache
POST /payment -> unavailable
```

Один Pod-level readiness bit не умеет выразить per-endpoint readiness.

Это limitation.

Если service имеет radically different availability modes, может быть нужно:

- split services;
- degraded responses;
- routing architecture;
- application-level fallback.

---

# 50. Custom readiness state

Application может publish availability change.

Conceptually:

```java
AvailabilityChangeEvent.publish(
    applicationContext,
    ReadinessState.REFUSING_TRAFFIC
);
```

Это позволяет application осознанно self-drain, например перед internal maintenance.

Использовать осторожно и документировать trigger.

---

# 51. Custom liveness state

Можно publish:

```java
AvailabilityChangeEvent.publish(
    applicationContext,
    LivenessState.BROKEN
);
```

Это сильное действие.

Если оно приводит к liveness DOWN, kubelet может restart container.

Использовать только для действительно unrecoverable local state.

---

# 52. Probe config example

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 30

livenessProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /readyz
    port: http
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
```

Это reference, не universal values.

---

# 53. Spring config example

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true

      show-details: never
```

Main server:

```text
/livez
/readyz
```

---

# 54. Failure lab — slow startup

1. Добавить artificial startup delay.
2. Сделать liveness aggressive.
3. Не использовать startupProbe.
4. Наблюдать restart.
5. Добавить startupProbe.
6. Сравнить behavior.

Expected lesson:

```text
slow startup
!= dead process
```

---

# 55. Failure lab — wrong readiness path

```yaml
readinessProbe:
  httpGet:
    path: /broken
    port: http
```

Observe:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice \
  -l kubernetes.io/service-name=<service>
```

Expected:

```text
Running
Ready=False
endpoint ready=false
```

---

# 56. Failure lab — wrong liveness path

```yaml
livenessProbe:
  httpGet:
    path: /broken
    port: http
```

Observe:

```bash
kubectl get pod -w
kubectl describe pod <pod>
kubectl logs <pod> --previous
```

Expected:

```text
restartCount grows
```

Root cause:

```text
probe configuration
```

not application crash.

---

# 57. Failure lab — DB outage with correct liveness

Scenario:

```text
DB down
```

Expected:

```text
liveness remains UP
```

Readiness depends on documented service contract.

Check:

- no mass restart;
- app logs controlled dependency failures;
- readiness expected state;
- recovery when DB returns.

---

# 58. Failure lab — DB in liveness

Intentionally configure DB health into liveness.

Take DB down.

Observe:

```text
all Pods restart
reconnect storm
```

This is a powerful demonstration of cascading failure.

---

# 59. Failure lab — management-port false positive

Configure management port separately.

Then simulate main HTTP connector failure/overload while management remains responsive.

Observe:

```text
management probe 200
main user traffic broken
```

Enable additional probe paths on main server and compare.

---

# 60. Failure lab — CPU starvation

Set tight CPU limit.

Apply load.

Observe:

- probe latency;
- timeouts;
- Ready condition;
- CPU throttling.

Goal:

```text
resource failure
can surface as probe failure
```

---

# 61. Failure lab — sidecar NotReady

Multi-container Pod:

```text
main Ready=true
sidecar Ready=false
```

Observe Pod Ready state and Service routing.

Goal: understand aggregate Pod readiness.

---

# 62. Failure lab — recovery

Make readiness fail temporarily, then restore dependency/state.

Observe:

```text
Ready=false
  ↓
no normal traffic
  ↓
condition recovers
  ↓
Ready=true
  ↓
endpoint restored
```

No container restart should be required for a pure readiness incident.

---

# 63. Rollout interaction

New revision:

```text
new Pod starts
  ↓
startupProbe
  ↓
readiness
  ↓
Available
  ↓
old Pod can scale down
```

Bad readiness can stall rollout even when JVM process runs.

---

# 64. minReadySeconds interaction

Deployment may require:

```yaml
minReadySeconds: 10
```

Then:

```text
Pod Ready=true
  ↓
must remain Ready for 10s
  ↓
Deployment counts Available
```

Probe flapping can prevent availability progress.

---

# 65. Probe flapping

```text
Ready true
Ready false
Ready true
Ready false
```

Possible causes:

- threshold too sensitive;
- dependency unstable;
- CPU pressure;
- health endpoint expensive;
- GC pauses.

Flapping can cause traffic churn and rollout stalls.

---

# 66. Probe observability

Useful metrics/logs:

- probe failure count;
- readiness transitions;
- restart count;
- startup duration;
- health indicator latency;
- dependency status;
- CPU/GC around failures.

Do not rely only on events after incident.

---

# 67. Analyst view

Analyst defines semantics:

| Question | Example |
|---|---|
| What means “ready”? | can serve core order flows |
| Can read-only work without DB? | no |
| Is Kafka required synchronously? | no |
| Can recommendations fail degraded? | yes |
| Max startup time | 45s |
| Max traffic drain time | 20s |
| Dependency outage behavior | 503/degraded, no restart |

Это превращает probes из platform detail в availability contract.

---

# 68. Developer view

Checklist:

- [ ] Actuator configured;
- [ ] liveness local;
- [ ] readiness matches service contract;
- [ ] startup measured;
- [ ] health indicators cheap;
- [ ] main server port probed;
- [ ] probe endpoints accessible but not overexposed;
- [ ] dependency health not blindly included;
- [ ] ApplicationAvailability understood;
- [ ] custom state changes tested;
- [ ] graceful shutdown compatible with readiness/termination.

---

# 69. Tester/SDET view

Tests:

- slow startup;
- startup timeout;
- bad liveness;
- bad readiness;
- DB outage;
- external API outage;
- management/main-port split;
- CPU starvation;
- GC pause under load;
- sidecar NotReady;
- recovery to Ready;
- rollout with flapping readiness.

For every test record:

```text
Expected:
restart?
traffic removed?
Pod remains Running?
EndpointSlice changes?
recovery automatic?
```

---

# 70. Platform/SRE view

Platform owns/shared:

- kubelet probe behavior baseline;
- Service/EndpointSlice visibility;
- ingress/LB interaction;
- platform health monitoring;
- restart alerts;
- readiness dashboards;
- probe-related policy/admission if any.

Developer owns application health semantics.

---

# 71. Anti-patterns

1. Один universal health endpoint без semantic analysis.
2. DB/Kafka/external API в liveness.
3. Startup через огромный fixed initialDelay.
4. Probe через management port, когда main port может fail independently.
5. Health endpoint делает expensive queries.
6. timeoutSeconds=1 без measurement.
7. failureThreshold слишком aggressive.
8. readiness зависит от shared dependency без degraded-mode analysis.
9. Actuator details exposed publicly.
10. curl-based exec probe только потому, что “так видел в примере”.
11. Readiness false при каждом short transient error.
12. Restart Pod при readiness-only incident.
13. Считать probe monitoring system.
14. Ignoring sidecar readiness.
15. Tuning probes without CPU/GC/load tests.

---

# 72. CKAD mapping

Нужно уверенно писать:

```yaml
startupProbe:
livenessProbe:
readinessProbe:
```

и понимать:

```text
startup failure
→ restart after threshold

liveness failure
→ restart

readiness failure
→ no normal Service traffic
```

Commands:

```bash
kubectl get pod
kubectl describe pod
kubectl get endpointslice
kubectl logs
kubectl logs --previous
```

Production добавляет Spring availability semantics, dependency policy, main-port caveat и degraded modes.

---

# 73. Связь с предыдущей главой

Глава 08:

```text
resources determine
how fast/healthy JVM can run
```

Глава 09:

```text
probes translate
application state
into Kubernetes actions
```

Получаем:

```text
CPU/memory
   ↓
JVM behavior
   ↓
ApplicationAvailability
   ↓
probe result
   ↓
Pod Ready/restart
   ↓
Service traffic
```

---

# 74. Что изучать дальше

Следующая глава:

- [10 — Graceful shutdown and termination](10-graceful-shutdown-termination.md)

Логика перехода:

```text
Глава 09:
Kubernetes понимает, когда Pod готов к traffic

Следующий вопрос:
что происходит, когда Pod нужно удалить?
когда Service перестаёт посылать traffic?
когда приходит SIGTERM?
как Spring завершает in-flight work?
как не потерять message/task/request?

       ↓

Глава 10
```

---

# 75. Production-like examples

- [Showcase 01 — Internal REST](../../showcases/01-internal-rest-service/README.md)
- [Showcase 01 annotated.yaml](../../showcases/01-internal-rest-service/annotated.yaml)
- [Showcase 02 — Public API / edge routing](../../showcases/02-public-api-ingress-tls/README.md)
- [Lab 02 — JVM/resources/probes](../../labs/02-jvm-resources-probes/README.md)

---

# 76. Control questions

1. Что такое ApplicationAvailability?
2. Чем LivenessState отличается от ReadinessState?
3. Что означает CORRECT?
4. Что означает ACCEPTING_TRAFFIC?
5. Как Actuator exposes probe states?
6. Что делает add-additional-paths?
7. Почему management port может дать false positive?
8. Что делает startupProbe?
9. Что происходит с liveness/readiness до startup success?
10. Почему initialDelay слабее startupProbe?
11. Что должно входить в liveness?
12. Почему DB в liveness опасна?
13. Когда DB может входить в readiness?
14. Чем Running отличается от Ready?
15. Как readiness влияет на EndpointSlice?
16. Что означают EndpointSlice ready/serving/terminating?
17. Чем HTTP probe лучше TCP для Spring?
18. Почему curl exec probe плохой default?
19. Что делает timeoutSeconds?
20. Что делает failureThreshold?
21. Что делает successThreshold?
22. Почему probe timings надо измерять?
23. Как CPU throttling может сломать probe?
24. Как GC pause может вызвать false negative?
25. Почему custom health indicator должен быть cheap?
26. Чем probe отличается от monitoring?
27. Почему Pod-level readiness ограничивает partial degradation model?
28. Как application может publish custom readiness state?
29. Как readiness влияет на rollout?
30. Как minReadySeconds связан с readiness stability?

---

# 77. Sources

## Spring Boot

- https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
- https://docs.spring.io/spring-boot/api/java/org/springframework/boot/availability/ApplicationAvailability.html

## Kubernetes

- https://kubernetes.io/docs/concepts/workloads/pods/probes/
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/reference/kubernetes-api/discovery-resources/endpoint-slice-v1/

Актуальные facts главы:

- startupProbe suppresses liveness/readiness until it succeeds.
- readiness failure makes Pod not-ready for normal Service traffic and EndpointSlice reflects this.
- Spring Boot ApplicationAvailability exposes liveness/readiness application states.
- Spring Boot probe health groups can be exposed on the main server as /livez and /readyz using add-additional-paths.
- Separate management context can remain healthy while main application connector is unhealthy, so main-port probe paths are often safer.
- EndpointSlice conditions include ready, serving and terminating.
