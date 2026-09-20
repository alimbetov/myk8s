# Walkthrough — Service-to-service security

## Архитектура

```text
orders-api
   |
   | DNS + TCP
   | bearer token
   v
payment-api
```

## Файлы

- [README](README.md)
- [Manifest](all.yaml)
- [Spring security/client config](application.yaml)
- [NetworkPolicy reference](../../docs/knowledge/manifests/networkpolicy.md)

## Security layers

```text
NetworkPolicy -> can packet reach?
TLS          -> encrypted transport?
mTLS         -> workload peer identity?
JWT          -> application identity?
Authorization-> allowed action?
```

Не заменяйте один слой другим.

## NetworkPolicy mapping

Target:

```yaml
podSelector:
  matchLabels:
    app: payment-api
```

Allowed caller:

```yaml
from:
  - podSelector:
      matchLabels:
        app: orders-api
```

Это label identity для network selection, не cryptographic identity.

## Service discovery

Orders calls:

```text
http://payment-api:8080
```

DNS -> Service -> EndpointSlice -> Ready payment Pods.

## JWT

Payment Spring Resource Server проверяет issuer/signature/claims согласно configured IdP.

Failure layers:

```text
DNS error        -> discovery
connect timeout  -> network
TLS error        -> transport
401              -> authentication
403              -> authorization
```

## Failure scenarios

- wrong Service name -> DNS;
- NetworkPolicy deny -> timeout;
- token absent/invalid -> 401;
- insufficient authority -> 403;
- payment NotReady -> no ready backend.

## Проверка

```bash
kubectl get svc payment-api
kubectl get endpointslice
kubectl get networkpolicy
kubectl exec <orders-pod> -- nslookup payment-api
```

## Production

Добавьте:
- real OIDC client credential flow/token exchange;
- TLS/mTLS if required;
- audience/scopes;
- secret rotation;
- auth metrics/logging without raw tokens.
