# 18 — Ingress, Gateway API and TLS

Проверено: 2026-09-20.

Эта глава про входящий traffic извне cluster и границу между application routing и platform ingress infrastructure.

## 1. Internal vs external

Internal:

```text
orders -> customer Service
```

External:

```text
user/client
 -> external LB
 -> ingress/gateway data plane
 -> Service
 -> Pod
```

Не все Services должны быть externally reachable.

## 2. Service type ClusterIP

Default internal Service.

Для application backend это основной building block behind ingress/gateway.

## 3. LoadBalancer

```yaml
type: LoadBalancer
```

может provision external load balancer depending on environment.

Не нужно делать каждый microservice LoadBalancer.

## 4. Ingress

Ingress historically provides HTTP/HTTPS routing:

```text
host/path
 -> Service
```

Но сам Ingress resource требует Ingress Controller.

Manifest без controller ничего полезного не реализует.

## 5. Gateway API

Gateway API — более современная extensible model:
- GatewayClass;
- Gateway;
- HTTPRoute;
- policy attachment ecosystem.

Она лучше разделяет infrastructure owner и application route owner.

## 6. Responsibility split

```text
Platform:
GatewayClass/Gateway/listeners/certs/policies

Application team:
HTTPRoute to its Service
```

Это часто удобнее shared enterprise ingress.

## 7. TLS termination

Common pattern:

```text
client HTTPS
 -> gateway terminates TLS
 -> HTTP or HTTPS internal
 -> Service
```

Если compliance требует encryption end-to-end, internal hop тоже TLS/mTLS.

## 8. Certificate

TLS requires:
- private key;
- certificate;
- chain;
- hostname/SAN;
- rotation;
- expiry monitoring.

Certificate Secret — sensitive.

## 9. Redirect HTTP -> HTTPS

Production public endpoints обычно enforce HTTPS.

Exact config depends on controller/Gateway implementation.

Не копируйте annotations одного ingress controller в другой.

## 10. Host/path routing

Example conceptual routes:

```text
api.example.kz/orders -> orders
api.example.kz/customers -> customer
```

Следите за:
- path rewrite;
- forwarded headers;
- context path;
- OpenAPI URLs.

## 11. Spring forwarded headers

За reverse proxy application должна корректно понимать original scheme/host where needed.

Проверяйте Spring Boot proxy/forward header configuration в вашей deployment architecture.

## 12. Timeouts

Ingress/gateway имеет собственные:
- connect;
- request;
- idle;
- body size;
- upload limits.

Они должны согласовываться с application/client timeouts.

## 13. Large uploads

Если Spring API принимает file upload:
- ingress body limit;
- Spring multipart limits;
- request timeout;
- object storage design
должны быть согласованы.

## 14. WebSocket/SSE/long polling

Имеют особые timeout/drain implications. Проверяйте controller/data-plane support.

## 15. Authentication location

Возможны:
- auth в application;
- auth proxy/gateway;
- оба уровня.

Gateway auth не всегда заменяет business authorization внутри Spring.

## 16. Failure practice: wrong Service port

Route ведёт на Service port, которого нет.

Проверить:
- route status;
- Service;
- EndpointSlice;
- controller logs/events.

## 17. Failure practice: certificate expired

DNS и TCP работают, но browser/client TLS fails.

Это PKI layer.

## 18. Failure practice: backend NotReady

Gateway healthy, route valid, но Service endpoints пусты.

Диагностика должна дойти до readiness/application.

## 19. Observability

Нужны metrics:
- request count;
- status;
- latency;
- TLS handshake;
- upstream failures;
- backend selection.

Correlation ID желательно прокидывать в application.

## 20. Ingress vs Gateway choice

Не надо переписывать рабочий Ingress только ради novelty.

Gateway API особенно полезен, когда нужны:
- clearer role separation;
- richer routing;
- shared infrastructure;
- portable modern API model.

## 21. Anti-patterns

- NodePort как постоянный public API без причины;
- LoadBalancer на каждый internal service;
- controller-specific annotation treated as Kubernetes standard;
- cert without rotation;
- public Actuator endpoints;
- internal calls через public endpoint.

## 22. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| API route | ✓ | | ✓ |
| gateway controller | | ✓ | |
| DNS/cert | | ✓ | ✓ |
| auth | ✓ | | ✓ |
| timeouts/body limits | | | ✓ |

## 23. CKAD

Ingress basics входят в application networking practice. Gateway API может зависеть от актуальной экзаменационной curriculum/version; production нужно знать оба подхода.

## Связанные production-like примеры

- [Ingress + TLS](../../showcases/02-public-api-ingress-tls/README.md)
- [Gateway API + several microservices](../../showcases/07-gateway-microservices/README.md)
- [Weighted HTTPRoute canary](../../showcases/09-canary/README.md)

Для каждого стенда откройте `README.md` → `WALKTHROUGH.md` → `all.yaml`.

## Sources

- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.io/docs/concepts/services-networking/gateway/

Проверено: **2026-09-20**.
