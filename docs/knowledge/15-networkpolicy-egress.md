# 15 — NetworkPolicy and egress

Проверено: 2026-09-20.

NetworkPolicy нужна не для authentication, а для ограничения network reachability.

## 1. Default Kubernetes network model

Без isolating NetworkPolicy Pods обычно могут общаться свободно в рамках возможностей cluster network.

Поэтому production часто движется к allow-list model.

## 2. Default deny ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]
```

## 3. Default deny egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes: [Egress]
```

Это серьёзное изменение: приложение больше не сможет даже DNS lookup без allow rule.

## 4. DNS egress

Нужно разрешить DNS к cluster DNS according to actual labels/namespace/CNI setup.

Не копируйте policy из интернета вслепую: CoreDNS labels и namespace должны совпасть с вашим cluster.

Диагностика:

```bash
kubectl get pods -n kube-system --show-labels
kubectl get svc -n kube-system
```

## 5. Allow application dependency

Например orders -> customer-api.

Лучше использовать namespace/pod labels, чем IP addresses, где это возможно.

## 6. External egress

Внешние SaaS/API сложнее:
- DNS names динамичны;
- standard NetworkPolicy работает в основном с IP/CIDR/selectors, не как FQDN firewall.

Некоторые CNI дают FQDN policies — это vendor-specific extension.

## 7. Database egress

Можно разрешить:
- only database namespace;
- only DB port.

Это уменьшает blast radius.

## 8. Ingress + app auth

Policy разрешает packet, Spring Security решает identity/authorization.

Оба слоя нужны для defense in depth.

## 9. CNI requirement

NetworkPolicy работает только если network plugin её реализует.

Manifest может примениться, но фактическое enforcement зависит от CNI.

Это нужно проверять platform documentation/testing.

## 10. Additive model

Multiple NetworkPolicies складывают разрешения. Они не работают как ordered firewall rules сверху вниз.

Это важное отличие от классического firewall mindset.

## 11. Failure practice: DNS blocked

1. Apply default-deny egress.
2. `nslookup customer-api`.
3. Observe timeout.
4. Add DNS allow.
5. Verify.

Root cause: DNS path, а не missing Service.

## 12. Failure practice: app port blocked

DNS resolves, curl times out.

```bash
kubectl exec debug -- nslookup customer-api
kubectl exec debug -- curl --connect-timeout 2 http://customer-api:8080
```

Это показывает layering.

## 13. Node/local exceptions

NetworkPolicy semantics вокруг host networking/node traffic/CNI могут иметь нюансы. Для production обязательно тестировать на используемом CNI.

## 14. Policy design method

Сначала inventory flows:

```text
orders -> DNS
orders -> postgres:5432
orders -> customer:8080
orders -> oidc:443
orders -> telemetry endpoint
```

Потом deny/allow.

Не начинайте с deny без карты dependencies.

## 15. Observability

Полезны:
- flow logs CNI;
- denied connection metrics;
- application timeouts;
- DNS metrics.

Без network observability deny policies трудно сопровождать.

## 16. Anti-patterns

- allow all egress навсегда;
- IP Pod rules;
- забыть DNS;
- считать policy authentication;
- копировать selectors, не проверяя labels;
- применять policy без CNI enforcement test.

## 17. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| dependency inventory | ✓ | | ✓ |
| CNI | | ✓ | |
| base deny policy | | ✓ | |
| app-specific allow | | | ✓ |
| debugging | ✓ | ✓ | ✓ |

## 18. CKAD

Нужно понимать podSelector, namespaceSelector, ports, ingress/egress, default deny.

## Связанные production-like примеры

- [Default deny and DB egress](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)
- [Caller-specific ingress](../../showcases/06-service-to-service-security/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://kubernetes.io/docs/concepts/services-networking/network-policies/

Проверено: **2026-09-20**.
