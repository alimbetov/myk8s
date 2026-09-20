# 02 — Services, DNS, EndpointSlice и service-to-service security

Проверено: 2026-09-20.

Эта глава отвечает на следующий вопрос после configuration:

> В `application.yaml` у нас появился URL `http://customer-api:8080`. **Что такое `customer-api`, кто создаёт этот адрес, как Kubernetes находит нужные Pods и на каком слое появляется security?**

Главная идея главы:

```text
Service discovery
      ≠
network reachability
      ≠
transport security
      ≠
authentication
      ≠
authorization
```

Все эти механизмы могут участвовать в одном HTTP вызове, но каждый отвечает на **свой вопрос**.

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь объяснить:

1. Почему Spring Boot service не должен хранить Pod IP другого сервиса.
2. Что такое Service и какую проблему он решает.
3. Как Service связан с Pod labels.
4. Что такое ClusterIP.
5. Чем `port` отличается от `targetPort`.
6. Почему named ports удобнее hardcoded numeric targetPort.
7. Что такое EndpointSlice и кто его создаёт.
8. Как readiness влияет на EndpointSlice и routing.
9. Как Kubernetes DNS строит Service names.
10. Почему short DNS name работает только в ожидаемом namespace context.
11. Чем normal Service отличается от headless Service.
12. Что означает selectorless Service.
13. Где участвуют kube-proxy или CNI/service dataplane.
14. Почему Service не является security boundary.
15. Как NetworkPolicy выбирает Pods.
16. Почему NetworkPolicy требует CNI implementation support.
17. Почему default-deny egress может сломать DNS.
18. Чем NetworkPolicy отличается от TLS/mTLS.
19. Чем JWT authentication отличается от ServiceAccount/RBAC.
20. Как по symptom отличать DNS, Service, network и application security failure.

---

# 2. Одна схема всего internal HTTP вызова

Будем использовать два Spring Boot сервиса:

```text
orders-api
   |
   | GET http://customer-api:8080/customers/42
   v
cluster DNS
   |
   v
Service customer-api
   |
   v
EndpointSlice
   |
   +--> customer Pod A Ready
   +--> customer Pod B Ready
   |
   v
network dataplane
   |
   | NetworkPolicy allows?
   v
TCP connection
   |
   | TLS/mTLS if configured
   v
customer-api Spring Boot
   |
   | JWT validation
   v
Authentication
   |
   | authorization rules
   v
CustomerController
```

Уже по этой схеме видно, почему слово “не работает сеть” слишком общее.

---

# 3. Как было бы без Service

Представим два customer Pods:

```text
customer-api-7d6f8-a
IP 10.42.1.17

customer-api-7d6f8-b
IP 10.42.2.24
```

orders-api мог бы попробовать:

```text
http://10.42.1.17:8080
```

Но после replacement:

```text
old Pod
10.42.1.17
   X

new Pod
10.42.3.91
```

Application contract сломан.

Pod IP — **runtime endpoint**, а не durable service identity.

---

# 4. Service решает проблему stable identity

Создаём:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: customer-api
spec:
  selector:
    app: customer-api

  ports:
    - name: http
      port: 8080
      targetPort: http
```

Теперь caller использует:

```text
http://customer-api:8080
```

Mental model:

```text
changing Pods
     ↓
stable Service abstraction
     ↓
stable DNS name
```

---

# 5. Service не является application server

Важно понять отрицательное определение.

Service:

- не запускает Spring Boot;
- не хранит business state;
- не проверяет JWT;
- не создаёт Deployment;
- не означает public endpoint.

Он даёт network abstraction над endpoints.

```text
Service
  |
  | stable identity + port abstraction
  v
current backend endpoints
```

---

# 6. Как Service находит Pods

Главный mapping:

```yaml
# Service
spec:
  selector:
    app: customer-api
```

Pod:

```yaml
metadata:
  labels:
    app: customer-api
```

Связь:

```text
Service.spec.selector
        =
Pod.metadata.labels
```

Service **не ищет Deployment по имени**.

Например:

```text
Deployment name: customer-backend-v17
Service name:    customer-api

Pod label:
app=customer-api
```

Это нормально.

---

# 7. Почему labels — важнейший Kubernetes contract

Label — это metadata, которую другие objects используют для selection.

Например:

```yaml
labels:
  app: customer-api
  version: v2
  component: backend
```

Разные механизмы могут использовать разные labels:

```text
Service
  -> app=customer-api

Canary Service
  -> app=customer-api,version=v2

NetworkPolicy
  -> app=customer-api

Observability
  -> version=v2
```

Поэтому labels — не декоративные tags.

---

# 8. port, targetPort и containerPort

Это одна из самых частых точек путаницы.

Container:

```yaml
ports:
  - name: http
    containerPort: 8080
```

Service:

```yaml
ports:
  - name: http
    port: 80
    targetPort: http
```

Flow:

```text
caller
  |
  | customer-api:80
  v
Service.port = 80
  |
  | targetPort=http
  v
Pod port named "http"
  |
  v
containerPort 8080
```

Итого:

```text
port
=
Service-facing port

targetPort
=
backend Pod port/name

containerPort
=
documentation/runtime port inside container spec
```

---

# 9. Почему named targetPort удобен

Вместо:

```yaml
targetPort: 8080
```

можно:

```yaml
targetPort: http
```

а container:

```yaml
- name: http
  containerPort: 8080
```

Теперь Service contract говорит:

> отправляй traffic в logical port `http`.

Если application container port меняется, mapping остаётся более читаемым.

---

# 10. ClusterIP

По умолчанию Service type:

```text
ClusterIP
```

Упрощённо:

```text
Service customer-api
      |
      +--> stable virtual service address
      |
      +--> stable DNS name
      |
      +--> current backend endpoints
```

Для internal Spring Boot microservice этого часто достаточно.

Не нужно автоматически делать каждый Service:

```yaml
type: LoadBalancer
```

или NodePort.

---

# 11. Что происходит после создания Service

Для selector-based Service control plane автоматически поддерживает EndpointSlices.

```text
Service selector
      |
      v
matching Pods
      |
      +--> Pod A
      +--> Pod B
      |
      v
EndpointSlice controller
      |
      v
EndpointSlice
      |
      +--> A IP / conditions
      +--> B IP / conditions
```

EndpointSlice API stable с Kubernetes 1.21. Legacy Endpoints API deprecated с Kubernetes 1.33; для диагностики и новых integrations лучше мыслить через EndpointSlice.

---

# 12. EndpointSlice: почему это важно разработчику

Разработчик обычно не создаёт EndpointSlice вручную для обычного Service.

Но при incident это один из важнейших объектов.

Проверка:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api

kubectl describe endpointslice <name>
```

Он отвечает на вопрос:

> Какие backend endpoints Kubernetes сейчас считает относящимися к Service?

---

# 13. Selector mismatch

Service:

```yaml
selector:
  app: customer-api
```

Pod:

```yaml
labels:
  app: customers-api
```

Одна лишняя `s`.

Runtime:

```text
Service exists
DNS exists
Pods Running
     |
     v
selector finds zero Pods
     |
     v
EndpointSlice has no useful backends
     |
     v
request fails
```

Это классический пример:

> object exists ≠ system works.

---

# 14. Readiness и Service routing

Допустим Pod label совпадает, но:

```text
Pod Running=True
Pod Ready=False
```

Например readiness endpoint:

```text
/readyz -> 503
```

Упрощённая chain:

```text
kubelet readinessProbe
      ↓
Pod Ready condition
      ↓
EndpointSlice readiness condition
      ↓
normal Service traffic avoids not-ready endpoint
```

Поэтому:

```text
Running
!=
Ready
```

---

# 15. Один incident глазами Kubernetes

Пусть user получает 503.

Проверяем:

```bash
kubectl get pod -l app=customer-api
```

Видим:

```text
NAME              READY   STATUS
customer-api-x    0/1     Running
customer-api-y    0/1     Running
```

Deployment “есть”.
Pods “Running”.

Но:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api
```

нет Ready endpoints.

Root cause находится не в DNS, а в application readiness.

---

# 16. Кто реально проксирует Service traffic

Conceptual model:

```text
Service
  |
  v
service dataplane
  |
  v
backend endpoint
```

В классической Kubernetes реализации service proxying часто реализует `kube-proxy`.

Но некоторые networking implementations используют собственный service dataplane, интегрированный с CNI/eBPF и не обязательно полагаются на kube-proxy.

Поэтому правильная mental model:

> Service API описывает desired virtual service; конкретный dataplane implementation зависит от cluster networking stack.

Не привязывайте application architecture к конкретным iptables rules.

---

# 17. DNS: откуда появляется имя customer-api

Kubernetes публикует DNS records для Services.

В Pod kubelet настраивает DNS resolver configuration.

Для Service:

```text
customer-api
```

в namespace `sales` canonical form обычно выглядит:

```text
customer-api.sales.svc.cluster.local
```

Где cluster domain часто `cluster.local`, но platform может использовать другой.

---

# 18. Short name и namespace search path

Если orders-api находится в namespace `sales`:

```text
customer-api
```

резолвится относительно search domains текущего Pod.

Conceptually:

```text
customer-api
   ↓
customer-api.sales.svc.cluster.local
```

Если customer-api находится в namespace `crm`:

```text
customer-api
```

из `sales` не означает автоматически `crm`.

Нужно:

```text
customer-api.crm
```

или FQDN.

---

# 19. Почему namespace — часть integration contract

Аналитик часто пишет:

> orders вызывает customer-api.

Но для Kubernetes architecture полезнее:

```text
caller:
  orders-api
  namespace=sales

target:
  customer-api
  namespace=crm

protocol:
  HTTP

port:
  8080
```

Потому что namespace влияет на:

- DNS;
- NetworkPolicy selectors;
- RBAC;
- ownership;
- isolation boundaries.

---

# 20. Как посмотреть DNS configuration из Pod

```bash
kubectl exec <pod> -- cat /etc/resolv.conf
```

Можно увидеть search domains и nameserver.

Для проверки:

```bash
kubectl exec <debug-pod> -- nslookup customer-api
```

или:

```bash
kubectl exec <debug-pod> -- nslookup customer-api.crm
```

---

# 21. NXDOMAIN: что это означает

Если:

```text
nslookup customer-api
-> NXDOMAIN
```

первый набор hypotheses:

- Service name wrong;
- namespace wrong;
- DNS search context wrong;
- Service отсутствует;
- DNS infrastructure problem.

NetworkPolicy egress к DNS обычно проявляется как timeout/failure reaching DNS resolver, а не как authoritative NXDOMAIN для корректно работающего DNS.

То есть важно различать:

```text
DNS answered "name does not exist"
vs
DNS server unreachable
```

---

# 22. DNS Service существует, но traffic всё равно не работает

DNS отвечает:

```text
customer-api -> ClusterIP
```

Это доказывает только:

> Service DNS name существует.

Это **не доказывает**:

- есть Ready endpoints;
- targetPort правильный;
- application слушает port;
- NetworkPolicy разрешает traffic;
- JWT валиден.

---

# 23. localhost: частая ошибка после монолита

Внутри Pod:

```text
localhost
=
network namespace текущего Pod
```

Если orders-api и payment-api — разные Deployments/Pods:

```text
orders Pod
  localhost
     |
     v
orders Pod itself
```

Не payment-api.

Правильно:

```text
http://payment-api:8080
```

Исключение: sidecar в том же Pod действительно доступен через localhost.

---

# 24. Headless Service: зачем иногда не нужен virtual service endpoint

Normal Service:

```text
client
  ↓
Service abstraction
  ↓
one backend selected
```

Headless:

```yaml
clusterIP: None
```

Mental model:

```text
client / peer discovery
       ↓
DNS
       ↓
individual endpoint identities
```

Это полезно для stateful peer discovery.

Пример:

- [Showcase 18 — StatefulSet + headless Service](../../showcases/18-statefulset-headless-service/README.md)

Для обычного REST API headless Service обычно не нужен.

---

# 25. Selectorless Service

Service может существовать без selector.

Тогда Kubernetes не может автоматически вывести backends из Pod labels.

Use cases:

- external/non-Pod backend;
- manual EndpointSlice;
- migration/integration scenarios.

Это advanced case.

Для типичного Spring Boot microservice:

```text
Deployment
+
selector-based Service
```

остаётся базовой моделью.

---

# 26. ExternalName: ещё один специальный Service type

```yaml
type: ExternalName
externalName: legacy-db.example.internal
```

Conceptually:

```text
Kubernetes Service DNS name
       ↓
DNS alias
       ↓
external DNS name
```

Это не proxy Service с обычными Pod endpoints.

Использовать осознанно: protocol/hostname/TLS behavior может зависеть от исходного hostname.

---

# 27. Теперь добавляем NetworkPolicy

До сих пор мы отвечали:

> Куда отправить traffic?

Теперь другой вопрос:

> Разрешено ли этому traffic вообще пройти?

```text
orders Pod
   |
   | packet
   v
NetworkPolicy enforcement
   |
   v
customer Pod
```

NetworkPolicy работает на L3/L4 model: Pods/namespaces/IP blocks + protocols/ports.

---

# 28. NetworkPolicy требует implementation support

Очень важная особенность:

```text
kubectl apply NetworkPolicy
      ↓
API object exists
```

ещё не гарантирует enforcement.

Нужен network plugin/CNI implementation, который поддерживает NetworkPolicy.

Иначе:

```text
NetworkPolicy object
      |
      X
no dataplane enforcement
```

Поэтому platform baseline должен явно отвечать:

> Какая CNI используется и поддерживает ли она NetworkPolicy?

---

# 29. По умолчанию отсутствие NetworkPolicy не означает isolation

Если Pod не изолирован подходящей policy:

```text
traffic generally allowed by NetworkPolicy model
```

Production environments часто переходят к:

```text
default deny
+
explicit allows
```

но это нужно делать осознанно.

---

# 30. Default deny ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Здесь нет `ingress` allow rules.

Conceptually:

```text
all selected Pods
  ↓
ingress isolated
  ↓
only traffic allowed by additive policies passes
```

---

# 31. Default deny egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

Это особенно опасно для новичка.

Почему:

```text
orders Pod
  |
  +--> customer-api
  +--> PostgreSQL
  +--> OIDC
  +--> telemetry
  +--> DNS
```

Все эти flows нужно осознанно разрешить.

---

# 32. Почему default-deny egress ломает DNS

DNS — тоже network traffic.

```text
Spring
  |
  | resolve customer-api
  v
cluster DNS Service
```

Если egress запрещён:

```text
DNS query blocked
   ↓
hostname cannot resolve
   ↓
UnknownHostException / resolver timeout
```

Official Kubernetes guidance прямо предупреждает, что default-deny egress требует отдельного DNS allowance, если workload нужен DNS.

---

# 33. Почему нельзя просто скопировать DNS allow rule из интернета

В разных clusters DNS component может иметь разные:

- namespace;
- labels;
- Service names;
- CNI behavior.

Например часто встречается CoreDNS в `kube-system`, но учебник не должен hardcode platform assumptions.

Правильный порядок:

```bash
kubectl get svc -n kube-system
kubectl get pod -n kube-system --show-labels
```

После этого написать policy под **реальный** cluster.

---

# 34. Allow orders -> customer

Допустим оба сервиса в одном namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: customer-from-orders
spec:
  podSelector:
    matchLabels:
      app: customer-api

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: orders-api

      ports:
        - protocol: TCP
          port: 8080
```

Mapping:

```text
policy podSelector
   -> target Pods

from.podSelector
   -> allowed source Pods

ports
   -> allowed destination ports
```

---

# 35. Cross-namespace policy

Допустим:

```text
orders-api namespace=sales

customer-api namespace=crm
```

Одного `podSelector` недостаточно, потому что podSelector в peer без namespaceSelector относится к policy namespace context.

Нужно сознательно моделировать namespace.

Conceptually:

```text
namespaceSelector:
  team=sales
        +
podSelector:
  app=orders-api
```

Это означает:

> orders Pods из разрешённого namespace set.

---

# 36. NetworkPolicy policies additive

NetworkPolicy — не ordered firewall rule list.

Нет:

```text
rule 1 allow
rule 2 deny
rule 3 overrides
```

Модель:

```text
Pod isolated?
   ↓
allowed traffic
=
union of applicable allow rules
```

В стандартном NetworkPolicy API нет explicit deny rule, которое “перебивает” allow.

Это важно для debugging.

---

# 37. NetworkPolicy не умеет TLS и JWT

NetworkPolicy отвечает примерно:

```text
source/destination/port/protocol
```

Она не знает:

- JWT subject;
- OAuth scope;
- TLS certificate identity;
- HTTP path;
- business customer ID.

Поэтому:

```text
NetworkPolicy
!=
Spring Security
```

---

# 38. Service тоже не является security boundary

Есть ошибочная mental model:

```text
Service type=ClusterIP
therefore secure
```

ClusterIP означает internal service exposure model, но не application authentication.

Если Pod может reach Service:

```text
TCP path exists
```

это ещё не отвечает:

> Имеет ли caller право выполнить `POST /payments`?

---

# 39. Security layers одного вызова

Теперь соберём layers:

```text
orders-api
   |
   | DNS
   v
customer-api Service
   |
   | NetworkPolicy
   v
TCP
   |
   | TLS
   v
encrypted channel
   |
   | optional mTLS/workload identity
   v
authenticated peer
   |
   | Bearer JWT
   v
Spring Security
   |
   | authorization
   v
business operation
```

Каждый слой отвечает на другой вопрос.

---

# 40. TLS

TLS обычно отвечает:

```text
confidentiality
integrity
server identity
```

Для HTTPS caller проверяет certificate server side.

Internal traffic может быть:

- plain HTTP inside trusted network;
- HTTPS;
- mTLS via application;
- service mesh;
- platform-specific encryption.

Это architectural decision, не автоматическое свойство Service.

---

# 41. mTLS

Mutual TLS добавляет client certificate authentication.

```text
orders workload certificate
       |
       v
TLS handshake
       |
       v
customer verifies caller certificate
```

Это может дать workload identity.

Но mTLS не обязательно переносит end-user context.

---

# 42. JWT/OAuth2 Resource Server

Spring Boot backend может выступать Resource Server.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: \${OIDC_ISSUER_URI}
```

Conceptually:

```text
request
   |
Authorization: Bearer <token>
   |
   v
Spring Security
   |
   +--> signature
   +--> issuer
   +--> time claims
   +--> optional audience/custom claims
   |
   v
Authentication
```

Spring Security поддерживает Resource Server для JWT и opaque bearer tokens.

---

# 43. 401 и 403 — разные failure classes

Очень полезная диагностическая модель.

## 401 Unauthorized

Обычно:

```text
caller not successfully authenticated
```

Примеры:

- no token;
- invalid token;
- expired token;
- wrong issuer;
- bad signature.

## 403 Forbidden

Обычно:

```text
caller authenticated
but action not authorized
```

Например token valid, но нет required authority.

---

# 44. Workload identity и end-user identity

В enterprise request могут существовать две identity одновременно.

```text
User 123
   |
   v
orders-api workload
   |
   v
customer-api
```

Customer API может хотеть знать:

```text
workload identity:
  orders-api

user identity:
  user=123
  roles=...
```

Это решается архитектурно через:

- propagated user token;
- token exchange;
- service credential;
- mTLS/workload identity;
- combinations.

Не пытайтесь решить всё одним source IP.

---

# 45. ServiceAccount и RBAC — другой security plane

ServiceAccount:

```text
Pod
 |
 | Kubernetes API credential
 v
API Server
 |
 v
RBAC
```

Это identity Pod **для Kubernetes API**.

Это не означает автоматически:

```text
customer-api knows
"caller is orders-api"
```

для business HTTP request.

---

# 46. Когда отключать ServiceAccount token automount

Типичный Spring REST backend не вызывает Kubernetes API.

Тогда:

```yaml
spec:
  automountServiceAccountToken: false
```

уменьшает unnecessary credential exposure.

Если workload использует operator/API integration, token может быть нужен — тогда даём dedicated ServiceAccount + least-privilege RBAC.

---

# 47. Полная диагностическая лестница internal call

Пусть:

```text
orders-api -> customer-api
```

не работает.

Идём по слоям.

## Step 1 — имя

```bash
kubectl get svc customer-api
```

## Step 2 — DNS

```bash
kubectl exec <debug-pod> -- nslookup customer-api
```

## Step 3 — endpoints

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api
```

## Step 4 — target Pods/readiness

```bash
kubectl get pod -l app=customer-api -o wide
kubectl describe pod <pod>
```

## Step 5 — network

```bash
kubectl get networkpolicy
```

и connection test.

## Step 6 — HTTP

```text
connection established?
HTTP response exists?
```

## Step 7 — security

```text
TLS?
401?
403?
```

---

# 48. Symptom matrix

| Symptom | Первый слой для проверки |
|---|---|
| `NXDOMAIN` | Service name / namespace / DNS |
| DNS timeout | DNS reachability / egress policy / cluster DNS |
| Service exists, no endpoints | selector / Pods / readiness |
| connection refused | backend listening / targetPort / endpoint |
| connect timeout | NetworkPolicy / routing / dependency path |
| TLS handshake error | certificates / trust / hostname / protocol |
| HTTP 401 | authentication/token |
| HTTP 403 | authorization |
| HTTP 404 | HTTP path/application/route |
| intermittent failures | subset of endpoints / rollout / readiness / dependency |

Это не абсолютная истина, но очень хорошая стартовая классификация.

---

# 49. Failure lab — selector mismatch

Service:

```yaml
selector:
  app: customer-api
```

Pod:

```yaml
labels:
  app: customer-api-broken
```

Проверяем:

```bash
kubectl get svc customer-api
kubectl get pod --show-labels
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api
```

Expected:

```text
Service exists
DNS exists
matching endpoints absent
```

---

# 50. Failure lab — wrong targetPort

Service:

```yaml
ports:
  - port: 8080
    targetPort: wrong-port
```

Pod:

```yaml
ports:
  - name: http
    containerPort: 8080
```

Mapping broken:

```text
targetPort=wrong-port
   X
containerPort.name=http
```

Expected symptom depends on resulting configuration/dataplane, but traffic cannot correctly reach intended application port.

Диагностика:

```bash
kubectl describe svc customer-api
kubectl get pod -o yaml
```

---

# 51. Failure lab — readiness removes backends

Break:

```yaml
readinessProbe:
  httpGet:
    path: /wrong
    port: http
```

Observe:

```bash
kubectl get pod -w
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api -w
```

Expected mental flow:

```text
container Running
   ↓
readiness failing
   ↓
Ready=False
   ↓
endpoint not ready
```

---

# 52. Failure lab — DNS blocked by NetworkPolicy

Apply default deny egress to orders-api without DNS allow.

Expected:

```text
customer-api Service exists
customer Pods healthy
       |
       X
DNS query cannot reach resolver
       |
       v
name resolution fails
```

Проверка:

```bash
kubectl exec <orders-pod> -- nslookup customer-api
kubectl get networkpolicy
```

Затем inspect cluster DNS и добавьте explicit allow according to your platform.

---

# 53. Failure lab — network allowed, JWT invalid

Сначала убедитесь:

```text
DNS works
TCP works
HTTP reaches customer-api
```

Отправить invalid token.

Expected:

```text
HTTP 401
```

Это демонстрирует:

```text
network layer healthy
application authentication failed
```

---

# 54. Failure lab — valid identity, forbidden action

Valid JWT без required authority.

Expected:

```text
HTTP 403
```

Это уже authorization.

Так тестировщик строит layered security tests вместо “API не работает”.

---

# 55. Для аналитика

Аналитик должен описывать integration contract структурно.

Пример:

| Поле | Значение |
|---|---|
| Caller | orders-api |
| Caller namespace | sales |
| Target | customer-api |
| Target namespace | crm |
| Protocol | HTTPS |
| Service port | 8443 |
| Authentication | OAuth2 bearer token |
| Required audience | customer-api |
| Authorization | `customer.read` |
| Timeout | connect 1s / read 3s |
| Retry | GET only, bounded |
| Network flow | sales/orders -> crm/customer:8443 |

Это гораздо полезнее, чем:

> orders вызывает customer.

---

# 56. Для разработчика

Checklist:

- [ ] URL использует Service DNS, не Pod IP;
- [ ] namespace contract понятен;
- [ ] connect/read timeout заданы;
- [ ] readiness semantics корректны;
- [ ] named ports понятны;
- [ ] internal call не идёт через public edge без причины;
- [ ] authentication separate от NetworkPolicy;
- [ ] token audience/issuer/scopes валидируются;
- [ ] Kubernetes API token отключён, если не нужен;
- [ ] errors/metrics различают DNS/TCP/TLS/HTTP auth.

---

# 57. Для тестировщика

Нужно тестировать слои отдельно.

```text
1 DNS
2 Service selection
3 readiness/endpoints
4 TCP reachability
5 TLS
6 authentication
7 authorization
8 business behavior
```

Если тест одновременно ломает пять слоёв, диагностика почти ничего не обучает.

---

# 58. Для platform engineer

Platform должен документировать:

- cluster domain;
- DNS implementation;
- CNI;
- NetworkPolicy support;
- service dataplane;
- namespace conventions;
- ingress/gateway architecture;
- TLS/workload identity platform;
- debug tooling;
- allowed egress model.

Developer не должен угадывать platform behavior.

---

# 59. Anti-patterns

1. Pod IP в `application.yaml`.
2. `localhost` для другого Deployment.
3. Service selector, не совпадающий с Pod labels.
4. Hardcoded numbers без ясного port mapping.
5. Считать Service authentication mechanism.
6. Считать ClusterIP security boundary.
7. Default-deny egress без DNS planning.
8. Считать NetworkPolicy JWT authorization.
9. Выдавать broad Kubernetes RBAC ради internal HTTP call.
10. Гонять internal traffic через public ingress без архитектурной причины.
11. Диагностировать `401` через NetworkPolicy.
12. Restart Pods до проверки EndpointSlice.

---

# 60. Связь с предыдущей главой

В главе 01 мы написали:

```yaml
CUSTOMER_API_URL: http://customer-api:8080
```

Теперь цепочка раскрыта:

```text
Spring property
   ↓
customer-api DNS
   ↓
Service
   ↓
EndpointSlice
   ↓
Ready Pod
```

То есть configuration и networking больше не отдельные темы — они образуют один runtime contract.

---

# 61. Что изучать дальше

Следующая глава:

- [03 — Deployments, probes and rollouts](03-deployments-probes-rollouts.md)

Логика перехода:

```text
Глава 02:
Service отправляет traffic только в usable backends

Следующий вопрос:
кто создаёт эти Pods?
как новая версия заменяет старую?
когда Pod становится Ready?
что происходит при failed rollout?

        ↓

Глава 03
```

---

# 62. Production-like examples

## Internal REST

- [Showcase 01](../../showcases/01-internal-rest-service/README.md)
- [annotated.yaml](../../showcases/01-internal-rest-service/annotated.yaml)

## Service-to-service security

- [Showcase 06](../../showcases/06-service-to-service-security/README.md)
- [annotated.yaml](../../showcases/06-service-to-service-security/annotated.yaml)

## Headless Service

- [Showcase 18](../../showcases/18-statefulset-headless-service/README.md)
- [annotated.yaml](../../showcases/18-statefulset-headless-service/annotated.yaml)

## Gateway / edge

- [Showcase 07](../../showcases/07-gateway-microservices/README.md)

---

# 63. Control questions

## Service / DNS

1. Почему Pod IP нельзя использовать как service contract?
2. Как Service находит Pods?
3. Должно ли имя Deployment совпадать с Service?
4. Чем `port` отличается от `targetPort`?
5. Что даёт named port?
6. Что такое ClusterIP?
7. Кто поддерживает EndpointSlice для selector-based Service?
8. Почему Running Pod может не использоваться Service?
9. Как readiness связана с EndpointSlice?
10. Как обратиться к Service в другом namespace?
11. Что такое cluster domain?
12. Чем headless Service отличается от обычного?

## Network

13. Почему NetworkPolicy может существовать и ничего не enforcement?
14. Что происходит без isolation policies?
15. Почему default-deny egress ломает DNS?
16. Почему NetworkPolicy policies additive?
17. Может ли стандартная NetworkPolicy проверить JWT?
18. Почему ClusterIP не является authentication boundary?

## Security

19. Что решает TLS?
20. Что добавляет mTLS?
21. Что проверяет JWT Resource Server?
22. Чем 401 отличается от 403?
23. Чем ServiceAccount отличается от application service identity?
24. Когда отключать automount ServiceAccount token?

## Troubleshooting

25. Что означает NXDOMAIN?
26. Что означает DNS timeout?
27. Что проверить, если EndpointSlice пуст?
28. Что проверить при connection refused?
29. Что проверить при connect timeout?
30. Что проверить при 401?
31. Что проверить при 403?

---

# 64. Sources

- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/concepts/security/service-accounts/
- https://docs.spring.io/spring-security/reference/servlet/oauth2/
- https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/

Актуальные facts, использованные в главе:

- Kubernetes DNS даёт Services стабильные DNS names; short-name resolution зависит от namespace/search path.
- EndpointSlice — stable API с Kubernetes 1.21; control plane автоматически создаёт EndpointSlices для Services с selectors. Legacy Endpoints API deprecated с 1.33.
- NetworkPolicy работает на L3/L4 model и требует network implementation, которое реально enforcement policies; сам API object без такого implementation эффекта не гарантирует.
- Default-deny egress блокирует DNS, пока DNS flow не разрешён отдельно.
- Standard NetworkPolicy API не является TLS/JWT policy engine и не поддерживает explicit deny ordering model.
- Spring Security поддерживает OAuth2 Resource Server для JWT и opaque bearer tokens; issuer-based JWT configuration валидирует issuer и token signature/time constraints according to Resource Server configuration.
