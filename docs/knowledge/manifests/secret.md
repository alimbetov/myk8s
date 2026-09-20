# Manifest Reference — Secret

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Purpose
Хранение и доставка sensitive runtime data.

## apiVersion / kind
- apiVersion: v1
- kind: Secret

## metadata
- name
- namespace
- labels
- annotations

## data fields
- type
- data
- stringData
- immutable

## Common types
- Opaque
- kubernetes.io/tls
- kubernetes.io/dockerconfigjson
- service-account-token legacy/special cases

## Consumption
- secretKeyRef
- secretRef
- volume.secret
- projected volume
- imagePullSecrets

## Security topics
- base64 != encryption
- RBAC
- etcd encryption at rest
- rotation
- external secret managers
- log hygiene

## Runtime behavior
- env restart requirement
- mounted file updates
- missing secret/key
- optional refs

## Troubleshooting
```bash
kubectl get secret
kubectl describe secret <name>
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

## Sources
- https://kubernetes.io/docs/concepts/configuration/secret/
