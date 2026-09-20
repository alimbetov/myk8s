# Walkthrough — Gateway API + several microservices

## Что строим

```text
https://api.example.kz/orders
        ↓
     Gateway
        ↓
    HTTPRoute
        ↓
 Service orders-api
        ↓
 EndpointSlice
        ↓
 Ready Pods
```

То же для `/customers`, но через другой HTTPRoute и Service.

## Файлы стенда

- [README](README.md)
- [Полный manifest](all.yaml)
- [Mapping cheat sheet](../MAPPING-CHEATSHEET.md)
- [Gateway/Ingress theory](../../docs/knowledge/18-ingress-gateway-api-tls.md)

## Responsibility chain

```text
GatewayClass   — implementation/platform
      ↓
Gateway        — listeners / edge
      ↓
HTTPRoute      — host/path routing
      ↓
Service        — stable backend identity
      ↓
EndpointSlice  — ready backend addresses
      ↓
Pods           — Spring Boot instances
```

### Важно

Gateway не знает Pod IP. Он направляет request в Service.

## GatewayClass

В `all.yaml`:

```yaml
gatewayClassName: replace-with-installed-class
```

Это placeholder. Сначала:

```bash
kubectl get gatewayclass
```

Если class не существует, Gateway object может быть создан, но data plane не будет запрограммирован.

## Listener

```yaml
listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: api.example.kz
```

Listener определяет входную точку Gateway.

## HTTPRoute parentRefs

```yaml
parentRefs:
  - name: public-api
    sectionName: http
```

Mapping:

```text
HTTPRoute.parentRefs.name
        =
Gateway.metadata.name

sectionName
        =
Gateway.listeners[].name
```

Это exact object reference.

## HTTPRoute backendRefs

```yaml
backendRefs:
  - name: orders-api
    port: 8080
```

Mapping:

```text
backendRefs.name
      =
Service.metadata.name

backendRefs.port
      =
Service.spec.ports[].port
```

### Не путать

Это не `containerPort`.

Дальше Service сам сопоставляет:

```text
Service.port
  ↓
Service.targetPort
  ↓
containerPort
```

## Service selector

```yaml
selector:
  app: orders-api
```

должен совпасть с Pod labels:

```yaml
labels:
  app: orders-api
```

Здесь связь уже не exact reference, а label selection.

## Runtime sequence

```text
1 Gateway controller sees Gateway
2 controller configures its data plane
3 HTTPRoute attaches to listener
4 request /orders arrives
5 route selects orders-api Service
6 Service resolves ready endpoints
7 one Ready Pod receives request
8 Spring Boot handles request
```

## Проверка по слоям

```bash
kubectl get gatewayclass
kubectl get gateway
kubectl describe gateway public-api

kubectl get httproute
kubectl describe httproute orders-route

kubectl get svc orders-api
kubectl get endpointslice   -l kubernetes.io/service-name=orders-api

kubectl get pod -l app=orders-api
```

## Production

Обычно добавятся:
- HTTPS listener;
- certificate management;
- authentication;
- rate limits;
- request/body timeouts;
- WAF where needed;
- access logs and metrics.

Gateway API задаёт routing model, но не делает эти policies автоматически.

## Failure scenarios

### Wrong GatewayClass
Gateway создан, но controller его не обслуживает.

### Wrong backend Service
HTTPRoute есть, backend reference invalid.

### Service selector mismatch
Gateway и route исправны, но EndpointSlice пуст.

### Pod NotReady
Service теряет backend capacity.

## Главное диагностическое правило

Если Gateway и HTTPRoute Accepted, Service существует, но user получает 503 — проверяйте Service endpoints и readiness до того, как менять edge routing.
