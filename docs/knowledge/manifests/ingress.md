# Manifest Reference — Ingress

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Статус API

`Ingress` stable с Kubernetes 1.19, но API **frozen**: Kubernetes project рекомендует Gateway API для новых advanced designs.

Ingress не планируется удалять, но новая функциональность развивается в Gateway API ecosystem.

## 2. Назначение

Ingress описывает HTTP/HTTPS routing from external entry point to Services.

```text
client
 -> load balancer / ingress controller
 -> Ingress routing rules
 -> Service
 -> EndpointSlice
 -> Pod
```

Ingress resource сам traffic не обрабатывает: нужен Ingress Controller.

## 3. API

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
```

## 4. Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders
spec:
  ingressClassName: traefik

  tls:
    - hosts:
        - api.example.kz
      secretName: api-example-tls

  rules:
    - host: api.example.kz
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: orders
                port:
                  number: 8080
```

## 5. ingressClassName

Выбирает IngressClass/controller implementation.

Если omitted, cluster должен иметь appropriate default IngressClass для predictable behavior.

Некоторые controllers historically process class-less Ingresses, но Kubernetes рекомендует explicit/default class configuration.

## 6. IngressClass

Separate cluster-scoped resource, связывающий class name с controller implementation/config.

```bash
kubectl get ingressclass
```

## 7. rules

List host/path routing rules.

Если request соответствует host/path, controller route к configured backend.

## 8. rules[].host

Optional hostname.

Если host отсутствует, rule может catch traffic across hosts according to controller semantics/rule evaluation.

Production public APIs обычно задают explicit DNS hosts.

## 9. http.paths[].path

HTTP path для match.

Example:
- /
- /orders
- /api/v1

Matching semantics зависят от `pathType`.

## 10. pathType

Required для each path в networking.k8s.io/v1 Ingress.

Values:
- Exact
- Prefix
- ImplementationSpecific

## 11. Exact

Case-sensitive exact URL path match.

`/orders` не matches `/orders/123`.

## 12. Prefix

Element-wise path prefix.

`/foo/bar` matches `/foo/bar/baz`, но не `/foo/barbaz`.

Case-sensitive.

## 13. ImplementationSpecific

Semantics определяет IngressClass/controller.

Portability ниже.

Использовать только когда controller-specific behavior нужен осознанно.

## 14. backend.service.name

Имя Kubernetes Service **в том же namespace Ingress** для normal Service backend reference.

Ingress routes to Service, not directly to Pod.

## 15. backend.service.port

Можно указать:
- number
- name

Например:

```yaml
port:
  name: http
```

Named Service ports уменьшают numeric duplication.

## 16. defaultBackend

Fallback backend для requests, не matched rules, если configured.

Если absent, controller-specific default handling applies.

## 17. TLS

```yaml
tls:
  - hosts:
      - api.example.kz
    secretName: api-example-tls
```

Secret обычно типа `kubernetes.io/tls`.

Ingress controller terminates TLS depending on implementation.

## 18. TLS Secret

Expected keys:
- tls.crt
- tls.key

Secret должен находиться в appropriate namespace according to Ingress/controller model; стандартный Ingress TLS reference — local Secret.

## 19. DNS

Ingress manifest не создаёт public DNS record автоматически в общем Kubernetes standard.

DNS может управляться:
- вручную;
- ExternalDNS;
- cloud/platform automation.

## 20. External Load Balancer

Controller implementation может использовать:
- Service LoadBalancer;
- node ports;
- host networking;
- cloud-specific LB.

Ingress spec не диктует конкретный data plane.

## 21. Annotations

Ingress implementations используют annotations для:
- rewrites;
- timeouts;
- auth;
- body size;
- TLS policies;
- canary и др.

Annotations **не являются portable Kubernetes standard** unless documented by implementation.

Не копировать NGINX annotation в Traefik/HAProxy/other controller.

## 22. Ingress NGINX note

Kubernetes project в 2026 отдельно предупредил о retirement Ingress-NGINX project и рекомендует планировать migration to maintained alternatives / Gateway API where applicable.

Это относится к конкретному controller project, а не к удалению Ingress API itself.

Для k3s default controller часто Traefik, поэтому сначала определить реально установленный controller.

## 23. Spring Boot forwarded headers

За reverse proxy application может получать:
- X-Forwarded-For
- X-Forwarded-Proto
- Forwarded

Spring configuration должна соответствовать trust model, особенно для redirects/security URL generation.

Не доверять forwarded headers от произвольного direct caller без proxy boundary.

## 24. Path rewrite

Ingress standard path match не равен rewrite.

Например route `/orders` -> Service может передать original path в backend.

Rewrite обычно controller-specific extension.

Spring context path/API design должен согласовываться с ingress behavior.

## 25. Timeout layers

Есть:
- client timeout;
- ingress proxy timeout;
- Spring server timeout;
- downstream timeout.

Controller-specific timeout annotation/configuration должна быть согласована end-to-end.

## 26. Request body limits

File upload может быть ограничен:
- ingress controller;
- Spring multipart settings;
- proxy/LB;
- application.

Failure 413 часто не Spring business error, а edge proxy limit.

## 27. WebSocket/SSE

Support/timeouts/draining зависят controller implementation.

Нужен explicit compatibility test.

## 28. Failure — no controller

Ingress object exists, но no implementation watches it.

```bash
kubectl get ingress
kubectl get ingressclass
kubectl get pods -A
```

Manifest accepted != traffic configured.

## 29. Failure — wrong ingressClassName

Controller ignores object / status not provisioned.

Check:
- ingress class;
- controller logs/events;
- Ingress status.

## 30. Failure — backend Service missing

Ingress route references nonexistent Service.

```bash
kubectl describe ingress orders
kubectl get svc orders
```

## 31. Failure — Service has no endpoints

Ingress/controller works, Service exists, but EndpointSlice empty/not ready.

```bash
kubectl get endpointslice -l kubernetes.io/service-name=orders
kubectl get pod
```

## 32. Failure — TLS Secret missing/invalid

Controller cannot configure HTTPS correctly.

Check:
- Secret exists;
- type/data keys;
- controller events/logs;
- certificate hostname/expiry.

## 33. Failure — pathType mismatch

Expected /foo/barbaz match under naive prefix expectation, but Prefix path semantics are element-based.

Use Exact/Prefix intentionally.

## 34. Security

- TLS certificates rotated;
- management/Actuator not unintentionally public;
- auth layer explicit;
- WAF/rate limit controller-specific;
- forwarded headers trust configured;
- Secret RBAC minimal.

## 35. Day-2

```bash
kubectl get ingress
kubectl describe ingress orders
kubectl get ingressclass
kubectl get svc orders
kubectl get endpointslice -l kubernetes.io/service-name=orders
kubectl get secret api-example-tls
```

Plus controller-specific logs/status.

## 36. Ingress vs Gateway API

Ingress:
- simple/frozen;
- broad existing support;
- controller annotations common.

Gateway API:
- successor;
- richer role separation/resources;
- active evolution;
- CRDs/implementation required.

Migration is architecture project, not blind YAML translation.

## 37. Anti-patterns

- Ingress without knowing controller;
- controller annotations treated as portable;
- public Actuator;
- TLS Secret no rotation;
- internal service calls via public ingress;
- rewrite assumptions;
- no backend readiness diagnostics;
- use retired/unmaintained controller without migration plan.

## 38. CKAD

Know:
- networking.k8s.io/v1;
- ingressClassName;
- rules/host/path;
- pathType;
- Service backend;
- TLS reference;
- describe/debug.

## Sources

- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/
- https://kubernetes.io/docs/concepts/services-networking/gateway/
