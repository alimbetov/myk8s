# Manifest Reference — ServiceAccount + RBAC

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Objects
- ServiceAccount
- Role
- ClusterRole
- RoleBinding
- ClusterRoleBinding

## ServiceAccount core
- metadata.name
- automountServiceAccountToken
- imagePullSecrets

## Role/ClusterRole core
- rules
- apiGroups
- resources
- resourceNames
- verbs
- nonResourceURLs

## Binding core
- subjects
- roleRef

## Concepts
- namespace scope vs cluster scope
- least privilege
- wildcard risk
- Secret access sensitivity
- projected ServiceAccount token
- default ServiceAccount

## Pod link
- spec.serviceAccountName
- spec.automountServiceAccountToken

## Troubleshooting
```bash
kubectl auth can-i get configmaps --as=system:serviceaccount:<ns>:<sa>
kubectl get role,rolebinding,clusterrole,clusterrolebinding
kubectl describe rolebinding <name>
```

## Sources
- https://kubernetes.io/docs/concepts/security/service-accounts/
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
