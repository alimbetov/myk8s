# Manifest Reference — NetworkPolicy

## Учебная схема объекта

### Где NetworkPolicy работает

```text
caller Pod
    |
    | network packet
    v
NetworkPolicy decision
    |
    v
target Pod
    |
    v
Spring Security / application auth
```

NetworkPolicy — L3/L4 reachability. Она не знает JWT, role пользователя или business permission.

### Annotated default deny

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress
spec:
  podSelector:
    matchLabels:
      app: orders

  policyTypes:
    - Egress

  egress: []        # deny all egress, включая DNS
```

### Диагностическая мысль

```text
DNS fails       -> egress/DNS rule?
DNS works
TCP timeout     -> network policy / port?
HTTP 401/403    -> уже application security
```

Практика: [service-to-service security](../../../showcases/06-service-to-service-security/annotated.yaml).

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Назначение

`NetworkPolicy` ограничивает L3/L4 traffic к/от выбранных Pods.

Она не:
- аутентифицирует user/service;
- не проверяет JWT;
- не делает TLS;
- не является business authorization.

## 2. API

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
```

## 3. Default deny ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Пустой `podSelector: {}` выбирает все Pods namespace.

Пустой ingress rule list означает: selected Pods изолированы по ingress, а ingress traffic не разрешён этой policy.

## 4. Default deny egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

После этого DNS и external dependencies тоже могут перестать работать, пока нет allow rules.

## 5. spec.podSelector

Выбирает Pods, к которым применяется policy.

```yaml
podSelector:
  matchLabels:
    app: orders
```

Пустой selector выбирает все Pods namespace.

## 6. policyTypes

Допустимые значения:
- `Ingress`
- `Egress`

Поле определяет направления isolation.

## 7. ingress rules

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: gateway
    ports:
      - protocol: TCP
        port: 8080
```

Traffic разрешён, если соответствует хотя бы одному rule среди всех policies, выбирающих Pod.

Policies additive, не ordered firewall chain.

## 8. egress rules

```yaml
egress:
  - to:
      - podSelector:
          matchLabels:
            app: customer-api
    ports:
      - protocol: TCP
        port: 8080
```

## 9. NetworkPolicyPeer.podSelector

Если указан только `podSelector`, он выбирает Pods **в namespace самой policy**.

Это частая ошибка при cross-namespace design.

## 10. namespaceSelector

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: crm
```

Выбирает namespaces по labels.

Пустой `namespaceSelector: {}` выбирает все namespaces.

## 11. podSelector + namespaceSelector вместе

```yaml
from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: crm
    podSelector:
      matchLabels:
        app: customer-api
```

Это Pods `app=customer-api` в выбранных namespaces.

Если selectors записаны двумя отдельными list items, semantics будет OR, а не AND.

## 12. ipBlock

```yaml
ipBlock:
  cidr: 10.20.0.0/16
  except:
    - 10.20.5.0/24
```

CIDR-based peer.

NAT/order behavior может зависеть от network implementation; production нужно проверять CNI specifics.

## 13. ports[].protocol

Допустимые:
- TCP
- UDP
- SCTP

**Default: TCP.**

## 14. ports[].port

Может быть numeric или named Pod port.

Если port отсутствует, rule соответствует всем ports указанного protocol context.

## 15. endPort

Разрешает range от `port` до `endPort` inclusive.

Constraints:
- `port` должен быть numeric;
- `endPort >= port`;
- нельзя использовать с named port.

## 16. Additive semantics

```text
Policy A allows gateway:8080
Policy B allows monitoring:9090
=> разрешены оба flow
```

Нет first-match / deny-rule precedence как в классическом firewall.

## 17. Isolation semantics

Pod становится isolated для direction, когда policy выбирает его и включает соответствующий policy type.

Для успешного flow должны быть разрешены applicable ingress/egress sides.

## 18. CNI dependency

API Server может принять NetworkPolicy manifest, но enforcement требует CNI/plugin с поддержкой NetworkPolicy.

Проверка должна быть functional, а не только:

```bash
kubectl get networkpolicy
```

## 19. DNS egress

Default-deny egress часто ломает DNS.

Нужно разрешить cluster DNS по фактическим labels/IP/namespace вашего cluster.

Не копировать CoreDNS policy вслепую.

## 20. Production flow inventory

До deny policy составить:

```text
orders -> DNS
orders -> customer-api:8080
orders -> postgres:5432
orders -> OIDC:443
orders -> telemetry
```

## 21. Failure — DNS blocked

```bash
kubectl get networkpolicy
kubectl exec <pod> -- nslookup customer-api
kubectl get svc -n kube-system
```

## 22. Failure — TCP blocked

DNS resolves but connect timeout.

```bash
kubectl exec <pod> -- nslookup customer-api
kubectl exec <pod> -- curl -v --connect-timeout 2 http://customer-api:8080
```

## 23. Failure — wrong namespace selector

```bash
kubectl get ns --show-labels
kubectl describe networkpolicy <name>
```

## 24. Failure — OR instead of AND

```yaml
from:
  - namespaceSelector: ...
  - podSelector: ...
```

Это два peer alternatives.

Если нужен intersection — оба selectors в одном peer object.

## 25. Security relationship

```text
NetworkPolicy -> reachability
TLS           -> transport
mTLS          -> workload identity
JWT/OAuth2    -> application identity
Authorization -> permission
```

## 26. Day-2

```bash
kubectl get networkpolicy
kubectl describe networkpolicy <name>
kubectl get pods --show-labels
kubectl get ns --show-labels
kubectl exec <debug> -- nslookup <service>
kubectl exec <debug> -- curl -v <url>
```

## 27. Anti-patterns

- allow-all forever;
- assume enforcement without CNI support;
- IP policy for ephemeral Pods;
- forget DNS;
- use policy as authentication;
- confuse OR/AND selector structure;
- strict egress without dependency inventory.

## 28. CKAD

Must know podSelector, namespaceSelector, ingress/egress, policyTypes, ports, default deny.

## Production-like examples

- [Egress isolation](../../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)
- [Service-to-service ingress allow-list](../../../showcases/06-service-to-service-security/README.md)

Каждый showcase содержит `WALKTHROUGH.md` с разбором mapping'ов и `all.yaml` с полной композицией.

## Sources

- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/network-policy-v1/
