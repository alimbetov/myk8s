# Walkthrough — Internal Spring Boot REST service

## Архитектура

```text
caller
  |
  | DNS customer-api
  v
Service
  |
  | selector
  v
EndpointSlice
  |
  v
Ready Pod
  |
  v
Spring Boot
```

## Файлы

- [README](README.md)
- [Полный manifest](all.yaml)
- [Spring config](application.yaml)
- [Service reference](../../docs/knowledge/manifests/service.md)

## Deployment -> Service

Pod labels:

```yaml
app.kubernetes.io/name: customer-api
```

Service selector должен совпасть.

## Named port

```text
Service.port=8080
 -> targetPort=http
 -> containerPort.name=http
 -> containerPort=8080
```

Именно named port связывает Service с container port.

## ConfigMap / Secret

ConfigMap содержит non-secret:
- DB URL;
- cache TTL;
- actuator settings.

Secret:
- DB username/password.

Spring Environment получает их через `envFrom`.

## Readiness

```text
Spring /readyz
 -> kubelet probe
 -> Pod Ready
 -> EndpointSlice
 -> Service traffic
```

Если readiness fails, процесс может оставаться Running.

## Production attention

- image должен быть immutable;
- resources должны быть измерены;
- liveness не должна зависеть от PostgreSQL;
- credentials должны иметь rotation strategy;
- Service остаётся ClusterIP.

## Failure scenarios

### selector mismatch
DNS работает, endpoints пусты.

### targetPort wrong
endpoints есть, connection fails.

### readiness wrong
Pod Running, Ready=False.

### Secret wrong
Kubernetes networking healthy, DB auth fails.

## Проверка

```bash
kubectl get deploy,pod,svc
kubectl get endpointslice   -l kubernetes.io/service-name=customer-api
kubectl describe pod <pod>
```
