# 12 — PostgreSQL connectivity and HikariCP

Проверено: 2026-09-20.

Эта глава про приложение-клиент PostgreSQL. Развёртывание самой HA базы будет отдельным stateful track.

## 1. Mental model

```text
Spring Boot Pod
 -> HikariCP
 -> DNS/Service endpoint
 -> PostgreSQL
```

На одном Pod Hikari pool выглядит локальной настройкой. В Kubernetes replicas превращают её в cluster-wide connection budget.

## 2. URL

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres-rw.database.svc:5432/orders
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Endpoint должен быть logical stable address, например operator-managed write Service.

## 3. Hikari maximumPoolSize

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
```

При 20 replicas:

```text
20 × 10 = 200 potential application connections
```

HPA до 50 replicas:

```text
50 × 10 = 500
```

Поэтому autoscaling нельзя проектировать без DB capacity.

## 4. connectionTimeout

```yaml
connection-timeout: 3000
```

Это сколько application thread готов ждать connection из pool.

Это не PostgreSQL TCP connect timeout и не query timeout.

Нужно различать уровни.

## 5. Pool saturation

```text
pool max 10
10 long transactions
11th request waits
...
request queue grows
```

Увеличить pool до 100 может ухудшить PostgreSQL, а не исправить root cause.

## 6. Little's Law intuition

Если queries/transactions медленные, throughput и concurrency связаны.

Первый вопрос при saturated pool:
- почему connection так долго занята?

А не:
- почему pool всего 10?

## 7. Transaction duration

Держать connection пока приложение делает HTTP call:

```text
BEGIN transaction
 -> SELECT
 -> call external API 5 sec
 -> UPDATE
 -> COMMIT
```

плохо: DB connection/locks удерживаются во время network wait.

Лучше границы transaction делать короткими, если business consistency позволяет.

## 8. Readiness

Нужно осторожно решать, делать ли DB частью readiness.

Если сервис совершенно бессмыслен без DB — readiness degradation может быть оправдан.

Но если все Pods одновременно выйдут NotReady во время краткого DB hiccup, Service станет без endpoints.

Это design decision, а не автоматическое правило.

Liveness к DB привязывать не следует.

## 9. Credential rotation

Env-based password:

```text
Secret updated
 -> existing JVM environment unchanged
 -> existing pool may keep old authenticated sessions
 -> new connections may fail after old credential revoked
```

Rotation должна учитывать:
- overlap credentials;
- restart/reload;
- pool connection recycling;
- verification;
- revoke old.

## 10. PostgreSQL failover

При primary failover existing TCP connections могут оборваться.

Application должна:
- получать finite error;
- освобождать broken connection;
- retry safe operation на appropriate level;
- подключаться через stable write endpoint.

Не hardcode primary Pod.

## 11. DNS and failover

Operator-managed Service обычно скрывает current primary behind stable DNS/ClusterIP.

Это проще для application, чем самостоятельное discovery primary.

## 12. Flyway/Liquibase startup

Если каждая из 30 replicas одновременно пытается выполнять migration, нужно понимать locking/coordination migration tool.

Для серьёзных releases часто migration отделяют в controlled pipeline/Job или тщательно используют single migration owner.

## 13. Schema compatibility

RollingUpdate означает v1 и v2 могут жить одновременно.

Поэтому:
- add nullable/new structures first;
- deploy compatible code;
- backfill/migrate;
- remove old structure позже.

## 14. Failure practice: wrong password

```bash
kubectl create secret generic orders-db   --from-literal=DB_USERNAME=orders   --from-literal=DB_PASSWORD=wrong   --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/orders
```

Проверить logs без утечки password.

## 15. Failure practice: DB unavailable

Остановите lab DB или заблокируйте egress.

Наблюдайте:
- connection acquisition;
- exception latency;
- retry behavior;
- readiness;
- liveness;
- Hikari metrics.

## 16. Failure practice: pool exhausted

Сделайте pool=2 и несколько long transactions.

Наблюдайте:
- pending threads;
- Hikari active/idle/pending;
- latency;
- connection timeout.

## 17. Metrics

Минимум:
- active connections;
- idle;
- pending;
- max;
- acquisition time;
- query/transaction duration;
- SQL errors;
- DB server connections.

## 18. Security

- password только Secret/external manager;
- TLS к DB where required;
- least-privilege DB role;
- separate migration/application privileges where useful;
- не логировать JDBC URL с credentials.

## 19. Anti-patterns

- pool=100 «на всякий случай»;
- long remote calls inside transaction;
- DB liveness;
- direct Pod IP primary;
- no connection timeout;
- HPA without DB capacity;
- breaking migration during rolling update.

## 20. Developer vs DBA/Platform

| Область | Developer | DBA/Platform | Shared |
|---|---:|---:|---:|
| pool sizing | ✓ | | ✓ |
| DB max connections | | ✓ | |
| transaction design | ✓ | | |
| failover endpoint | | ✓ | ✓ |
| migration | ✓ | | ✓ |
| credential rotation | | | ✓ |

## 21. CKAD

CKAD касается Secret, Service DNS, probes/config. Hikari/transactions/failover — production layer.

## 22. Checklist

- stable DB endpoint;
- finite acquisition/connect/query behavior;
- pool × max replicas рассчитан;
- DB capacity известна;
- transactions короткие;
- credential rotation tested;
- failover tested;
- migrations rolling-compatible;
- Hikari metrics exported.

## Связанные production-like примеры

- [Hikari × replicas × HPA](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)
- [CloudNativePG rw/ro Services](../../showcases/12-cloudnativepg-primary-replicas/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://docs.spring.io/spring-boot/reference/data/sql.html
- https://github.com/brettwooldridge/HikariCP
- https://www.postgresql.org/docs/

Проверено: **2026-09-20**.
