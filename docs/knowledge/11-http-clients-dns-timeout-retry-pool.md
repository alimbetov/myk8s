# 11 — HTTP clients: DNS, timeout, retry and connection pool

Проверено: 2026-09-20.

Микросервис часто падает не из-за собственного controller, а из-за неправильного client behavior к другому сервису.

## 1. Mental model

```text
Spring service A
 -> DNS lookup
 -> TCP connection
 -> optional TLS
 -> HTTP request
 -> service B
```

На каждом этапе может быть отдельный timeout/failure.

## 2. URL

Внутри namespace:

```yaml
clients:
  customer:
    base-url: http://customer-api:8080
```

Не Pod IP.

## 3. DNS

DNS failure:

```text
UnknownHostException
```

Это отличается от connect timeout.

Проверить:

```bash
kubectl exec <pod> -- nslookup customer-api
kubectl get svc customer-api
```

## 4. Connect timeout

Сколько ждать установления network connection.

```text
DNS resolved
TCP cannot connect
 -> connect timeout
```

Причины:
- NetworkPolicy;
- target down;
- wrong port;
- route/firewall.

## 5. Read/response timeout

Connection установлено, но server долго не отвечает.

Это другой failure class.

Нужно иметь конечные значения обоих timeout.

## 6. Overall deadline

Если operation имеет 5-second business deadline, нельзя позволить:
- 3 retries × 5-second read timeout;
- плюс backoff;
- плюс queue wait.

Timeout budget нужно считать end-to-end.

## 7. Spring configuration contract

```yaml
app:
  customer-client:
    base-url: ${CUSTOMER_URL:http://customer-api:8080}
    connect-timeout: 1s
    read-timeout: 3s
    max-attempts: 2
```

```java
@ConfigurationProperties("app.customer-client")
public record CustomerClientProperties(
    URI baseUrl,
    Duration connectTimeout,
    Duration readTimeout,
    int maxAttempts) {}
```

## 8. RestClient/WebClient

Конкретная client library может отличаться, но contract должен оставаться явным:
- URL;
- connect timeout;
- response timeout;
- pool;
- retry policy;
- TLS.

Не прячьте эти настройки в random bean code.

## 9. Connection pool

Создание TCP/TLS connection на каждый request дорого.

Pool reuse уменьшает overhead.

Но pool имеет limits:
- max connections;
- pending acquisition;
- idle/keepalive;
- connection lifetime.

## 10. Pool saturation

```text
100 app threads
client pool max = 20
20 requests downstream
80 wait for connection
```

Если connection acquisition timeout бесконечный, thread/request backlog растёт.

## 11. Retry storm

```text
100 rps
3 extra attempts
dependency failing
=> up to ~400 attempts/sec total
```

Retry увеличивает нагрузку именно когда dependency слабая.

## 12. Когда retry допустим

Обычно лучше для:
- transient network errors;
- idempotent GET;
- safe idempotent operation с idempotency key.

Опасно retry:
- non-idempotent POST;
- payment/transfer без idempotency;
- validation 4xx;
- permanent auth failure.

## 13. Exponential backoff + jitter

```text
attempt 1
100ms
attempt 2
250ms
attempt 3
600ms + jitter
```

Jitter уменьшает synchronized retry wave между Pods.

## 14. Circuit breaker

Circuit breaker не «чинит» dependency. Он может быстро прекращать заведомо бесполезные calls после threshold failures.

Нужен только если semantics/traffic оправдывают. Не добавляйте Resilience4j mechanically everywhere.

## 15. Kubernetes readiness и client retry — разные layers

Readiness server B уменьшает routing к unhealthy Pods.

Но:
- вся service B может быть недоступна;
- external API может быть вне cluster.

Client всё равно обязан иметь timeouts.

## 16. DNS cache

JVM/HTTP stack имеет DNS caching behavior. Kubernetes Service ClusterIP обычно стабилен, поэтому caller часто резолвит Service, а backend Pods меняются за Service.

Headless services/stateful discovery требуют большего внимания к DNS TTL/cache.

## 17. Failure practice: wrong service name

```yaml
CUSTOMER_URL: http://customer-apix:8080
```

Expected:
- UnknownHost/DNS error.

Проверить nslookup.

## 18. Failure practice: policy deny

DNS resolves, но TCP timeout.

Проверить:
- NetworkPolicy;
- endpoints;
- curl connect timeout.

## 19. Failure practice: slow dependency

Добавьте delay 10s, client timeout 2s.

Проверьте:
- latency;
- exception;
- retry count;
- thread/pool saturation.

## 20. Observability

Client metrics:
- calls;
- latency;
- timeout count;
- status codes;
- pool leased/pending;
- retry count;
- circuit state.

Без этого distributed latency невозможно нормально расследовать.

## 21. Anti-patterns

- no timeout;
- 10 retries by default;
- retry POST blindly;
- Pod IP;
- unbounded pool;
- huge thread pool to hide slow dependency;
- liveness dependent on downstream;
- log Authorization headers.

## 22. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| client timeout | ✓ | | |
| retry semantics | ✓ | | |
| DNS/CNI | | ✓ | |
| NetworkPolicy | | | ✓ |
| TLS trust | | | ✓ |
| client metrics | ✓ | | ✓ |

## 23. CKAD

CKAD: Service DNS, connectivity, ConfigMap/env, NetworkPolicy. Production: timeout budgets, retries, pools, TLS, observability.

## Связанные production-like примеры

- [orders -> payment internal HTTP call](../../showcases/06-service-to-service-security/README.md)
- [Gateway + multiple HTTP services](../../showcases/07-gateway-microservices/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://docs.spring.io/spring-boot/reference/io/rest-client.html
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://kubernetes.io/docs/concepts/services-networking/service/

Проверено: **2026-09-20**.
