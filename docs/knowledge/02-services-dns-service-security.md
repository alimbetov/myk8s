# 02 — Services, DNS and service-to-service security

## Учебная карта темы

### Где Service находится в request flow

```text
orders-api Pod
    |
    | http://customer-api:8080
    v
Cluster DNS
    |
    v
Service customer-api
    |
    v
EndpointSlice
    |
    +--> Ready Pod A
    +--> Ready Pod B
```

Service — это **stable network identity**, а не application server и не security mechanism.

### Главный mapping

```yaml
# Service
spec:
  selector:
    app: customer-api       # выбирает Pods по label
  ports:
    - port: 8080            # порт, который видит caller
      targetPort: http      # named container port
```

```text
Service.selector
      =
Pod labels

Service.targetPort=http
      =
containerPort.name=http
```

### Не путать слои

```text
Service       -> куда отправить
NetworkPolicy -> можно ли отправить
TLS/mTLS      -> защищён ли transport / кто peer
JWT/OAuth2    -> кто caller
Authorization -> что caller может делать
```

Практика: [Showcase 06 — service-to-service security](../../showcases/06-service-to-service-security/README.md).

## Для аналитика, разработчика и тестировщика

### Аналитик

Описывает service contracts через logical names и ports, а не IP. В интеграционной схеме должны быть видны:
- caller;
- target Service;
- protocol/port;
- namespace;
- security layer.

### Разработчик

Использует Service DNS в URLs, finite timeouts и readiness-aware design. Не привязывается к Pod IP и не использует localhost для другого workload.

### Тестировщик

Проверяет отдельно:
- DNS resolve;
- Service endpoints;
- port/targetPort;
- readiness;
- NetworkPolicy;
- 401/403 application layer.

### Перед следующей главой

Нужно уметь диагностически разделить:

```text
DNS problem
Service selection problem
network reachability problem
application authentication problem
```

Проверено: 2026-09-20.

Эта глава объясняет, **как один Spring Boot service находит другой**, почему Pod IP нельзя считать адресом сервиса и почему NetworkPolicy, TLS и JWT решают разные задачи.

## 1. Проблема: Pod IP нестабилен

```text
customer Pod A -> 10.42.1.15
Pod A crash
customer Pod B -> 10.42.2.31
```

Если orders хранит IP первого Pod, связь ломается.

Поэтому:

```text
orders Pod
  -> http://customer-api:8080
  -> cluster DNS
  -> Service customer-api
  -> EndpointSlice
  -> Ready customer Pods
```

Service — стабильная logical identity, Pods — изменяемые backends.

---

## 2. Service

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

Pod:

```yaml
metadata:
  labels:
    app: customer-api
```

Selector Service должен совпасть с labels Pods.

### Практическая диагностика

```bash
kubectl get svc customer-api
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api
```

Если Service есть, но EndpointSlice пустой, частые причины:
- selector mismatch;
- Pods not Ready;
- Pods вообще отсутствуют.

---

## 3. DNS

В том же namespace:

```text
http://customer-api:8080
```

В другом namespace:

```text
http://customer-api.crm:8080
```

FQDN:

```text
customer-api.crm.svc.cluster.local
```

Spring:

```yaml
clients:
  customer:
    base-url: ${CUSTOMER_API_URL:http://customer-api:8080}
```

В cluster используется DNS Service; локально env можно заменить.

---

## 4. DNS problem или Service problem?

```bash
kubectl exec <debug-pod> -- nslookup customer-api
kubectl exec <debug-pod> -- curl -v http://customer-api:8080/readyz
```

### DNS не resolves

Проверяем:
- имя Service;
- namespace;
- CoreDNS;
- ``/etc/resolv.conf``;
- egress NetworkPolicy к DNS.

### DNS resolves, HTTP не работает

Проверяем:
- EndpointSlice;
- targetPort;
- readiness;
- NetworkPolicy;
- приложение действительно слушает ожидаемый port/interface.

---

## 5. Почему localhost — не другой microservice

В Pod:

```text
localhost = network namespace этого Pod
```

``http://localhost:8081`` подходит для sidecar/другого container в том же Pod.

Для другого Deployment нужен Service DNS.

---

## 6. port и targetPort

```yaml
ports:
  - port: 80
    targetPort: http
```

- ``port`` — port Service.
- ``targetPort`` — backend port Pod.

Named port:

```yaml
containers:
  - ports:
      - name: http
        containerPort: 8080
```

позволяет Service ссылаться на ``targetPort: http``.

---

## 7. NetworkPolicy

NetworkPolicy отвечает:

> Может ли этот Pod установить или принять L3/L4 соединение?

Она не проверяет JWT и business role.

Если policies отсутствуют, Kubernetes NetworkPolicy model по умолчанию не изолирует traffic.

---

## 8. Default deny

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Важная деталь: DNS

Default-deny egress блокирует и DNS, если не добавить отдельное allow rule.

Симптом:

```text
UnknownHostException
```

хотя Service существует.

> **Практика:** сначала проектировать required flows, затем вводить deny, а не закрывать namespace и после этого хаотично открывать всё обратно.

---

## 9. Allow only orders -> customer

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: customer-ingress
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
              app: orders
      ports:
        - protocol: TCP
          port: 8080
```

Для cross-namespace flow нужно осознанно использовать ``namespaceSelector`` вместе с ``podSelector``.

---

## 10. NetworkPolicy != authentication

Разрешение network path означает только:

```text
TCP connection allowed
```

Скомпрометированный orders Pod тоже сможет использовать этот path.

Поэтому application authentication остаётся обязательной там, где нужна identity.

---

## 11. Security layers

```text
NetworkPolicy
  -> можно ли установить connection

TLS
  -> защищён ли channel

mTLS / workload identity
  -> какой workload на другой стороне

OAuth2/JWT
  -> кто authenticated principal

authorization
  -> какие действия разрешены

business authorization
  -> имеет ли principal доступ к конкретному объекту
```

Один слой не заменяет другой.

---

## 12. Spring Security Resource Server

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OIDC_ISSUER_URI}
```

API валидирует token и строит Authentication.

Service/DNS решает discovery, а не authentication.

---

## 13. mTLS и JWT вместе

mTLS может доказать:

```text
caller workload = orders-service
```

JWT может переносить:

```text
userId=123
roles=[OPERATOR]
scope=customer.read
```

В enterprise architecture часто полезны обе identity: workload и end user.

---

## 14. ServiceAccount и RBAC

ServiceAccount — identity Pod для Kubernetes API.

Обычному REST backend Kubernetes API часто вообще не нужен.

Тогда:

```yaml
automountServiceAccountToken: false
```

уменьшает unnecessary credential exposure.

Не выдавайте application ``get/list secrets`` только потому, что оно работает в Kubernetes. Injection конкретного Secret не требует, чтобы приложение само ходило в Kubernetes API.

---

## 15. Failure practice: selector mismatch

Service ожидает:

```yaml
selector:
  app: customer-api
```

Pod имеет:

```yaml
labels:
  app: customers-api
```

Диагностика:

```bash
kubectl get svc customer-api
kubectl get pod --show-labels
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api
```

DNS будет существовать, но endpoints не будет.

---

## 16. Failure practice: readiness убрала endpoint

Сломайте readiness и наблюдайте:

```bash
kubectl get pod -w
kubectl get endpointslice \
  -l kubernetes.io/service-name=customer-api -w
```

Так видно, как application health влияет на service routing.

---

## 17. Failure practice: NetworkPolicy deny

До policy:

```bash
kubectl exec orders-debug -- \
  curl http://customer-api:8080/readyz
```

После deny connection не проходит.

Диагностика:

```bash
kubectl get networkpolicy
kubectl describe networkpolicy
kubectl exec orders-debug -- nslookup customer-api
kubectl exec orders-debug -- \
  curl -v --connect-timeout 2 http://customer-api:8080/readyz
```

Разделяйте DNS resolution, TCP connectivity и HTTP security.

---

## 18. HTTP 401/403 — это уже не NetworkPolicy

```text
connection succeeds
HTTP 401
```

Обычно authentication problem.

```text
HTTP 403
```

Обычно authenticated principal не authorized.

Это простое разделение резко ускоряет troubleshooting.

---

## 19. Ingress и internal traffic

Internal:

```text
orders -> customer-api Service
```

External:

```text
client -> ingress/gateway/load balancer
       -> Service
       -> Pod
```

Не отправляйте internal calls через public ingress без явной причины.

---

## 20. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| Service contract/ports | ✓ |  | ✓ |
| application auth | ✓ |  | ✓ |
| CNI |  | ✓ |  |
| cluster DNS |  | ✓ |  |
| NetworkPolicy |  |  | ✓ |
| TLS infrastructure |  | ✓ | ✓ |
| authorization rules | ✓ |  | ✓ |
| Kubernetes RBAC |  | ✓ | ✓ |

---

## 21. Anti-patterns

- Pod IP в properties;
- localhost для другого Deployment;
- считать ClusterIP security boundary;
- считать NetworkPolicy заменой JWT;
- default-deny egress без DNS allowance;
- широкие RBAC permissions;
- internal traffic через public ingress без причины;
- authorization только по source network.

---

## 22. CKAD mapping

Нужно быстро уметь:

```bash
kubectl expose deployment
kubectl get svc
kubectl get endpointslice
kubectl describe svc
kubectl get networkpolicy
kubectl exec
```

Production глубже CKAD в OAuth2, mTLS, PKI, workload identity и egress governance.

---

## Sources: для проверки, а не вместо объяснения

- https://kubernetes.io/docs/concepts/services-networking/service/ — Service abstraction.
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/ — DNS naming.
- https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/ — service backends.
- https://kubernetes.io/docs/concepts/services-networking/network-policies/ — isolation/default deny.
- https://kubernetes.io/docs/concepts/security/service-accounts/ — Kubernetes workload identity.
- https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/index.html — JWT/OAuth2 Resource Server.

Проверено: **2026-09-20**.
