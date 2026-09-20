# Prompt 02 — Internal Services, Networking and Microservice Security

Цель — устранить монолитное мышление «если сервис внутренний, значит он защищен».

## Исследовать

### Service discovery
Объяснить:
- Pod IP;
- Service;
- ClusterIP;
- headless Service;
- EndpointSlice;
- CoreDNS;
- namespace-aware DNS;
- Ingress/Gateway только для необходимого внешнего трафика.

### Internal service design
Показать архитектуру:

```text
client -> ingress/gateway -> public API
                         -> internal Service A
                         -> internal Service B
                         -> database/message broker
```

Зафиксировать: `ClusterIP` делает сервис внутренним с точки зрения экспозиции, но сам по себе не является authentication/authorization boundary.

### Defense in depth
Разобрать отдельно:

1. L3/L4 isolation — NetworkPolicy.
2. Workload identity — ServiceAccount / external workload identity / service mesh identity.
3. Application authentication — JWT/OAuth2/OIDC or mTLS identity depending on architecture.
4. Application authorization — scopes/roles/permissions.
5. Transport security — TLS/mTLS.
6. Kubernetes API authorization — RBAC.
7. edge authentication — API Gateway/Ingress integration.

### Spring Boot
Показать:
- resource server;
- JWT validation;
- propagation vs token exchange trade-offs;
- internal client with timeout;
- circuit breaker/retry boundaries;
- correlation/trace IDs.

Не рекомендовать слепо прокидывать пользовательский access token через всю систему без анализа trust boundaries.

### NetworkPolicy lab
Сделать:
- default-deny ingress/egress;
- allow DNS;
- allow frontend -> backend;
- allow backend -> PostgreSQL;
- запрет frontend -> PostgreSQL;
- проверку через temporary debug Pod.

### Threat model
Минимум:
- compromised pod;
- leaked service credential;
- rogue pod in same namespace;
- unauthorized east-west call;
- SSRF into internal services;
- Kubernetes API abuse.

### Deliverables
- `docs/knowledge/02-services-networking-security.md`;
- `examples/network-security/`;
- `labs/02-network-policy/`.
