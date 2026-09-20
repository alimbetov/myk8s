# Manifest Reference — NetworkPolicy

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Purpose
L3/L4 isolation for selected Pods.

## apiVersion / kind
- apiVersion: networking.k8s.io/v1
- kind: NetworkPolicy

## metadata
- name
- namespace

## spec core
- podSelector
- policyTypes
- ingress
- egress

## peer selectors
- podSelector
- namespaceSelector
- ipBlock
- ipBlock.cidr
- ipBlock.except

## port rules
- protocol
- port
- endPort

## Concepts
- default deny
- additive policies
- ingress isolation
- egress isolation
- DNS egress
- CNI enforcement dependency

## Security boundary
NetworkPolicy != authentication/authorization.

## Troubleshooting
```bash
kubectl get networkpolicy
kubectl describe networkpolicy <name>
kubectl exec <pod> -- nslookup <service>
kubectl exec <pod> -- curl -v <url>
```

## Sources
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
