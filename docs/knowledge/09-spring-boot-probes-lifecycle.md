# 09 — Spring Boot probes and application lifecycle

## Учебная карта темы

### Три разных вопроса Kubernetes

```text
startupProbe
  "Приложение вообще успело стартовать?"

livenessProbe
  "Этот process нужно перезапустить?"

readinessProbe
  "Можно ли отправлять сюда новый traffic?"
```

### Runtime flow

```text
container starts
   ↓
startupProbe
   ↓ success
liveness/readiness active
   ↓
readiness 200
   ↓
Pod Ready=True
   ↓
EndpointSlice ready=true
   ↓
Service routes traffic
```

### Annotated fragment

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 5
  failureThreshold: 30   # до ~150s на startup

readinessProbe:
  httpGet:
    path: /readyz
    port: http
```

> **Не делайте DB/Kafka availability liveness condition.** Иначе outage зависимости превращается в mass restart storm.

Практика: [Showcase 01](../../showcases/01-internal-rest-service/README.md).

## Для аналитика, разработчика и тестировщика

### Аналитик

Определяет, что означает “сервис готов” с точки зрения user flow, но не превращает каждую внешнюю dependency в liveness requirement.

### Разработчик

Разделяет startup, liveness и readiness endpoints по смыслу и не использует один “универсальный health” без анализа последствий.

### Тестировщик

Проверяет:
- slow startup;
- dead process;
- temporary NotReady;
- dependency outage;
- EndpointSlice removal;
- recovery без unnecessary restart.

### Перед следующей главой

Вы должны уметь ответить:

> В каком случае Pod должен быть перезапущен, а в каком достаточно временно убрать его из traffic?

Проверено: 2026-09-20.

Probe — это не просто URL в YAML. Это контракт между kubelet, Spring Boot и Service о состоянии процесса.

## 1. Три вопроса

```text
startupProbe  -> приложение уже стартовало?
livenessProbe -> процесс нужно перезапустить?
readinessProbe-> можно давать новый traffic?
```

Если эти вопросы смешать, Kubernetes начнёт принимать неправильные решения.

## 2. Lifecycle Spring Boot

Упрощённо:

```text
JVM start
 -> SpringApplication
 -> ApplicationContext
 -> beans
 -> web server
 -> startup runners
 -> application ready
 -> traffic
 -> SIGTERM
 -> graceful shutdown
 -> context close
```

Probe settings должны соответствовать реальному lifecycle.

## 3. Actuator

Добавьте dependency:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Configuration:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true
```

Это позволяет использовать liveness/readiness health groups и дополнительные paths на основном server port.

## 4. Почему liveness не должна проверять DB

Сценарий:

```text
PostgreSQL outage
 -> liveness DOWN
 -> kubelet restarts every Pod
 -> all JVMs reconnect
 -> DB receives reconnect storm
```

Kubernetes не чинит PostgreSQL рестартом Java.

Liveness — «сам process irrecoverably broken?»

## 5. Readiness и dependencies

Readiness может учитывать необходимые условия обслуживания traffic, но осторожно.

Если каждую краткую ошибку external API превращать в NotReady, все Pods могут одновременно выйти из Service.

Нужно решить:
- dependency обязательна для всех endpoints?
- можно ли деградировать?
- нужен circuit breaker?
- какой failure duration допустим?

## 6. Startup probe

Startup probe особенно полезна при длинном Spring startup.

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 5
  failureThreshold: 30
```

До успешного startup kubelet не применяет обычную liveness logic так, чтобы преждевременно убивать медленный startup.

## 7. Почему initialDelay — слабее startupProbe

```yaml
livenessProbe:
  initialDelaySeconds: 120
```

фиксированно ждёт 120 секунд независимо от реального startup.

Startup probe заканчивает startup phase сразу после успеха и лучше выражает intent.

## 8. timeoutSeconds

Слишком короткий timeout даёт false negative при:
- CPU throttling;
- GC pause;
- transient load.

Слишком длинный замедляет detection.

Настройка должна опираться на измерения.

## 9. failureThreshold

```text
period=10s
failureThreshold=3
```

означает примерно 30 секунд repeated failures до действия, не учитывая другие детали timing.

Это tolerance, а не магическое число.

## 10. Readiness и EndpointSlice

```text
readiness false
 -> Pod Ready=False
 -> endpoint condition not ready
 -> Service stops using it as normal ready backend
```

Проверить:

```bash
kubectl get pod
kubectl get endpointslice
kubectl describe pod <pod>
```

## 11. Running != Ready

Pod может быть Running, но `READY 0/1`.

Это значит:
- process жив;
- Kubernetes не считает его готовым к Service traffic.

Это нормальное transitional/degraded состояние.

## 12. Custom health indicators

Не надо превращать health endpoint в дорогое distributed transaction.

Health check должен быть:
- быстрым;
- локально объяснимым;
- безопасным под частым polling.

## 13. Failure practice: wrong readiness path

```yaml
readinessProbe:
  httpGet:
    path: /broken
    port: http
```

Проверить:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice
```

Expected: Running, NotReady.

## 14. Failure practice: wrong liveness

Сломайте liveness path.

```bash
kubectl get pod -w
kubectl logs <pod> --previous
```

Expected: restart count растёт.

## 15. Failure practice: slow startup

Искусственно задержите startup. Без startupProbe liveness может начать убивать process раньше завершения startup.

Добавьте startupProbe и сравните.

## 16. Probe vs monitoring

Probe решает локальное operational action.

Monitoring отвечает на более широкий вопрос:
- error rate;
- latency;
- saturation;
- business availability.

Не используйте liveness endpoint вместо полноценного monitoring.

## 17. Security

Health endpoints не должны раскрывать:
- passwords;
- connection strings;
- sensitive environment details.

External exposure Actuator endpoints должно быть осознанным.

## 18. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| health semantics | ✓ | | |
| Actuator config | ✓ | | |
| probe timings | | | ✓ |
| monitoring | | | ✓ |
| traffic routing | | ✓ | ✓ |

## 19. CKAD

Нужно писать startup/liveness/readiness probes и диагностировать failures. Production добавляет Spring lifecycle semantics, degraded modes и dependency design.

## 20. Checklist

- startup отдельно от liveness;
- liveness не зависит от external DB;
- readiness соответствует traffic ability;
- main port действительно проверяется;
- timeout/failureThreshold измерены;
- health endpoint быстрый;
- Actuator не раскрывает sensitive details.

## Связанные production-like примеры

- [Internal REST — readiness -> EndpointSlice](../../showcases/01-internal-rest-service/README.md)
- [Public API — edge routing + Ready backends](../../showcases/02-public-api-ingress-tls/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
- https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/

Проверено: **2026-09-20**.
