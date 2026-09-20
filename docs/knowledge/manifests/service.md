# Manifest Reference — Service

## Учебная схема объекта

### Где Service находится в системе

```text
caller
  |
  | DNS: orders
  v
Service orders
  |
  | selector
  v
EndpointSlice
  |
  +--> Ready Pod A
  +--> Ready Pod B
```

Service создаёт **stable network identity** поверх disposable Pods.

### Annotated fragment

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders

spec:
  type: ClusterIP

  selector:
    app: orders          # должен совпасть с Pod label

  ports:
    - name: http
      port: 8080         # caller uses orders:8080
      targetPort: http   # maps to containerPort.name=http
```

### Не путать

```text
Service.metadata.name
 !=
Deployment.metadata.name requirement

Service selector
 ->
Pod labels
```

Полный пример: [Showcase 01 annotated Service](../../../showcases/01-internal-rest-service/annotated.yaml).

Проверено: 2026-09-20. Базовая версия Kubernetes: 1.37.

Эта глава — подробный справочник по `Service`. Она нужна, чтобы вы могли открыть любой Service manifest и понимать не только синтаксис, но и **что Kubernetes реально сделает с каждым полем**.

---

## 1. Зачем нужен Service

Pods являются временными:

```text
customer-api Pod A -> 10.42.1.15
Pod A deleted
customer-api Pod B -> 10.42.2.31
```

Клиент не должен знать Pod IP.

Service создаёт стабильную logical identity:

```text
orders-service
     |
     | http://customer-api:8080
     v
Kubernetes DNS
     |
     v
Service customer-api
     |
     v
EndpointSlice
     |
     +--> Pod A 10.42.1.15:8080 READY
     +--> Pod B 10.42.2.31:8080 READY
```

Для Spring Boot это означает: URL зависимости обычно указывает на **Service name**, а не на Pod.

---

# 2. Наш минимальный Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-app
spec:
  selector:
    app: spring-app
  ports:
    - name: http
      port: 8080
      targetPort: http
```

Теперь разберём каждую строку.

---

# 3. apiVersion

```yaml
apiVersion: v1
```

Service относится к core Kubernetes API group, поэтому здесь просто `v1`, а не:

```text
apps/v1
networking.k8s.io/v1
autoscaling/v2
```

Сравнение:

| Resource | apiVersion |
|---|---|
| Service | `v1` |
| ConfigMap | `v1` |
| Secret | `v1` |
| Deployment | `apps/v1` |
| NetworkPolicy | `networking.k8s.io/v1` |

**Практика CKAD:** `Service = v1` нужно знать без поиска.

---

# 4. kind

```yaml
kind: Service
```

Говорит API Server, объект какого типа создаётся.

Проверить:

```bash
kubectl api-resources | grep -i service
```

---

# 5. metadata.name

```yaml
metadata:
  name: spring-app
```

Это имя Service и основа DNS name.

В namespace `payments`:

```text
spring-app
spring-app.payments
spring-app.payments.svc.cluster.local
```

## Spring Boot example

```yaml
clients:
  spring-app:
    base-url: http://spring-app:8080
```

Если caller находится в том же namespace, короткого имени обычно достаточно.

### Что нельзя делать

Не использовать:

```yaml
base-url: http://10.42.2.14:8080
```

Это Pod IP, который изменится при replacement.

---

# 6. metadata.namespace

В manifest его можно не писать:

```yaml
metadata:
  name: spring-app
```

Тогда namespace определяется context/apply command.

Например:

```bash
kubectl apply -n production -f service.yaml
```

## Практика

Для reusable manifests часто namespace не hardcode.

Для GitOps environments namespace может задаваться Kustomize/Helm/platform layer.

---

# 7. spec.type

В нашем manifest поле отсутствует:

```yaml
spec:
  selector:
    ...
```

По умолчанию:

```text
type: ClusterIP
```

Это означает Service с cluster-internal virtual IP.

В Kubernetes доступны основные types:

```text
ClusterIP
NodePort
LoadBalancer
ExternalName
```

---

# 8. ClusterIP

Это default и обычный выбор для internal Spring Boot microservice.

```yaml
spec:
  type: ClusterIP
```

Kubernetes выделяет Service IP из service CIDR.

Проверить:

```bash
kubectl get svc spring-app -o wide
```

Пример:

```text
NAME         TYPE        CLUSTER-IP     PORT(S)
spring-app   ClusterIP   10.43.51.100   8080/TCP
```

## Важно

Spring Boot обычно не должен хранить этот ClusterIP.

Используется DNS name:

```text
spring-app
```

Почему?

ClusterIP стабилен в lifecycle Service, но DNS abstraction проще и переносимее.

---

# 9. selector

```yaml
selector:
  app: spring-app
```

Service ищет Pods с label:

```yaml
metadata:
  labels:
    app: spring-app
```

Deployment:

```yaml
spec:
  template:
    metadata:
      labels:
        app: spring-app
```

Получаем:

```text
Service selector
 app=spring-app
      |
      v
Pods with label
 app=spring-app
```

## Это не Deployment selector

Есть два разных selector:

Deployment:

```yaml
spec:
  selector:
    matchLabels:
      app: spring-app
```

Service:

```yaml
spec:
  selector:
    app: spring-app
```

Deployment selector отвечает за ownership ReplicaSet/Pods.

Service selector отвечает за **network backends**.

---

# 10. Что делает Kubernetes после selector

Control plane отслеживает matching Pods и создаёт/обновляет EndpointSlice.

Проверить:

```bash
kubectl get endpointslice   -l kubernetes.io/service-name=spring-app
```

Таким образом runtime chain:

```text
Pod labels
   |
   v
Service selector
   |
   v
EndpointSlice
   |
   v
network data plane
```

---

# 11. selector mismatch

Service:

```yaml
selector:
  app: spring-app
```

Pod:

```yaml
labels:
  app: spring-api
```

Получаем:

```text
Service exists
DNS exists
ClusterIP exists
but
EndpointSlice has no matching backend
```

Диагностика:

```bash
kubectl get svc spring-app
kubectl get pods --show-labels
kubectl get endpointslice   -l kubernetes.io/service-name=spring-app
```

Это один из самых частых Service incidents.

---

# 12. spec.ports

```yaml
ports:
  - name: http
    port: 8080
    targetPort: http
```

Service может иметь один или несколько ports.

Если несколько ports, имена нужны для однозначности.

---

# 13. ports[].name

```yaml
name: http
```

Имя Service port.

Полезно для:
- читаемости;
- multi-port Service;
- integrations, которые используют named ports.

При нескольких ports Kubernetes требует уникальные names.

Пример:

```yaml
ports:
  - name: http
    port: 8080
    targetPort: http

  - name: management
    port: 9090
    targetPort: management
```

---

# 14. ports[].port

```yaml
port: 8080
```

Это порт **Service**, который использует caller.

Caller:

```text
http://spring-app:8080
```

Очень важно:

> `port` — это не обязательно port контейнера.

---

# 15. ports[].targetPort

```yaml
targetPort: http
```

Это порт backend Pod.

Он может быть числом:

```yaml
targetPort: 8080
```

или named port:

```yaml
targetPort: http
```

Container:

```yaml
ports:
  - name: http
    containerPort: 8080
```

Named targetPort удобнее.

---

# 16. Почему named targetPort полезен

Сегодня:

```yaml
containerPort: 8080
name: http
```

Завтра application переходит на:

```yaml
containerPort: 8090
name: http
```

Service остаётся:

```yaml
targetPort: http
```

External logical contract Service не обязан меняться.

---

# 17. Default targetPort

Если `targetPort` не указан, Kubernetes использует то же значение, что `port`.

Пример:

```yaml
ports:
  - port: 8080
```

эквивалентно логически:

```yaml
ports:
  - port: 8080
    targetPort: 8080
```

Но explicit targetPort часто делает manifest понятнее.

---

# 18. ports[].protocol

Мы его не указали:

```yaml
ports:
  - port: 8080
```

Default:

```text
protocol: TCP
```

Для HTTP Spring Boot это почти всегда то, что нужно.

Можно явно:

```yaml
protocol: TCP
```

---

# 19. containerPort и Service port — не одно поле

Deployment:

```yaml
containers:
  - ports:
      - name: http
        containerPort: 8080
```

Service:

```yaml
ports:
  - port: 80
    targetPort: http
```

Caller вызывает:

```text
http://spring-app:80
```

Backend получает:

```text
Pod:8080
```

Схема:

```text
caller
  |
  | :80
  v
Service
  |
  | targetPort=http
  v
Pod :8080
```

---

# 20. Ошибка port/targetPort

Service:

```yaml
port: 8080
targetPort: 9090
```

Spring слушает:

```text
8080
```

DNS работает, Service существует, EndpointSlice может существовать — но connection к backend port будет неуспешным.

Диагностика:

```bash
kubectl get svc spring-app -o yaml
kubectl get pod -o yaml
kubectl exec <debug-pod> --   curl -v http://spring-app:8080
```

---

# 21. type: NodePort

```yaml
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: http
      nodePort: 30080
```

Схема:

```text
client
 -> NodeIP:30080
 -> Service
 -> Pod:8080
```

NodePort строится поверх ClusterIP.

## Когда использовать

- labs;
- bare-metal debugging;
- как building block для некоторых external load balancer implementations.

## Когда не использовать как default

Обычный production public API лучше публиковать через LoadBalancer/Gateway/Ingress architecture.

Не выдавать каждому microservice собственный NodePort без причины.

---

# 22. nodePort

Если не задавать вручную, control plane назначает порт из configured NodePort range.

В стандартной Kubernetes конфигурации часто используется диапазон:

```text
30000-32767
```

но platform может настроить диапазон иначе.

Не hardcode предположение о range без проверки cluster configuration.

---

# 23. type: LoadBalancer

```yaml
spec:
  type: LoadBalancer
```

Kubernetes просит внешний load balancer у интегрированного implementation/provider.

Важно:

> Kubernetes API описывает intent, но сам по себе Kubernetes не предоставляет универсальный внешний cloud load balancer.

В cloud это может быть provider integration.

В bare metal:
- MetalLB;
- другой LB controller;
- platform-specific implementation.

---

# 24. LoadBalancer не нужен для каждого backend

Плохо:

```text
orders -> LoadBalancer
customers -> LoadBalancer
payments -> LoadBalancer
reports -> LoadBalancer
```

если все должны быть доступны только через единый API boundary.

Обычно:

```text
Internet
 -> Gateway/Ingress
 -> ClusterIP Services
 -> Pods
```

---

# 25. type: ExternalName

Пример:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-db
spec:
  type: ExternalName
  externalName: database.company.internal
```

Kubernetes DNS возвращает CNAME-style mapping.

Нет:
- Pod selector;
- Kubernetes proxying;
- cluster backend load balancing.

Это DNS abstraction для external name.

---

# 26. Service without selector

Можно создать:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-api
spec:
  ports:
    - port: 8080
      targetPort: 8080
```

Без selector Kubernetes не создаёт EndpointSlice автоматически из Pods.

Это используется для manual/external endpoints и migration scenarios.

Важно не использовать это без понимания ownership endpoints.

---

# 27. Headless Service

```yaml
spec:
  clusterIP: None
```

У headless Service нет virtual ClusterIP load balancing.

DNS может вернуть individual backend Pod addresses.

Схема:

```text
normal Service DNS
 -> Service virtual IP

headless Service DNS
 -> Pod IP A
 -> Pod IP B
 -> Pod IP C
```

Полезно для:
- StatefulSet;
- Kafka;
- databases;
- protocols, которые сами выбирают peer.

Для обычного Spring Boot REST API чаще нужен обычный ClusterIP.

---

# 28. clusterIP

Обычно Kubernetes назначает автоматически.

```yaml
spec:
  clusterIP: 10.43.x.x
```

Не нужно вручную задавать без реальной причины.

Special value:

```yaml
clusterIP: None
```

означает headless.

---

# 29. sessionAffinity

Default:

```text
None
```

Можно:

```yaml
sessionAffinity: ClientIP
```

Тогда Kubernetes старается отправлять requests от одного client IP к одному backend.

## Для Spring Boot

Не использовать sticky sessions для маскировки неправильной stateful application architecture.

Лучше:
- stateless HTTP where practical;
- external session store if server sessions required.

---

# 30. internalTrafficPolicy

Default:

```text
Cluster
```

То есть internal traffic может использовать ready endpoints на любых подходящих nodes.

Можно:

```yaml
internalTrafficPolicy: Local
```

Тогда используются только node-local endpoints.

Риск:

```text
caller on node A
no backend endpoint on node A
=> Service behaves as no endpoint for that caller
```

Это optimization/semantics field, не базовая настройка для каждого REST Service.

---

# 31. externalTrafficPolicy

Актуально для external Service types.

Типичные значения:

```text
Cluster
Local
```

`Local` может сохранять source IP и избегать extra hops, но влияет на availability/distribution.

Это platform/network design decision.

---

# 32. trafficDistribution

В Kubernetes 1.37 Service поддерживает preference hints вроде:

```text
PreferSameZone
PreferSameNode
```

Это **preference**, а не строгий guarantee.

Использовать только после понимания topology/latency/cost goals.

Для базового Spring Boot Service поле не нужно.

---

# 33. appProtocol

Можно подсказать application protocol:

```yaml
ports:
  - name: http2
    port: 8080
    targetPort: http
    appProtocol: kubernetes.io/h2c
```

Это hint для implementations. Не заменяет `protocol: TCP`.

Для обычного HTTP/1 Spring Boot Service часто не требуется.

---

# 34. readiness и Service

Это одна из самых важных связей.

```text
Spring Boot
 /readyz returns 503
      |
      v
readinessProbe fails
      |
      v
Pod Ready=False
      |
      v
EndpointSlice endpoint not ready
      |
      v
Service stops using it as normal ready backend
```

Поэтому Service и probes нельзя изучать отдельно.

---

# 35. Что Service НЕ делает

Service не:
- authenticates caller;
- authorizes user;
- encrypts traffic;
- гарантирует retry;
- делает circuit breaker;
- хранит session;
- проверяет business health.

Эти задачи относятся к другим layers.

---

# 36. Service + NetworkPolicy

Service говорит:

```text
куда направить traffic
```

NetworkPolicy:

```text
разрешён ли этот network flow
```

Spring Security:

```text
кто caller и что ему можно
```

Три разных слоя.

---

# 37. Production Service example

Для обычного internal Spring Boot API:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
  labels:
    app.kubernetes.io/name: orders
spec:
  type: ClusterIP

  selector:
    app.kubernetes.io/name: orders

  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: http

  sessionAffinity: None
  internalTrafficPolicy: Cluster
```

Здесь часть defaults указана явно **для учебной читаемости**.

В реальном manifest можно не писать default fields, если команда предпочитает меньше YAML.

---

# 38. Spring Boot Deployment pair

Deployment:

```yaml
metadata:
  name: orders

spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: orders

  template:
    metadata:
      labels:
        app.kubernetes.io/name: orders

    spec:
      containers:
        - name: orders
          image: registry.example/orders:1.0.0

          ports:
            - name: http
              containerPort: 8080

          readinessProbe:
            httpGet:
              path: /readyz
              port: http
```

Service:

```yaml
spec:
  selector:
    app.kubernetes.io/name: orders

  ports:
    - port: 8080
      targetPort: http
```

Обратите внимание на два coupling contracts:

```text
Service selector
        =
Pod labels

Service targetPort name
        =
container port name
```

---

# 39. Практика: проверить Service целиком

```bash
kubectl get svc orders
kubectl describe svc orders
kubectl get svc orders -o yaml

kubectl get pods --show-labels

kubectl get endpointslice   -l kubernetes.io/service-name=orders

kubectl exec <debug-pod> -- nslookup orders

kubectl exec <debug-pod> --   curl -v http://orders:8080/readyz
```

---

# 40. Troubleshooting algorithm

```text
1 Service exists?
      |
      no -> wrong namespace/name

2 DNS resolves?
      |
      no -> DNS/policy issue

3 EndpointSlice has endpoints?
      |
      no -> selector/Pod issue

4 endpoints Ready?
      |
      no -> readiness issue

5 targetPort correct?
      |
      no -> port mismatch

6 TCP works?
      |
      no -> NetworkPolicy/CNI/app listener

7 HTTP 401/403?
      |
      auth/authz layer
```

Это полезный production runbook.

---

# 41. Failure lab 1 — selector mismatch

Изменить:

```yaml
selector:
  app: wrong-name
```

Проверить:

```bash
kubectl get endpointslice
kubectl get pods --show-labels
```

Ожидаем:
- Service есть;
- DNS есть;
- endpoints отсутствуют.

---

# 42. Failure lab 2 — wrong targetPort

Поменять:

```yaml
targetPort: 9999
```

Ожидаем:
- selector работает;
- EndpointSlice содержит Pod;
- connection к backend неуспешно.

Это помогает отделить selector failure от port failure.

---

# 43. Failure lab 3 — readiness

Сломайте `/readyz`.

Ожидаем:
- Pod Running;
- Ready=False;
- Service backend not ready.

---

# 44. Failure lab 4 — NetworkPolicy

DNS и endpoint существуют, но traffic denied.

Это показывает:

```text
Service discovery success
!=
network reachability success
```

---

# 45. CKAD: что уметь быстро

Создать:

```bash
kubectl expose deployment orders   --name=orders   --port=8080   --target-port=8080   --type=ClusterIP
```

Проверить:

```bash
kubectl get svc
kubectl describe svc orders
kubectl get endpointslice
```

Уметь руками написать:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
spec:
  selector:
    app: orders
  ports:
    - port: 8080
      targetPort: 8080
```

---

# 46. Anti-patterns

- Pod IP вместо Service;
- NodePort для каждого internal API;
- LoadBalancer для каждого backend;
- Service selector не совпадает с labels;
- numeric targetPort duplicated everywhere вместо named port;
- sticky sessions как исправление stateful backend design;
- считать Service security boundary;
- internal requests гонять через public ingress;
- manually choose clusterIP без причины.

---

# 47. Поля, которые чаще всего нужны Spring Boot developer

Практический минимум:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
spec:
  selector:
    app: orders
  ports:
    - name: http
      port: 8080
      targetPort: http
```

Нужно понимать глубоко:

```text
name
selector
port
targetPort
protocol
type
ClusterIP
EndpointSlice
readiness relationship
DNS
```

Advanced по необходимости:

```text
NodePort
LoadBalancer
ExternalName
headless
sessionAffinity
internalTrafficPolicy
externalTrafficPolicy
trafficDistribution
appProtocol
```

---

# 48. Developer vs Platform responsibility

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| application port | ✓ | | |
| Service port contract | ✓ | | ✓ |
| labels/selectors | ✓ | | ✓ |
| ClusterIP networking | | ✓ | |
| external LoadBalancer | | ✓ | |
| Gateway/Ingress | | ✓ | ✓ |
| NetworkPolicy | | ✓ | ✓ |
| readiness semantics | ✓ | | ✓ |

---

# 49. Production checklist

Перед release Service:

1. `type` выбран осознанно.
2. Internal backend обычно `ClusterIP`.
3. Selector совпадает с Pod labels.
4. Named targetPort совпадает с container port name.
5. Readiness корректно влияет на endpoints.
6. Caller использует DNS Service, не Pod IP.
7. Нет ненужного NodePort/LoadBalancer.
8. NetworkPolicy рассмотрена отдельно.
9. Authentication рассмотрена отдельно.
10. EndpointSlice проверен.
11. Multi-port ports имеют names.
12. External publishing идёт через выбранную platform architecture.

---

# 50. Sources: проверка спецификации

Основной материал выше самодостаточен. Sources нужны для сверки Kubernetes API и advanced возможностей.

- https://kubernetes.io/docs/concepts/services-networking/service/  
  Использовано: ClusterIP default, Service types, selector/EndpointSlice, port/targetPort, named ports, headless Services, ExternalName, appProtocol, traffic distribution.
- https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/  
  Использовано: `internalTrafficPolicy`, default `Cluster`, behavior `Local`.
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/  
  Использовано: Service DNS model.

Проверено: **2026-09-20**.

## Production-like examples

- [ClusterIP internal service](../../../showcases/01-internal-rest-service/README.md)
- [Service selector cutover](../../../showcases/08-blue-green/README.md)
- [Headless Service](../../../showcases/18-statefulset-headless-service/README.md)

Каждый showcase содержит `WALKTHROUGH.md` с разбором mapping'ов и `all.yaml` с полной композицией.
