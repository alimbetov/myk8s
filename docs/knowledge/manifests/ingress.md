# Manifest Reference — Ingress

Проверено: 2026-09-20.

Статус: ITERATION 1 / STRUCTURE ONLY.

## Purpose
HTTP/HTTPS routing from external entry point to Services.

## apiVersion / kind
- apiVersion: networking.k8s.io/v1
- kind: Ingress

## metadata
- name
- namespace
- annotations

## spec core
- ingressClassName
- defaultBackend
- rules
- rules.host
- rules.http.paths
- path
- pathType
- backend.service.name
- backend.service.port
- tls
- tls.hosts
- tls.secretName

## pathType
- Exact
- Prefix
- ImplementationSpecific

## Related components
- Ingress Controller
- Service
- TLS Secret
- DNS
- external load balancer

## Important concept
Ingress resource without controller does not provide actual ingress traffic.

## Controller-specific area
Annotations are implementation-specific and must be documented separately.

## Troubleshooting
```bash
kubectl get ingress
kubectl describe ingress <name>
kubectl get ingressclass
kubectl get svc
kubectl get endpointslice
```

## Sources
- https://kubernetes.io/docs/concepts/services-networking/ingress/
