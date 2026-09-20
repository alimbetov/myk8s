# 10 — Graceful shutdown, Pod termination и draining

Проверено: 2026-09-20.

Эта глава отвечает на вопрос:

> Новый Pod уже умеет становиться Ready. **Что происходит со старым Pod, когда его нужно удалить, как перестать принимать новую работу, как закончить in-flight requests/messages/transactions и что произойдёт, если приложение не успеет завершиться до hard deadline?**

Главная mental model:

```text
Pod deletion requested
      ↓
termination grace starts
      ↓
two things begin in parallel
      |
      +--> endpoint becomes terminating / ready=false
      |
      +--> kubelet begins local shutdown
                |
                +--> preStop if configured
                |
                +--> TERM to container process
                |
                +--> Spring graceful shutdown
                |
                +--> in-flight work drains
                |
                +--> process exits
      |
      v
if grace expires
      ↓
forceful kill
```

Ключевой вывод:

> Graceful shutdown — это не одна команда. Это **distributed coordination между control plane, kubelet, service dataplane, application server и текущей работой процесса**.

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь объяснить:

1. Что происходит после `kubectl delete pod`.
2. Когда начинается `terminationGracePeriodSeconds`.
3. Когда вызывается `preStop`.
4. Когда process получает TERM.
5. Почему `preStop` расходует тот же grace budget.
6. Что происходит после истечения grace period.
7. Как EndpointSlice отражает terminating endpoint.
8. Чем `ready`, `serving` и `terminating` отличаются.
9. Почему нельзя считать termination строго последовательным “traffic off → SIGTERM”.
10. Как Spring Boot graceful shutdown работает с HTTP.
11. Что делает `spring.lifecycle.timeout-per-shutdown-phase`.
12. Почему application timeout должен быть меньше Kubernetes hard deadline.
13. Что происходит с HTTP keep-alive.
14. Почему long requests усложняют termination.
15. Как Kafka consumer должен завершаться.
16. Как RabbitMQ listener должен завершаться.
17. Что происходит с DB transaction при graceful vs forced termination.
18. Почему graceful shutdown не заменяет idempotency/Outbox/Saga.
19. Как scheduled/background tasks влияют на shutdown.
20. Как sidecars влияют на termination order.
21. Когда `preStop sleep` оправдан, а когда это cargo cult.
22. Как rollout и node drain используют один termination lifecycle.
23. Как тестировать graceful shutdown.
24. Какие race conditions возможны.
25. Что должен фиксировать analyst/developer/tester/platform engineer.

---

# 2. Почему shutdown — часть zero-downtime rollout

Обычно внимание сосредоточено на новом Pod:

```text
new Pod starts
   ↓
Ready
   ↓
traffic
```

Но rollout также удаляет старый Pod.

Если shutdown плохой:

```text
old Pod receives request
      ↓
termination starts
      ↓
process killed
      ↓
TCP reset / 5xx
```

То есть:

```text
new Pod readiness
+
old Pod graceful termination
=
rollout availability
```

---

# 3. Что именно запускает termination

Причины:

- manual delete;
- Deployment scale-down;
- RollingUpdate;
- node drain / eviction;
- preemption/resource management event;
- some probe-triggered container restarts;
- Pod replacement by controller.

Но важно отличать:

```text
Pod termination lifecycle
vs
container process crash
```

Если process сам crashed, `preStop` не выполняется.

---

# 4. deletionTimestamp

После delete request API object не исчезает мгновенно.

Pod получает termination metadata:

```text
deletionTimestamp
+
grace period
```

CLI:

```text
STATUS=Terminating
```

Это сигнал:

> Pod уже не считается нормальным in-service replica, но local shutdown ещё может идти.

---

# 5. terminationGracePeriodSeconds

Default Pod termination grace period:

```text
30 seconds
```

Пример:

```yaml
spec:
  terminationGracePeriodSeconds: 30
```

Это **общий outer deadline** для graceful Pod/container termination.

---

# 6. Grace countdown начинается до preStop

Очень важный nuance:

```text
delete requested
   ↓
grace timer starts
   ↓
preStop
   ↓
TERM
   ↓
application shutdown
```

То есть:

```text
preStop time
+
application shutdown time
<=
terminationGracePeriodSeconds
```

---

# 7. preStop не даёт дополнительное время

Пример:

```text
grace = 30s
preStop = 20s
Spring shutdown = 20s
```

Нужно:

```text
40s
```

но доступно около:

```text
30s
```

Следовательно application может быть force-killed до clean exit.

Kubernetes даёт небольшое one-off extension, если preStop всё ещё выполняется в момент истечения grace, но на это нельзя проектировать normal shutdown.

---

# 8. preStop выполняется до TERM

Flow:

```text
preStop hook
  ↓ completes
TERM
  ↓
process shutdown
```

Это важно.

Если `preStop` завис:

```text
TERM задерживается
```

а grace timer уже идёт.

---

# 9. TERM идёт main process

После preStop kubelet просит runtime остановить container.

Обычно runtime отправляет:

```text
SIGTERM
```

process 1 внутри container.

Если image задаёт `STOPSIGNAL`, runtime может использовать его.

В современных Kubernetes есть alpha container-level custom stop signal feature, но типичный Spring Boot contract должен корректно обрабатывать TERM.

---

# 10. Почему PID 1 снова важен

Хорошо:

```text
PID 1 = java
```

Тогда TERM получает JVM.

Плохо:

```text
PID 1 = shell
   |
   +--> java
```

если shell не forward signal.

Wrapper:

```sh
#!/bin/sh
exec java -jar app.jar
```

решает этот class problems.

---

# 11. EndpointSlice меняется параллельно

В то же время control plane обновляет Service endpoint state.

Conceptually:

```text
Pod deletion
   |
   +--> kubelet shutdown
   |
   +--> EndpointSlice update
```

Это не строгая последовательность:

```text
first completely remove all traffic
then send TERM
```

Distributed components видят changes не мгновенно.

---

# 12. EndpointSlice conditions при termination

Для terminating endpoint:

```text
terminating = true
ready       = false
```

`ready=false` сделано в том числе для backward compatibility, чтобы обычные service proxies/load balancers не использовали terminating endpoint для normal traffic.

При этом:

```text
serving
```

может отражать фактическую ability endpoint ещё обслуживать traffic.

---

# 13. ready vs serving vs terminating

Mental model:

```text
ready
= serving AND not terminating
  для обычного routing semantics

serving
= endpoint способен обслуживать traffic

terminating
= Pod termination начался
```

Это особенно важно для sophisticated connection draining implementations.

---

# 14. Почему traffic может ещё существовать после начала termination

Причины:

- already established connection;
- HTTP keep-alive;
- client connection pool;
- proxy/load balancer propagation;
- requests already in flight;
- terminating endpoint draining behavior.

Поэтому application должна уметь:

```text
stop new work
+
finish existing work
```

а не рассчитывать, что network мгновенно становится zero.

---

# 15. Spring Boot graceful shutdown

В актуальном Spring Boot graceful shutdown enabled by default для supported embedded web servers.

Conceptually:

```text
SIGTERM
  ↓
Spring ApplicationContext closing
  ↓
WebServer graceful shutdown
  ↓
stop accepting new requests
  ↓
allow active requests to finish
  ↓
SmartLifecycle shutdown phases
```

---

# 16. Spring timeout

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

Это timeout per shutdown phase, используемый Spring lifecycle shutdown processing.

Kubernetes:

```yaml
terminationGracePeriodSeconds: 30
```

Хорошая relationship:

```text
application graceful budget
<
platform hard deadline
```

с запасом.

---

# 17. Почему budget нужен с запасом

Например:

```text
Kubernetes grace = 30s
Spring timeout   = 29s
```

Остаётся почти ноль времени на:

- preStop;
- scheduler/runtime delays;
- connection close;
- JVM finalization;
- container stop overhead.

Лучше иметь explicit margin.

---

# 18. HTTP graceful shutdown

Spring Boot web server старается:

```text
no new requests
+
existing requests complete
```

Но exact behavior зависит от server implementation и persistent connections.

Tomcat, Jetty и Reactor Netty прекращают принимать новые requests на network layer в graceful shutdown mode.

---

# 19. HTTP keep-alive

Client может иметь existing connection:

```text
client
  |
  | keep-alive TCP connection
  v
terminating Pod
```

Поэтому shutdown test должен включать не только новые connections, но и existing keep-alive behavior.

---

# 20. Long-running request

Scenario:

```text
GET /export
takes 60s
```

Grace:

```text
30s
```

Если termination начинается на second 5:

```text
remaining request time = 55s
available grace        = 30s
```

Request может быть interrupted.

Это architecture issue, не “увеличим grace бесконечно”.

---

# 21. Когда long request нужно externalize

Если synchronous request выполняет минуты:

```text
request
  ↓
create job
  ↓
return job id
  ↓
worker processes asynchronously
  ↓
state stored externally
```

Тогда Pod lifecycle меньше связан с user connection lifetime.

---

# 22. HTTP failure lab

Endpoint:

```text
GET /slow
sleep 10s
```

Запустить:

```bash
curl http://orders/slow &
kubectl delete pod <pod>
```

Наблюдать:

- completed ли request;
- сколько Pod Terminating;
- replacement Pod;
- application shutdown logs;
- client reset/5xx;
- endpoint conditions.

---

# 23. preStop: когда он нужен

`preStop` оправдан, если есть **конкретное действие до TERM**.

Например:

- вызвать application self-drain endpoint;
- deregister из external non-Kubernetes registry;
- выполнить short protocol-specific step.

Но большинство basic Spring Boot HTTP workloads должны сначала попытаться опираться на:

```text
EndpointSlice termination
+
Spring graceful shutdown
```

---

# 24. preStop sleep cargo cult

Часто встречается:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 10"]
```

Мотив:

> “Дадим ingress время убрать Pod.”

Проблемы:

- требует shell;
- тратит grace;
- фиксированное число не связано с measured propagation;
- может скрывать реальный draining flaw.

Sleep допустим как measured workaround, но не default architecture.

---

# 25. preStop и minimal images

Distroless image может не иметь:

```text
/bin/sh
sleep
curl
```

Exec preStop, зависящий от shell, тогда fail.

Можно использовать HTTP lifecycle hook или image/runtime mechanism, если он нужен.

---

# 26. Container lifecycle hooks

`preStop` handlers могут быть:

- exec;
- HTTP;
- sleep handler в поддерживаемой lifecycle API.

Но semantics важнее mechanism:

> Что именно должно закончиться до TERM и почему?

---

# 27. Sidecars и termination order

Современный Kubernetes имеет special lifecycle semantics для native sidecars.

Main application containers завершаются раньше sidecars, а sidecars затем останавливаются в обратном порядке definition.

Это полезно для logging/proxy sidecars, которые должны оставаться живыми, пока main app ещё завершает работу.

Для обычных containers без native sidecar semantics stop order не гарантирован.

---

# 28. Почему это важно для proxy sidecar

Pod:

```text
app
+
proxy sidecar
```

Если proxy завершится раньше app:

```text
app still finishing request
   ↓
network proxy gone
   ↓
request fails
```

Native sidecar lifecycle помогает решить shutdown ordering.

---

# 29. Kafka consumer termination

HTTP draining и Kafka shutdown — разные problems.

Consumer lifecycle:

```text
consumer active
   ↓
stop polling/accepting new records
   ↓
finish current processing
   ↓
commit offset when safe
   ↓
leave group / close
```

---

# 30. Kafka duplicate scenario

Record:

```text
offset=100
```

Processing:

```text
DB update succeeds
   ↓
SIGKILL before offset commit
   ↓
consumer group re-delivers offset=100
```

Business operation may run twice.

Graceful shutdown reduces probability, but does not remove duplicate-delivery semantics.

Hence:

```text
idempotency
still required
```

---

# 31. Kafka rebalance

When consumer leaves:

```text
partition ownership changes
```

During shutdown/rebalance:

- in-flight work;
- offset commit;
- processing time;
- session/poll settings

must be compatible.

A consumer that takes longer to finish than termination grace can still be killed.

---

# 32. RabbitMQ listener termination

Safe conceptual flow:

```text
stop new deliveries
   ↓
finish current message
   ↓
DB/external side effect
   ↓
ACK
   ↓
close channel/connection
```

If processing fails/interrupted:

```text
NACK/requeue or redelivery semantics
```

depending on listener/container config.

---

# 33. Rabbit duplicate scenario

Message processing:

```text
charge customer
   ↓
process killed before ACK
   ↓
message redelivered
   ↓
charge customer again
```

Graceful shutdown helps but does not solve exactly-once business effects.

Need:

- idempotency key;
- transactional design;
- outbox/inbox;
- deduplication.

---

# 34. DB transaction during termination

Case A:

```text
SIGTERM
   ↓
Spring allows current request to finish
   ↓
transaction commits
```

Case B:

```text
grace expires
   ↓
SIGKILL
   ↓
DB connection closes
   ↓
uncommitted DB transaction rolls back
```

But external effects outside DB may already have occurred.

---

# 35. Transaction != distributed atomicity

Example:

```text
DB commit
   ↓
external payment API
   ↓
process killed before local state update
```

или наоборот.

Graceful shutdown не делает эти steps atomic.

Это место для:

- Saga;
- Outbox;
- idempotency;
- retry-safe design.

---

# 36. Scheduled tasks

Если Spring scheduler работает в каждом Pod:

```text
5 replicas
  ↓
same scheduled method
  ↓
potentially 5 executions
```

Shutdown scheduled task тоже должен иметь defined semantics.

Для cluster-wide scheduled job часто лучше:

- CronJob;
- distributed lock;
- dedicated scheduler.

---

# 37. Background executors

Custom `ExecutorService`/thread pools могут продолжать работу при context close, если lifecycle configured неправильно.

Developer должен проверить:

- accept new tasks stops;
- queued tasks behavior;
- wait-for-tasks-to-complete;
- timeout;
- interrupt behavior.

---

# 38. Shutdown hooks должны быть bounded

Плохо:

```text
on shutdown:
  retry external API forever
```

Это гарантирует forced kill.

Каждый shutdown step должен иметь finite timeout.

---

# 39. SIGKILL

Если grace expires:

```text
remaining processes
   ↓
forceful kill
```

SIGKILL:

- нельзя catch;
- нельзя graceful cleanup;
- finally/shutdown hooks не гарантируются.

Поэтому hard deadline — настоящий boundary.

---

# 40. Pod deletion vs container restart

Liveness failure может restart container внутри Pod.

`preStop` вызывается и перед management-triggered container termination, если container ещё running.

Но EndpointSlice Pod termination semantics относятся именно к Pod deletion/termination lifecycle.

Не смешивайте:

```text
container restart
vs
Pod deletion
```

---

# 41. Node drain

`kubectl drain` инициирует eviction для обычных Pods.

Дальше:

```text
eviction
  ↓
Pod termination lifecycle
  ↓
preStop/TERM/grace
```

Поэтому graceful shutdown должен быть протестирован не только при rollout, но и при maintenance.

---

# 42. PDB + drain

PDB может ограничить simultaneous voluntary disruptions.

Но PDB не продлевает application grace period и не завершает work за приложение.

Он отвечает:

```text
можно ли сейчас добровольно evict ещё один Pod?
```

а graceful shutdown отвечает:

```text
как уже terminating Pod корректно закончится?
```

---

# 43. Rollout sequence

RollingUpdate conceptually:

```text
new Pod created
   ↓
new Pod Ready
   ↓
old Pod selected for scale-down
   ↓
old endpoint terminating
   ↓
old app graceful shutdown
   ↓
old Pod gone
```

Если old Pod termination плохая, `maxUnavailable=0` не спасает in-flight request.

---

# 44. Endpoint race during rollout

Possible timeline:

```text
T0 delete old Pod
T0+ε EndpointSlice marked terminating
T0+ε kubelet starts preStop
T0+δ proxies observe endpoint update
T0+δ existing connection still sends data
T0+... app shutdown progresses
```

Это distributed propagation.

Поэтому shutdown design должен быть tolerant к short races.

---

# 45. Connection draining layers

Possible actors:

- kubelet;
- EndpointSlice controller;
- kube-proxy/CNI service dataplane;
- Ingress controller;
- Gateway;
- cloud load balancer;
- service mesh;
- client connection pool;
- application server.

Каждый может иметь своё propagation/draining timing.

---

# 46. External LoadBalancer caveat

Если traffic path:

```text
Internet
  ↓
cloud LB
  ↓
Ingress
  ↓
Service
  ↓
Pod
```

то endpoint removal из Kubernetes — только часть chain.

Cloud LB/Ingress may have their own deregistration/drain behavior.

Нужно тестировать actual platform path.

---

# 47. Service mesh caveat

Sidecar proxy может:

- intercept traffic;
- hold connections;
- implement drain;
- have own termination ordering.

Поэтому mesh changes shutdown semantics.

Do not blindly copy non-mesh preStop recipes into mesh environment.

---

# 48. Grace period sizing

Start from longest safe in-flight unit.

Example:

```text
HTTP p99 max safe request = 8s
Rabbit message p99       = 12s
shutdown overhead        = 3s
margin                   = 5s
```

Then:

```text
grace ≈ > 20s
```

Exact design depends on workload.

---

# 49. Too short grace

Symptoms:

- TCP resets;
- partial responses;
- unacked/redelivered messages;
- unfinished cleanup;
- repeated work;
- forced termination.

---

# 50. Too long grace

Costs:

- rollout slow;
- node drain slow;
- bad/stuck Pod holds resources;
- maintenance delayed.

Grace is not “the bigger the safer”.

---

# 51. Shutdown SLO

Useful requirement:

```text
95% Pods terminate < 8s
99% Pods terminate < 15s
hard grace = 30s
forced kill rate = 0
```

This is measurable.

---

# 52. Failure lab — grace too short

```yaml
terminationGracePeriodSeconds: 2
```

Endpoint takes 10s.

Expected:

```text
forced termination
request may fail
```

Observe client and Pod events/logs.

---

# 53. Failure lab — stuck preStop

Hook:

```text
sleep longer than grace
```

Observe:

```text
Pod Terminating
TERM delayed
grace expires
forced termination
```

Goal:

> understand that preStop is part of the budget.

---

# 54. Failure lab — wrapper without exec

Compare:

```sh
java -jar app.jar
```

with:

```sh
exec java -jar app.jar
```

Delete Pod and compare Spring shutdown logs/timing.

---

# 55. Failure lab — HTTP keep-alive

Use client with connection reuse.

Start several requests while terminating Pod.

Observe:

- new requests;
- existing connection behavior;
- response completion;
- proxy/load balancer behavior.

---

# 56. Failure lab — Kafka in-flight message

Make handler take 10–15s.

Delete Pod mid-processing.

Observe:

- offset commit;
- redelivery;
- duplicate business side effect;
- rebalance;
- replacement consumer.

Then add idempotency and repeat.

---

# 57. Failure lab — Rabbit in-flight message

Make listener slow.

Delete Pod.

Observe:

- ACK/NACK;
- redelivery;
- queue/unacked;
- duplicate effect.

---

# 58. Failure lab — DB transaction

Open transaction, make it slow, delete Pod.

Observe:

- commit or rollback;
- connection close;
- app logs;
- external side effects.

Goal:

```text
DB rollback
does not automatically undo external effects
```

---

# 59. Failure lab — node drain

Before:

```bash
kubectl get pod -o wide
kubectl get pdb
```

Drain test node.

Observe:

- eviction;
- replacement scheduling;
- terminating endpoint;
- graceful shutdown duration;
- service error rate.

---

# 60. Failure lab — rollout under load

Generate steady traffic.

Roll new image.

Measure:

- 5xx;
- resets;
- p95/p99;
- active Pods;
- terminating Pods;
- endpoint states;
- shutdown duration.

A rollout is “zero downtime” only if user-visible SLI confirms it.

---

# 61. Logs you want during shutdown

Useful events:

```text
shutdown-start
readiness-refusing
http-drain-start
active-requests=N
consumer-stop-start
message-finish
db-pool-close
shutdown-complete
duration_ms
```

Avoid secrets.

---

# 62. Metrics you want

- active HTTP requests;
- shutdown duration;
- forced kills;
- termination count;
- consumer in-flight;
- redeliveries;
- unacked messages;
- Kafka rebalance/lag;
- DB transactions;
- rollout 5xx.

---

# 63. Analyst view

Analyst should define termination semantics.

| Question | Example |
|---|---|
| Longest synchronous request | 8s |
| Can request retry safely? | GET yes, payment POST via idempotency key |
| Message delivery | at-least-once |
| Duplicate effect allowed? | no |
| Max shutdown | 20s |
| Forced kill acceptable? | only exceptional |
| Batch interruption | restartable |
| User-visible rollout error budget | < 0.1% |

“Graceful shutdown required” alone is insufficient requirement.

---

# 64. Developer view

Checklist:

- [ ] PID 1/signal path correct;
- [ ] Spring graceful shutdown enabled/default understood;
- [ ] shutdown timeout explicit;
- [ ] no unbounded shutdown hooks;
- [ ] HTTP long requests bounded;
- [ ] Kafka/Rabbit listeners tested;
- [ ] DB transactions understood;
- [ ] idempotency for redelivery/retry;
- [ ] custom executors stop correctly;
- [ ] scheduled tasks have cluster semantics;
- [ ] shutdown logs/metrics exist;
- [ ] no blind preStop sleep.

---

# 65. Tester/SDET view

Tests:

- delete Pod during HTTP request;
- keep-alive during termination;
- short grace;
- stuck preStop;
- wrapper/signal failure;
- Kafka message in flight;
- Rabbit delivery in flight;
- DB transaction in flight;
- rollout under load;
- node drain;
- sidecar/proxy drain;
- forced kill.

For each:

```text
Was work completed?
Was it duplicated?
Was it rolled back?
Did user see error?
Did replacement take over?
Was recovery automatic?
```

---

# 66. Platform/SRE view

Platform owns/shared:

- default grace conventions;
- ingress/LB draining;
- service mesh behavior;
- node drain procedures;
- PDB;
- eviction;
- runtime stop signal behavior;
- endpoint observability;
- forced termination alerts.

Application team owns protocol/business shutdown semantics.

---

# 67. Anti-patterns

1. считать endpoint removal мгновенным;
2. считать termination строго последовательным;
3. grace=5s для 30s work unit;
4. blind preStop sleep;
5. preStop longer than grace;
6. wrapper without exec;
7. long synchronous operations без interruption design;
8. ACK/offset commit before business work safely complete;
9. graceful shutdown как замена idempotency;
10. scheduled job in every replica without cluster semantics;
11. no shutdown test under load;
12. no metrics for forced kill;
13. PDB considered graceful shutdown mechanism;
14. rollout only tested at zero traffic;
15. ignore sidecar/proxy draining.

---

# 68. CKAD mapping

Нужно понимать:

- `terminationGracePeriodSeconds`;
- lifecycle hooks;
- Pod deletion;
- Deployment rollout;
- probes;
- logs;
- container lifecycle.

Production добавляет:

```text
HTTP draining
keep-alive
consumer semantics
DB transactions
idempotency
EndpointSlice conditions
ingress/LB propagation
sidecar ordering
```

---

# 69. Связь с предыдущей главой

Глава 09:

```text
readiness
  ↓
traffic eligibility
```

Глава 10:

```text
termination begins
  ↓
endpoint terminating
  ↓
traffic drains
  ↓
process shuts down
```

Together:

```text
startup
  ↓
Ready
  ↓
serve
  ↓
termination
  ↓
drain
  ↓
exit
```

---

# 70. Что изучать дальше

Следующая глава:

- [11 — HTTP clients, DNS, timeout, retry and pools](11-http-clients-dns-timeout-retry-pool.md)

Логика перехода:

```text
Глава 10:
мы научились корректно завершать requests

Следующий вопрос:
как Spring Boot должен создавать outgoing HTTP calls,
как работают DNS, connection pools, timeouts,
retry и deadlines,
и почему неправильный client config
может превратить dependency failure
в outage собственного сервиса?

       ↓

Глава 11
```

---

# 71. Production-like examples

- [Showcase 01 — Internal REST](../../showcases/01-internal-rest-service/README.md)
- [Showcase 04 — Batch CronJob](../../showcases/04-batch-cronjob/README.md)
- [Showcase 10 — Kafka producer/consumer](../../showcases/10-kafka-producer-consumer/README.md)
- [Showcase 11 — RabbitMQ worker](../../showcases/11-rabbitmq-worker/README.md)
- [Showcase 17 — Native sidecar](../../showcases/17-native-sidecar/README.md)

---

# 72. Control questions

1. Когда начинается termination grace?
2. Что происходит раньше: preStop или TERM?
3. Почему preStop расходует тот же grace period?
4. Что произойдёт после grace expiry?
5. Что означает EndpointSlice terminating=true?
6. Почему terminating endpoint ready=false?
7. Что означает serving?
8. Почему shutdown и endpoint update идут параллельно?
9. Почему traffic может ещё идти после delete?
10. Как Spring Boot graceful shutdown работает с HTTP?
11. Что делает timeout-per-shutdown-phase?
12. Почему application timeout должен быть меньше Pod grace?
13. Как keep-alive влияет на draining?
14. Почему 60s request и 30s grace конфликтуют?
15. Когда нужен preStop?
16. Почему sleep preStop плохой default?
17. Почему PID 1 важен?
18. Что происходит с Kafka offset при kill до commit?
19. Почему Rabbit message может быть redelivered?
20. Что происходит с uncommitted DB transaction при forced kill?
21. Почему idempotency всё равно нужна?
22. Как scheduled tasks усложняют multi-replica deployment?
23. Чем PDB отличается от graceful shutdown?
24. Почему rollout нужно тестировать под load?
25. Что нужно измерять как shutdown SLO?

---

# 73. Sources

## Kubernetes

- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/
- https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- https://kubernetes.io/docs/tutorials/services/pods-and-endpoint-termination-flow/

## Spring Boot

- https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html

Актуальные facts главы:

- Pod termination grace countdown begins before `preStop`.
- `preStop` must finish before TERM is sent and consumes the same grace budget.
- Default Pod termination grace period is 30 seconds.
- When the grace period expires, remaining processes are forcefully terminated.
- Terminating Pod endpoints remain represented in EndpointSlice with `terminating=true` and `ready=false`; `serving` can indicate that they can still serve while draining.
- Endpoint termination updates and kubelet local shutdown proceed concurrently.
- Spring Boot graceful shutdown is enabled by default for supported embedded web servers and is bounded by `spring.lifecycle.timeout-per-shutdown-phase`.
