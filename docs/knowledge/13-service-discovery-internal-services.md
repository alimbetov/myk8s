# 13 — Service discovery and internal services

## Учебная карта темы

### Service discovery без Eureka

```text
Spring client
   |
   | customer-api
   v
cluster DNS
   |
   v
Service
   |
   v
EndpointSlice
   |
   v
Ready Pods
```

В Kubernetes для типичного internal HTTP сервиса:

```text
Eureka instance registry
        ↓
заменяется во многом на
        ↓
Service + DNS + EndpointSlice
```

Это не значит, что Service Mesh/API registry никогда не нужны, но базовый location discovery уже встроен.

### DNS scopes

```text
customer-api
customer-api.crm
customer-api.crm.svc.cluster.local
```

### Headless exception

```text
normal Service -> virtual service identity
headless Service -> individual Pod DNS/IPs
```

Практика: [Internal REST](../../showcases/01-internal-rest-service/README.md) и [StatefulSet/headless](../../showcases/18-statefulset-headless-service/README.md).

## Для аналитика, разработчика и тестировщика

### Аналитик

Использует logical service names в архитектурных схемах и явно отмечает namespace/cross-namespace contracts.

### Разработчик

Переходит от Eureka-instance thinking к Service/DNS/EndpointSlice, где это достаточно, и понимает headless Service как отдельный случай.

### Тестировщик

Проверяет:
- DNS names;
- namespace resolution;
- selector mismatch;
- no ready endpoints;
- headless individual identities.

### Перед следующей главой

Нужно различать:

```text
service discovery
load balancing
authentication
authorization
```

Они могут идти в одной request chain, но решают разные задачи.

Проверено: 2026-09-20.

Цель главы — понять, как микросервисы находят друг друга в Kubernetes и почему привычные Eureka/IP-based подходы часто становятся не нужны внутри cluster.

## 1. Как было раньше

В VM/microservice architecture часто использовали:

```text
service
 -> Eureka/Consul
 -> instance list
 -> client-side load balancing
 -> instance IP
```

Это решало проблему динамических instances вне Kubernetes.

## 2. Как сейчас в Kubernetes

Kubernetes уже имеет:
- Service;
- DNS;
- EndpointSlice;
- kube-proxy/eBPF/CNI data plane.

```text
orders
 -> customer-api
 -> DNS
 -> Service
 -> EndpointSlice
 -> ready Pods
```

Для многих внутренних HTTP services этого достаточно.

## 3. ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: customer-api
spec:
  selector:
    app: customer-api
  ports:
    - port: 8080
      targetPort: http
```

Spring:

```yaml
clients:
  customer:
    base-url: http://customer-api:8080
```

## 4. DNS names

Same namespace:

```text
customer-api
```

Cross namespace:

```text
customer-api.crm
```

FQDN:

```text
customer-api.crm.svc.cluster.local
```

Prefer shortest stable name appropriate to namespace boundaries.

## 5. EndpointSlice

Service selector не отправляет traffic магически. Kubernetes tracks backend endpoints in EndpointSlices.

```bash
kubectl get endpointslice   -l kubernetes.io/service-name=customer-api
```

Это один из лучших debugging commands.

## 6. Readiness integration

NotReady Pod не должен участвовать как обычный ready backend.

```text
Spring readiness false
 -> Pod Ready=False
 -> endpoint not ready
 -> Service routes elsewhere
```

Service discovery напрямую связан с application health.

## 7. Headless Service

```yaml
clusterIP: None
```

Headless Service полезен, когда client должен видеть individual Pod addresses, например некоторые stateful protocols.

Для обычного REST backend обычно ClusterIP проще.

## 8. Нужно ли Eureka внутри Kubernetes

Не автоматически.

Если Eureka нужен только для:
- instance registration;
- discovery;
- client load balancing;

Kubernetes Service/DNS уже решает эти задачи на platform level.

Eureka может оставаться, если:
- hybrid environment;
- специфичный Spring Cloud contract;
- cross-cluster/non-Kubernetes discovery;
- migration stage.

## 9. Migration from Eureka

Переход можно делать постепенно:

```text
phase 1:
Kubernetes runs app, Eureka remains

phase 2:
internal URLs move to Services

phase 3:
remove client-side discovery where no longer needed
```

Не удаляйте discovery framework без анализа retries/load balancing/config dependencies.

## 10. Client-side vs service-side load balancing

Eureka-era client мог выбирать instance.

Kubernetes Service abstraction скрывает Pod set behind one service identity.

Это упрощает application but shifts some concerns into platform networking.

## 11. Stateful service naming

StatefulSet + headless service может дать stable Pod DNS:

```text
broker-0.kafka-headless
broker-1.kafka-headless
```

Но application protocol/operator должен понимать, когда это нужно.

## 12. Failure practice: selector mismatch

```bash
kubectl get pod --show-labels
kubectl get svc
kubectl get endpointslice
```

Service DNS существует, endpoints нет.

## 13. Failure practice: wrong namespace

Caller uses:

```text
http://customer-api:8080
```

но Service в namespace `crm`.

Исправление:

```text
http://customer-api.crm:8080
```

либо архитектурно colocate services only if justified.

## 14. Failure practice: no ready endpoints

Все Pods Running, readiness false.

```bash
kubectl get pods
kubectl get endpointslice
kubectl describe pod
```

Root cause находится в application readiness, а не Service object.

## 15. Anti-patterns

- Pod IP в config;
- manual hosts file;
- public ingress для каждого internal call;
- Eureka + Kubernetes discovery без понятной причины;
- hardcoded FQDN cluster domain везде;
- treating Service as authentication mechanism.

## 16. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| service contract | ✓ | | ✓ |
| DNS/CNI | | ✓ | |
| labels/selectors | | | ✓ |
| discovery architecture | | | ✓ |
| Eureka migration | ✓ | | ✓ |

## 17. CKAD

Нужно уметь Service, selector, DNS, EndpointSlice, headless basics.

## 18. Checklist

- caller uses Service name;
- selector tested;
- readiness affects endpoints correctly;
- namespace naming understood;
- no unnecessary external ingress;
- discovery technology has explicit purpose.

## Связанные production-like примеры

- [Service DNS and EndpointSlice](../../showcases/01-internal-rest-service/README.md)
- [Two internal Spring services](../../showcases/06-service-to-service-security/README.md)
- [Headless Service and stable Pod DNS](../../showcases/18-statefulset-headless-service/README.md)

В каждом showcase откройте `README.md`, затем `WALKTHROUGH.md` и `all.yaml`: теория этой главы там показана как часть связной архитектуры.

## Sources

- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/

Проверено: **2026-09-20**.
