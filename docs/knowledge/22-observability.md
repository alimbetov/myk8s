# 22 — Observability: metrics, logs, traces and Kubernetes signals

Проверено: 2026-09-20.

Observability нужна, чтобы по внешним signals понять внутреннее состояние distributed system. В Kubernetes одного `kubectl logs` недостаточно.

## 1. Четыре источника

```text
metrics
logs
traces
Kubernetes events/state
```

Они отвечают на разные вопросы.

## 2. Metrics

Для HTTP полезна RED model:
- Rate;
- Errors;
- Duration.

Spring Boot + Micrometer/Actuator:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
```

External exposure endpoint должен быть secured according to platform design.

## 3. JVM metrics

Следить:
- heap;
- non-heap/metaspace;
- GC pauses/rate;
- threads;
- CPU;
- process uptime;
- file descriptors where relevant.

Container metrics:
- memory working set;
- CPU;
- throttling;
- OOM/restarts.

## 4. Dependency metrics

HTTP:
- latency/status/timeouts;
- pool pending.

DB/Hikari:
- active;
- idle;
- pending;
- max;
- acquisition.

Kafka:
- consumer lag;
- errors/rebalances.

Rabbit:
- queue depth;
- unacked;
- publish/consume rate.

## 5. Logs

В container environment предпочтительно stdout/stderr.

Structured logging:

```json
{
  "service":"orders",
  "version":"1.7.3",
  "traceId":"...",
  "level":"ERROR",
  "message":"..."
}
```

Не обязательно JSON everywhere, но fields должны быть queryable.

## 6. Sensitive logging

Не логировать:
- passwords;
- bearer token;
- cookies/session secrets;
- private keys;
- full card/account sensitive payloads.

Masking policy должна быть централизована.

## 7. Correlation ID

Distributed request:

```text
gateway
 -> orders
 -> customer
 -> payment
```

Без trace/correlation identifier logs превращаются в четыре независимых истории.

## 8. Distributed tracing

Trace показывает spans:

```text
HTTP /orders        420ms
  DB select          20ms
  customer HTTP      80ms
  payment HTTP      280ms
```

Это помогает найти latency contribution.

## 9. OpenTelemetry

Современная common model:
- instrumentation;
- OTLP;
- collector;
- backend.

Конкретный backend может быть Prometheus/Grafana/Tempo/Jaeger/Elastic/vendor platform.

Не привязывайте application design к одному UI.

## 10. Kubernetes state

Application metrics не покажут:
- Pending due scheduling;
- ImagePullBackOff;
- node pressure;
- PVC Pending.

Поэтому:

```bash
kubectl get
kubectl describe
kubectl get events
```

остаются важны.

## 11. Events не long-term audit log

Kubernetes Events имеют ограниченный lifecycle. Их нужно использовать для operational diagnosis, но не как долговечную incident database.

## 12. Golden signals

Классический набор:
- latency;
- traffic;
- errors;
- saturation.

Для business service добавьте domain signals:
- orders processed;
- payments failed;
- messages stuck.

## 13. SLI/SLO

SLI — измерение.

Пример:

```text
successful HTTP requests / all eligible requests
```

SLO — цель:

```text
99.9% successful over period
```

Alerts лучше связывать с user impact/SLO, а не только «CPU > 80».

## 14. Alert fatigue

Если alert постоянно шумит и не требует action, его перестают замечать.

Каждый production alert должен иметь:
- meaning;
- severity;
- owner;
- runbook;
- actionable condition.

## 15. Version visibility

При rollout полезно видеть application version/build metadata в metrics/logs.

Тогда можно сравнить:
- v1 error rate;
- v2 error rate.

Для canary это критично.

## 16. Probe metrics != SLO

Readiness success говорит «можно route traffic», а не «пользователи получают правильный результат с нужной latency».

Не заменяйте SLO probe status.

## 17. Failure practice: CPU throttling

Pod Ready, no restart, но p99 растёт.

Коррелируйте:
- throttling;
- CPU;
- latency.

Это пример failure, который `kubectl get pod` не объясняет.

## 18. Failure practice: pool saturation

Observe:
- Hikari pending;
- latency;
- DB transaction time.

Не увеличивайте pool до анализа root cause.

## 19. Failure practice: error after rollout

Сравните metrics by version/Pod.

```bash
kubectl rollout history deploy/orders
kubectl get pod -l app=orders -o wide
```

Rollback decision должен основываться на evidence.

## 20. Logs during crash

```bash
kubectl logs <pod> --previous
```

Central log store должен сохранить old container logs даже после Pod deletion.

## 21. Dashboards

Минимальный service dashboard:
- request rate;
- p50/p95/p99;
- 4xx/5xx;
- CPU/memory;
- restarts;
- Hikari;
- downstream calls;
- version/replicas.

## 22. Runbook link

Alert должен вести к конкретной странице:

```text
High 5xx orders-service
 -> docs/runbooks/orders-high-5xx.md
```

Тогда `myk8s` становится operational library.

## 23. Observability ownership

| Область | Developer | Platform/SRE | Shared |
|---|---:|---:|---:|
| application metrics | ✓ | | |
| log content | ✓ | | |
| collectors/backends | | ✓ | |
| dashboards | | | ✓ |
| SLO | | | ✓ |
| runbooks | | | ✓ |

## 24. CKAD

CKAD: logs, probes, resource visibility/basic troubleshooting. Production: metrics platform, traces, SLO, alerting, centralized logging.

## 25. Checklist

- metrics exported;
- logs centralized;
- traces/correlation available;
- version visible;
- no secrets logged;
- DB/client pools instrumented;
- Kubernetes state monitored;
- alerts actionable;
- runbooks linked;
- SLO defined for critical services.

## Sources

- https://docs.spring.io/spring-boot/reference/actuator/
- https://docs.micrometer.io/
- https://opentelemetry.io/docs/
- https://kubernetes.io/docs/tasks/debug/

Проверено: **2026-09-20**.
