# Manifest Reference — ServiceAccount + RBAC

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

Покрывает:
- ServiceAccount
- Role
- ClusterRole
- RoleBinding
- ClusterRoleBinding

# Part A — ServiceAccount

## 1. Назначение

ServiceAccount — workload identity для Kubernetes API ecosystem. Это не Spring Security principal.

## 2. Manifest

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders
automountServiceAccountToken: false
```

## 3. default ServiceAccount

Если Pod не задаёт `serviceAccountName`, используется `default` ServiceAccount namespace.

Не выдавать broad permissions default SA.

## 4. automountServiceAccountToken

Для backend без Kubernetes API:

```yaml
automountServiceAccountToken: false
```

уменьшает credential exposure.

## 5. imagePullSecrets

```yaml
imagePullSecrets:
  - name: registry-credentials
```

Registry auth, не API RBAC permission.

# Part B — Role

## 6. API

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
```

Role namespaced.

## 7. Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: orders-config-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["orders-runtime"]
    verbs: ["get"]
```

## 8. rules[].apiGroups

Core API group:

```yaml
apiGroups: [""]
```

Deployment API:

```yaml
apiGroups: ["apps"]
```

## 9. resources

Plural API resource names:
- pods
- configmaps
- secrets
- deployments

Subresources могут задаваться отдельно, например `pods/log` или `deployments/scale`.

## 10. verbs

Common:
- get
- list
- watch
- create
- update
- patch
- delete

Wildcard `*` значительно расширяет permission.

## 11. resourceNames

Ограничивает rule named resources для поддерживаемых request forms.

Не использовать как универсальную замену namespace scope; list/watch и некоторые request patterns имеют дополнительные ограничения.

# Part C — ClusterRole

## 12. Назначение

ClusterRole — cluster-scoped logical grouping rules.

Может описывать:
- cluster resources;
- non-resource URLs;
- namespaced resources reusable across namespaces.

## 13. Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Без binding permissions никому не выданы.

## 14. aggregationRule

Позволяет controller агрегировать ClusterRole rules по labels.

Если используется aggregationRule, direct edits rules могут быть перезаписаны controller.

# Part D — RoleBinding

## 15. Назначение

RoleBinding grants permissions в namespace binding.

Может ссылаться:
- на Role того же namespace;
- на ClusterRole, но grant scope остаётся namespace RoleBinding.

## 16. Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orders-config-reader
subjects:
  - kind: ServiceAccount
    name: orders
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: orders-config-reader
```

## 17. subjects

Kinds:
- User
- Group
- ServiceAccount

ServiceAccount subject namespace должен быть указан корректно.

## 18. roleRef

Fields:
- apiGroup
- kind
- name

`roleRef` immutable; смена target role обычно требует recreate binding.

# Part E — ClusterRoleBinding

## 19. Назначение

ClusterRoleBinding grants ClusterRole cluster-wide.

Большой blast radius.

## 20. Scope decision

```text
Only one namespace?
 -> Role + RoleBinding

Reusable role, one namespace grant?
 -> ClusterRole + RoleBinding

Cluster-wide?
 -> ClusterRole + ClusterRoleBinding
```

## 21. Secret permissions

```yaml
resources: ["secrets"]
verbs: ["get", "list", "watch"]
```

крайне sensitive.

Для DB password лучше Secret injection, а не API permission приложению.

## 22. Pod linkage

```yaml
spec:
  serviceAccountName: orders
```

RBAC применяется к Kubernetes API requests identity ServiceAccount.

## 23. Projected tokens

Современные projected ServiceAccount tokens предпочтительнее legacy long-lived static token Secrets.

Не создавать legacy token Secret без explicit requirement.

## 24. auth can-i

```bash
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:payments:orders

kubectl auth can-i list secrets \
  --as=system:serviceaccount:payments:orders
```

## 25. Failure — Forbidden

HTTP 403 от Kubernetes API.

Проверить:
- actual ServiceAccount;
- namespace;
- RoleBinding;
- verb/resource;
- roleRef.

## 26. Failure — wrong binding namespace

RoleBinding grants only in its namespace. Role must be same namespace if roleRef.kind=Role.

## 27. Failure — excessive privileges

Runtime может работать, но security posture fails audit due to broad ClusterRoleBinding/wildcards.

## 28. nonResourceURLs

ClusterRole can grant:

```yaml
nonResourceURLs: ["/healthz"]
verbs: ["get"]
```

Non-resource URLs cluster-scoped by nature.

## 29. Security checklist

- unique workload SA where useful;
- no cluster-admin;
- no wildcards unless justified;
- avoid list/watch secrets;
- RoleBinding over ClusterRoleBinding where possible;
- token automount false if unused;
- audit bindings.

## 30. Day-2

```bash
kubectl get sa
kubectl get role,rolebinding
kubectl get clusterrole,clusterrolebinding
kubectl describe role <name>
kubectl describe rolebinding <name>
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>
```

## 31. CKAD

Must know ServiceAccount, Role, RoleBinding, serviceAccountName, auth can-i, namespaced vs cluster scope.

## Sources

- https://kubernetes.io/docs/concepts/security/service-accounts/
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
