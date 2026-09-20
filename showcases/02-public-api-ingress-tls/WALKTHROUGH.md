# Walkthrough — Public API / Ingress / TLS

## Архитектура

```text
Internet
  |
 HTTPS
  v
Ingress Controller
  |
  v
Ingress rule
  |
  v
ClusterIP Service
  |
  v
Ready Spring Pods
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [Spring proxy config](application.yaml)
- [Ingress reference](../../docs/knowledge/manifests/ingress.md)

## Почему Service остаётся ClusterIP

Public exposure делает Ingress controller.

Backend Service не обязан быть LoadBalancer.

Это уменьшает number of public entry points.

## Ingress backend mapping

```text
Ingress backend service.name
      =
Service.metadata.name

Ingress backend port.name
      =
Service.spec.ports[].name
```

## TLS

```text
api.example.kz
 -> tls.hosts
 -> Secret api-example-tls
 -> controller termination
```

Certificate должен соответствовать hostname.

## Forwarded headers

После TLS termination Spring может видеть internal HTTP connection.

Поэтому proxy/forward-header configuration важна для:
- redirects;
- generated URLs;
- security scheme detection.

## Failure scenarios

### Wrong IngressClass
Resource существует, controller ignores it.

### TLS Secret bad
HTTPS fails before Spring.

### Backend Service missing
Ingress controller cannot route.

### Service has no endpoints
Ingress healthy, backend unavailable.

## Проверка

```bash
kubectl get ingress
kubectl describe ingress orders-api
kubectl get ingressclass
kubectl get svc orders-api
kubectl get endpointslice
```

## Production

Controller-specific:
- timeouts;
- body limit;
- auth;
- rate limit;
- WAF
нужно документировать отдельно от portable Ingress API.
