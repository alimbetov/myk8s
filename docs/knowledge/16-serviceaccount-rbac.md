# 16 — ServiceAccount and RBAC

Проверено: 2026-09-20.

ServiceAccount и RBAC нужны, когда workload взаимодействует с Kubernetes API. Обычному Spring Boot API это часто вообще не требуется.

## 1. Identity

Pod может запускаться от ServiceAccount:

```yaml
spec:
  serviceAccountName: orders
```

Это Kubernetes workload identity, не business user.

## 2. Default ServiceAccount

Если явно не указать, Pod использует default ServiceAccount namespace.

Плохая привычка — выдавать broad permissions default account.

Лучше отдельный ServiceAccount per workload role.

## 3. Disable token automount

Если Kubernetes API не нужен:

```yaml
automountServiceAccountToken: false
```

Это уменьшает credential exposure.

## 4. Role

Namespace-scoped permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["orders-runtime"]
    verbs: ["get"]
```

## 5. RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orders-config-reader
subjects:
  - kind: ServiceAccount
    name: orders
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: config-reader
```

## 6. ClusterRole

ClusterRole может описывать cluster-scoped или reusable permissions.

Не использовать ClusterRoleBinding, если namespace Role достаточно.

Least privilege начинается с минимального scope.

## 7. Secret access

Очень чувствительное permission:

```yaml
resources: ["secrets"]
verbs: ["get", "list"]
```

Если приложению нужен один password, лучше inject Secret, чем давать API permission читать Secrets.

`list secrets` фактически даёт доступ к большому объёму credentials.

## 8. kubectl auth can-i

```bash
kubectl auth can-i get configmaps   --as=system:serviceaccount:default:orders

kubectl auth can-i list secrets   --as=system:serviceaccount:default:orders
```

Это обязательный troubleshooting/security command.

## 9. Когда Spring app действительно нужен Kubernetes API

Примеры:
- operator/controller;
- dynamic leader election через Kubernetes lease;
- platform integration;
- reading specific CRD.

Для обычного CRUD backend обычно нет.

## 10. RBAC != application authorization

RBAC Kubernetes отвечает:

```text
может ли ServiceAccount вызвать Kubernetes API verb/resource
```

Spring Security отвечает:

```text
может ли user/service вызвать business endpoint
```

Не смешивать.

## 11. Token rotation

Projected ServiceAccount tokens имеют modern short-lived behavior compared with legacy long-lived Secret tokens. Не создавайте legacy SA token Secrets без необходимости.

## 12. Failure practice: forbidden

Приложение/Pod пытается Kubernetes API без permission.

Expected:

```text
403 Forbidden
```

Проверить `kubectl auth can-i`.

## 13. Failure practice: wrong ServiceAccount

Deployment ссылается не на тот SA.

```bash
kubectl get pod -o jsonpath='{.spec.serviceAccountName}'
kubectl describe rolebinding
```

## 14. Security audit

Проверять:
- wildcard verbs;
- wildcard resources;
- cluster-admin bindings;
- secret access;
- default ServiceAccount privileges;
- unused bindings.

## 15. Anti-patterns

- cluster-admin для приложения;
- wildcard `*`;
- list/watch secrets без причины;
- один SA на весь namespace;
- confused Kubernetes RBAC with business roles.

## 16. Developer vs Platform

| Область | Developer | Platform/Security | Shared |
|---|---:|---:|---:|
| declare API need | ✓ | | |
| RBAC policy | | ✓ | ✓ |
| ServiceAccount | | | ✓ |
| security audit | | ✓ | |
| app authorization | ✓ | | |

## 17. CKAD

ServiceAccount, Role, RoleBinding, SecurityContext и `auth can-i` — важная практическая зона.

## Связанные production-like примеры

- [ServiceAccount without API token](../../showcases/01-internal-rest-service/README.md)
- [Least-privilege RBAC manifest example](../../examples/security/rbac/serviceaccount-role-binding.yaml)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://kubernetes.io/docs/concepts/security/service-accounts/
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/

Проверено: **2026-09-20**.
