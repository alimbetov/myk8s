# 00 — Platform baseline: как Spring Boot приложение живёт в Kubernetes

Проверено: 2026-09-20.

Эта глава — фундамент всего учебника.

Её цель не научить вас писать YAML наизусть. После неё вы должны понимать **что именно меняется в способе запуска и эксплуатации приложения**, когда Spring Boot переезжает с VM/systemd в Kubernetes.

Если вы аналитик, разработчик или тестировщик и раньше почти не работали с Kubernetes — начинайте отсюда.

---

# 1. Что вы должны понять после этой главы

После главы вы должны уметь своими словами объяснить:

```text
Что такое Pod?
Зачем нужен Deployment?
Почему нельзя использовать Pod IP?
Что решает Service?
Откуда Spring Boot получает configuration?
Где должны храниться secrets?
Что означает Ready?
Кто перезапускает container?
Где живут persistent data?
Что делает Ingress/Gateway?
Чем NetworkPolicy отличается от JWT?
Что происходит после kubectl apply?
Где искать проблему, если сервис недоступен?
```

Главный mental model:

```text
Мы больше не управляем "сервером с приложением".

Мы описываем desired state,
а Kubernetes постоянно поддерживает его.
```

---

# 2. Одна картинка всей системы

Сначала посмотрите на всю систему целиком.

```text
                         USERS / OTHER SYSTEMS
                                  |
                                  v
                      +-----------------------+
                      | Ingress / Gateway     |
                      | external entry point  |
                      +-----------+-----------+
                                  |
                                  v
                      +-----------------------+
                      | Service               |
                      | stable DNS / port     |
                      +-----------+-----------+
                                  |
                         selects Ready Pods
                                  |
                  +---------------+---------------+
                  |                               |
                  v                               v
          +---------------+               +---------------+
          | Pod orders-1  |               | Pod orders-2  |
          |               |               |               |
          | Spring Boot   |               | Spring Boot   |
          +-------+-------+               +-------+-------+
                  |                               |
                  +---------------+---------------+
                                  |
          +-----------------------+------------------------+
          |                       |                        |
          v                       v                        v
      PostgreSQL                Kafka                     S3
      Service/Operator          Service/Operator          endpoint

Pod получает runtime inputs:
  ConfigMap
  Secret
  ServiceAccount
  resources
  probes
  volumes

Kubernetes platform обеспечивает:
  scheduling
  reconciliation
  DNS
  networking
  lifecycle
  resource isolation
```

Не пытайтесь пока запоминать каждый объект. В следующих разделах мы будем добавлять их **по одной причине за раз**.

---

# 3. С чего мы пришли: Spring Boot на VM

Представим обычный сервис `orders-service`.

До Kubernetes он мог выглядеть так:

```text
VM 10.20.10.15
 |
 +-- /opt/orders/orders.jar
 |
 +-- /opt/orders/application.properties
 |
 +-- systemd
 |     |
 |     +--> java -jar orders.jar
 |
 +-- logs/
 |
 +-- connection:
       jdbc:postgresql://10.20.20.11:5432/orders
```

Systemd unit:

```ini
[Service]
ExecStart=/usr/bin/java -jar /opt/orders/orders.jar
Restart=always
```

Spring properties:

```properties
server.port=8080
spring.datasource.url=jdbc:postgresql://10.20.20.11:5432/orders
spring.datasource.username=orders_app
spring.datasource.password=secret123
```

## Какие скрытые предположения здесь существуют

Часто приложение и команда начинают считать, что:

```text
server существует долго
IP почти постоянен
filesystem "наш"
log file останется на машине
restart делает systemd/admin
configuration лежит рядом с JAR
database имеет фиксированный IP
application instance имеет стабильную identity
```

На одной VM такая модель может долго работать.

Kubernetes сознательно ломает многие из этих предположений.

---

# 4. Главная смена мышления: instance disposable

В Kubernetes отдельный экземпляр приложения **не должен быть драгоценным**.

Представьте:

```text
orders Pod A
   |
   X   node failure / rollout / eviction / manual delete
   |
   v
orders Pod B
```

Новый Pod может получить:

- другое имя;
- другой IP;
- другой node;
- новый container filesystem.

При этом пользователю желательно вообще не заметить replacement.

## Почему Kubernetes так устроен

Потому что platform работает не вокруг конкретного процесса:

```text
"следи, чтобы PID 15822 жил"
```

а вокруг desired state:

```text
"у меня должны быть 3 экземпляра orders приложения"
```

Это принцип reconciliation.

---

# 5. Desired state и reconciliation

Пусть мы описали:

```yaml
spec:
  replicas: 3
```

Это не команда:

> создай три процесса один раз.

Это statement:

> desired state = три подходящих replicas.

Runtime:

```text
Desired: 3
Actual: 3
   |
   +--> ничего делать не надо


Desired: 3
Actual: 2
   |
   v
controller creates replacement


Desired: 3
Actual: 4
   |
   v
controller removes excess replica
```

Вот почему Kubernetes называют declarative system.

---

# 6. Кто внутри Kubernetes что делает

Новичку очень важно не представлять Kubernetes как один большой процесс.

Упрощённо:

```text
                    CONTROL PLANE
+------------------------------------------------+
|                                                |
| API Server                                     |
|   "принимаю manifests и являюсь API входом"    |
|                                                |
| Scheduler                                      |
|   "решаю, на какой node поставить новый Pod"   |
|                                                |
| Controllers                                    |
|   "сравниваю desired и actual state"            |
|                                                |
+----------------------+-------------------------+
                       |
                       |
              Kubernetes API
                       |
        +--------------+--------------+
        |                             |
        v                             v
+---------------+             +---------------+
| Worker Node 1 |             | Worker Node 2 |
|               |             |               |
| kubelet       |             | kubelet       |
| container     |             | container     |
| runtime       |             | runtime       |
| Pods          |             | Pods          |
+---------------+             +---------------+
```

## API Server

Через него проходят API operations:

```text
kubectl apply
kubectl get
controllers
operators
kubelets
```

Он не запускает ваш Java process сам.

## Scheduler

Решает:

> на какой node назначить Pod?

Смотрит, среди прочего, на:

- resource requests;
- node selectors;
- affinity;
- taints/tolerations;
- topology constraints.

## Controllers

Следят за desired state.

Например Deployment controller:

```text
Deployment
   ↓
ReplicaSet
   ↓
Pods
```

## kubelet

Работает на worker node и отвечает за реализацию Pod spec на этом node:

- запускает containers через runtime;
- выполняет probes;
- монтирует volumes;
- сообщает status.

### Не путать

```text
Scheduler
= выбирает node

kubelet
= запускает Pod на уже выбранном node

Deployment controller
= следит, чтобы desired replicas существовали
```

---

# 7. Первый объект: Pod

Теперь возникает конкретная задача:

> Где вообще запускается Spring Boot process?

Ответ: внутри container, который находится в Pod.

```text
Pod
 |
 +-- container: app
       |
       +-- java
             |
             +-- Spring Boot
```

Пример:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: orders
spec:
  containers:
    - name: app
      image: registry.example/orders:1.0.0
```

## Что здесь важно

### `image`

Это package приложения:

```text
JRE
+
orders application
+
runtime filesystem
+
startup command
```

### Pod — не VM

Pod может иметь:

- один main container;
- initContainers;
- sidecars;
- shared volumes;
- shared network namespace.

Но типичный Spring REST backend:

```text
one Pod
  -> one application container
  -> one JVM
```

---

# 8. Почему нельзя создавать одиночный Pod вручную в production

Если мы создадим только:

```yaml
kind: Pod
```

и удалим его:

```bash
kubectl delete pod orders
```

ничто не обязано создать его заново.

Нам нужен controller.

---

# 9. Deployment: кто поддерживает Spring Boot replicas

Теперь решаем следующую проблему:

> Я хочу, чтобы всегда существовало несколько экземпляров orders-service.

Используем Deployment.

```text
Deployment orders
      |
      v
ReplicaSet
      |
      +--> Pod orders-abc
      +--> Pod orders-def
      +--> Pod orders-ghi
```

Учебный manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 3

  selector:
    matchLabels:
      app: orders

  template:
    metadata:
      labels:
        app: orders

    spec:
      containers:
        - name: app
          image: registry.example/orders:1.0.0
```

## Главная связь

```text
Deployment.spec.selector
          =
Pod template labels
```

Именно так Deployment понимает, какие Pods относятся к нему.

## Что произойдёт, если один Pod удалить

```text
replicas desired = 3

Pod deleted
   ↓
actual = 2
   ↓
ReplicaSet sees mismatch
   ↓
new Pod created
   ↓
actual = 3
```

---

# 10. Но появляется следующая проблема: Pod IP меняется

Допустим customer-service хочет вызвать orders-service.

Плохая идея:

```text
http://10.42.3.17:8080
```

Почему?

```text
Pod A
10.42.3.17
   |
   X deleted
   |
   v
Pod B
10.42.7.22
```

Caller потеряет endpoint.

Нужна stable logical identity.

---

# 11. Service: постоянный адрес перед временными Pods

Создаём Service.

```text
customer-service
      |
      | http://orders:8080
      v
Service orders
      |
      | selector app=orders
      v
EndpointSlice
      |
      +--> orders Pod A
      +--> orders Pod B
      +--> orders Pod C
```

Manifest:

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

Deployment:

```yaml
containers:
  - name: app
    ports:
      - name: http
        containerPort: 8080
```

## Здесь сразу две связи

```text
Service.selector
      =
Pod labels
```

и:

```text
Service.targetPort=http
      =
containerPort.name=http
```

### Важный вывод

Service **не ищет Deployment по имени**.

Можно иметь:

```text
Deployment = orders-backend-v2
Service    = orders
```

Это нормально, если labels совпадают.

---

# 12. DNS: как Spring Boot находит Service

Kubernetes DNS даёт Service logical hostname.

В том же namespace:

```text
http://orders:8080
```

В другом namespace:

```text
http://orders.sales:8080
```

Полная форма:

```text
orders.sales.svc.cluster.local
```

Spring configuration:

```yaml
app:
  orders:
    base-url: ${ORDERS_URL:http://orders:8080}
```

Это намного устойчивее, чем Pod IP.

---

# 13. EndpointSlice: что находится между Service и Pod

Для новичка часто магическим выглядит вопрос:

> Как Service узнаёт адреса Pods?

Упрощённая цепочка:

```text
Pod labels + readiness
        |
        v
EndpointSlice controller
        |
        v
EndpointSlice
        |
        +--> 10.42.1.4 ready=true
        +--> 10.42.2.8 ready=true
        +--> 10.42.3.9 ready=false
        |
        v
Service traffic
```

Поэтому диагностика Service почти всегда включает:

```bash
kubectl get svc orders
kubectl get endpointslice   -l kubernetes.io/service-name=orders
```

---

# 14. Running не означает Ready

Spring process может существовать, но ещё не быть готовым обслуживать traffic.

Например:

```text
JVM started
   ↓
Spring context starts
   ↓
Flyway/Liquibase
   ↓
cache warmup
   ↓
HTTP server initialized
   ↓
application ready
```

Kubernetes различает:

```text
Running
  process/container существует

Ready
  можно отправлять обычный traffic
```

---

# 15. Probes: три разных вопроса

```text
startupProbe
  "успело ли приложение нормально стартовать?"

livenessProbe
  "этот process сломан настолько, что его надо restart?"

readinessProbe
  "можно ли сейчас отправлять сюда traffic?"
```

Пример:

```yaml
startupProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 5
  failureThreshold: 30

livenessProbe:
  httpGet:
    path: /livez
    port: http

readinessProbe:
  httpGet:
    path: /readyz
    port: http
```

## Как readiness влияет на Service

```text
GET /readyz
   |
   +--> 200
   |      ↓
   |   Pod Ready=True
   |      ↓
   |   EndpointSlice ready=true
   |      ↓
   |   Service can send traffic
   |
   +--> failure
          ↓
       Pod may remain Running
          ↓
       Ready=False
          ↓
       endpoint not ready
```

### Очень важное правило

Не делайте liveness напрямую зависимой от PostgreSQL/Kafka/external HTTP.

Иначе:

```text
PostgreSQL outage
   ↓
all app liveness fails
   ↓
all containers restart
   ↓
reconnect storm
   ↓
database recovery becomes harder
```

---

# 16. Следующая проблема: где хранить configuration

Раньше:

```text
/opt/orders/application.properties
```

В Kubernetes мы хотим:

```text
same image
  |
  +--> dev config
  +--> test config
  +--> prod config
```

Image должен быть одинаковым.

Configuration приходит runtime.

---

# 17. ConfigMap и Secret

Упрощённое разделение:

```text
ConfigMap
  -> URLs
  -> timeouts
  -> feature flags
  -> non-sensitive settings

Secret
  -> passwords
  -> API keys
  -> private keys
  -> credentials
```

Пример ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
data:
  CUSTOMER_API_URL: http://customer-api:8080
  DB_POOL_SIZE: "10"
```

Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-db
type: Opaque
stringData:
  username: orders_app
  password: replace-at-deploy-time
```

Pod:

```yaml
envFrom:
  - configMapRef:
      name: orders-config
  - secretRef:
      name: orders-db
```

Spring:

```yaml
spring:
  datasource:
    username: ${username}
    password: ${password}
```

## Но Secret не означает “автоматически безопасно”

Нужно сразу запомнить:

```text
base64 != encryption

Secret object
!=
full secret lifecycle
```

Полную тему мы разберём в главе 17.

---

# 18. Где реально хранить passwords и keys

Упрощённая production chain:

```text
Vault / cloud secret manager / protected CI
          |
          v
Kubernetes Secret / CSI projection
          |
          v
Pod
          |
          v
Spring Boot
```

А storage внутри Kubernetes может дополнительно защищаться encryption-at-rest/KMS на уровне platform.

Разделяйте:

```text
application password
!=
Kubernetes encryption key
```

Разработчик обычно управляет первым contract, platform/security team — вторым.

---

# 19. Следующая проблема: resources не бесконечны

На VM JVM могла практически конкурировать со всем сервером.

В Kubernetes Pod должен иметь resource contract.

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    memory: 768Mi
```

## requests

Используются для scheduling/capacity planning.

## limits

Runtime ceiling.

Для Java:

```text
memory limit
  !=
heap size
```

Потому что JVM использует:

```text
heap
metaspace
thread stacks
direct buffers
code cache
native memory
```

---

# 20. Resources связаны с autoscaling

CPU request — это не только scheduler value.

HPA Utilization примерно опирается на:

```text
actual CPU
-----------
CPU request
```

Например:

```text
actual = 350m
request = 500m
=> ~70%
```

Если изменить request на 250m:

```text
350 / 250 = 140%
```

То есть одно поле Deployment меняет autoscaling semantics.

---

# 21. Replicas связаны с connection pools

Допустим:

```text
Hikari max pool = 10
```

Тогда:

```text
3 Pods
× 10
= up to ~30 DB connections
```

А HPA увеличил до 20:

```text
20 × 10
= up to ~200 DB connections
```

Вот почему Kubernetes нельзя изучать только как YAML.

```text
HPA
 -> more Pods
 -> more JVMs
 -> more pools
 -> more DB connections
 -> possible PostgreSQL saturation
```

---

# 22. Persistent data: что нельзя хранить в container filesystem

Плохая модель:

```text
Pod
  |
  +--> /app/uploads/customer.pdf
```

Pod заменился:

```text
new Pod
  |
  +--> empty filesystem
```

Для state используют:

```text
PostgreSQL
Kafka
RabbitMQ
Object storage
PVC
```

в зависимости от semantics.

---

# 23. PVC: если приложению действительно нужен filesystem

Storage chain:

```text
Container
   |
   | /data
   v
volumeMount
   |
   v
Pod volume
   |
   v
PVC
   |
   v
PV
   |
   v
StorageClass / CSI
   |
   v
physical storage
```

Но:

```text
PVC persistence
!=
backup
```

И:

```text
StatefulSet
!=
database HA
```

---

# 24. StatefulSet: когда identity тоже имеет значение

Для обычного REST API Pod identity не важна.

Для некоторых distributed systems нужна stable identity:

```text
member-0
member-1
member-2
```

Тогда используется StatefulSet.

Он даёт:

```text
stable ordinal identity
stable storage association
ordered lifecycle
```

Но не реализует:

```text
leader election
replication
quorum
backup
product failover
```

Поэтому PostgreSQL/Kafka/RabbitMQ в production часто управляются product-aware Operators.

---

# 25. Operator: controller, который понимает конкретный продукт

Обычный Deployment controller знает:

> сколько Pods нужно.

CloudNativePG operator знает:

> какой PostgreSQL instance primary, как управлять replicas и failover.

Strimzi знает Kafka topology.

RabbitMQ Operator знает broker cluster lifecycle.

Схема:

```text
Custom Resource
      |
      v
Product Operator
      |
      v
Kubernetes resources
      |
      v
actual product cluster
```

---

# 26. Внешний вход: Ingress и Gateway

Internal Service сам по себе обычно не означает public Internet endpoint.

External flow:

```text
Internet
   |
   v
Load Balancer / Gateway / Ingress Controller
   |
   v
Ingress or HTTPRoute
   |
   v
Service
   |
   v
Ready Pods
```

Для новичка важно:

```text
Ingress/Gateway
!=
Service

Service
!=
Pod
```

Это разные слои.

---

# 27. Security — это несколько независимых механизмов

Представим orders вызывает payment.

```text
orders
  |
  v
payment
```

Нужно задать несколько разных вопросов.

## Может ли packet вообще пройти?

```text
NetworkPolicy
```

## Зашифрован ли transport?

```text
TLS
```

## Кто peer?

```text
mTLS / workload identity
```

## Кто application caller?

```text
OAuth2/JWT
```

## Что caller может делать?

```text
application authorization
```

## Может ли Pod читать Kubernetes API?

```text
ServiceAccount + RBAC
```

Это разные security planes.

---

# 28. ServiceAccount/RBAC не заменяют Spring Security

Очень частое заблуждение:

```text
Pod has ServiceAccount
      ↓
значит другой сервис знает, кто его вызвал
```

Нет.

ServiceAccount в первую очередь связан с Kubernetes API identity.

```text
Pod
 |
 | serviceAccount token
 v
Kubernetes API Server
 |
 v
RBAC
```

Business API authentication остаётся отдельной задачей.

---

# 29. Что происходит после kubectl apply

Теперь соберём процесс полностью.

```text
Developer / CI
      |
      | kubectl apply
      v
API Server
      |
      | validates + stores desired state
      v
Deployment Controller
      |
      v
ReplicaSet
      |
      v
Pod objects
      |
      v
Scheduler
      |
      | chooses node
      v
kubelet
      |
      | pulls image
      | mounts config/volumes
      v
container runtime
      |
      v
JVM / Spring Boot
      |
      v
startupProbe
      |
      v
readinessProbe
      |
      v
Pod Ready
      |
      v
EndpointSlice ready endpoint
      |
      v
Service can route traffic
```

Вот эта схема должна стать базовой картиной Kubernetes в голове.

---

# 30. Один complete Spring Boot example

Смотрите полный учебный manifest:

- [Showcase 01 README](../../showcases/01-internal-rest-service/README.md)
- [Showcase 01 annotated.yaml](../../showcases/01-internal-rest-service/annotated.yaml)
- [Showcase 01 clean all.yaml](../../showcases/01-internal-rest-service/all.yaml)
- [Showcase 01 walkthrough](../../showcases/01-internal-rest-service/WALKTHROUGH.md)

Читать его нужно так:

```text
Deployment labels
   ↓
Service selector
   ↓
named targetPort
   ↓
probes
   ↓
ConfigMap/Secret
   ↓
Spring Boot
```

---

# 31. Почему минимальный YAML недостаточен для production

Минимальный Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: app
          image: registry.example/orders:1.0.0
```

Он полезен для понимания primitive.

Но production обычно требует решения вопросов:

```text
How does it become Ready?
What are CPU/memory requirements?
How does it stop?
Where is config?
Where are secrets?
How does traffic reach it?
What can call it?
What can it call?
What happens on rollout?
Where are logs/metrics/traces?
What happens if DB is down?
```

**CKAD minimal != production complete.**

---

# 32. Failure thinking: не начинайте с restart

Если сервис недоступен, думайте слоями.

```text
1 Desired state
2 Scheduling
3 Image/container startup
4 Spring startup
5 Probes
6 Service/EndpointSlice
7 DNS/network
8 Authentication/authorization
9 Dependency
10 Storage
11 Rollout/schema
```

## Примеры

### Pod Pending

Вероятнее:

```text
resources
PVC
node selector
affinity
taints
```

а не Java code.

### ImagePullBackOff

```text
registry
image tag
credentials
network
```

### CrashLoopBackOff

```text
Spring startup exception
bad config
OOM/process exit
```

### Running 0/1

```text
container running
readiness failing
```

### Service exists, EndpointSlice empty

```text
selector mismatch
or
no matching Ready Pods
```

### DNS resolves, TCP timeout

```text
NetworkPolicy
wrong targetPort
backend not listening
dependency unavailable
```

### 401

Authentication.

### 403

Authorization.

---

# 33. Кто за что отвечает

Упрощённая responsibility map:

| Область | Аналитик | Developer | Tester | Platform/Infra |
|---|---:|---:|---:|---:|
| business/service contract | ✓ | ✓ | ✓ |  |
| Spring configuration contract |  | ✓ | ✓ |  |
| image |  | ✓ | ✓ | shared |
| Deployment/probes | understands | ✓ | ✓ | shared |
| Service/DNS | understands | ✓ | ✓ | cluster DNS |
| CNI/CSI | awareness |  | tests behavior | ✓ |
| secrets lifecycle | requirements | shared | ✓ | ✓ |
| authentication | requirements | ✓ | ✓ | IdP shared |
| NetworkPolicy | flow requirements | shared | ✓ | shared |
| database HA/backup | requirements | shared | verifies | DBA/operator |
| observability/SLO | ✓ | ✓ | ✓ | shared |

Реальные границы зависят от организации, но сама идея ownership должна быть явной.

---

# 34. Что должен понимать аналитик

После этой главы аналитик должен уметь читать такую схему:

```text
Browser
  ↓
Gateway
  ↓
orders Service
  ↓
orders Pods
  |
  +--> PostgreSQL
  +--> payment-api
  +--> Kafka
```

и задавать вопросы:

- orders внутренний или внешний?
- какой hostname/path?
- какое authentication?
- можно ли retry payment call?
- операция idempotent?
- что происходит, если Kafka недоступна?
- где хранится state?
- какой RPO/RTO?
- какое expected behavior при rollout?

Аналитику не обязательно сразу писать YAML.

Ему нужно понимать **архитектурные последствия требований**.

---

# 35. Что должен понимать разработчик

Разработчик должен начать воспринимать Spring Boot как Kubernetes-friendly process.

Checklist:

```text
external config
finite timeouts
typed properties
graceful shutdown
correct probes
no local persistent assumptions
stdout/stderr logs
resource awareness
retry discipline
idempotency where needed
schema compatibility
```

Главный contract:

```text
Kubernetes
  manages process lifecycle

Spring Boot
  must behave correctly inside that lifecycle
```

---

# 36. Что должен понимать тестировщик

Тестировщику Kubernetes даёт новые failure dimensions.

Недостаточно проверить:

```text
POST /orders -> 200
```

Нужно уметь проверять:

```text
delete Pod
wrong Secret
broken readiness
wrong Service selector
DNS blocked
DB unavailable
bad rollout
memory limit
consumer duplicate
primary failover
```

Тестирование становится ближе к resilience/system testing.

---

# 37. Мини-лаборатория после главы

Используйте [Lab 01](../../labs/01-spring-boot-baseline/README.md).

Минимальный learning loop:

```bash
kubectl get deployment
kubectl get pod -o wide
kubectl get service
kubectl get endpointslice
```

Удалите Pod:

```bash
kubectl delete pod <orders-pod>
```

Наблюдайте:

```bash
kubectl get pod -w
```

Ответьте:

1. Кто создал replacement?
2. Изменилось ли имя Pod?
3. Изменился ли IP?
4. Изменился ли Service DNS?
5. Почему caller не должен знать Pod IP?

---

# 38. Первая intentional failure simulation

Измените Service selector так, чтобы он больше не совпадал с Pods.

Например:

```yaml
selector:
  app: orders-wrong
```

Теперь:

```bash
kubectl get svc orders
kubectl get endpointslice   -l kubernetes.io/service-name=orders
kubectl get pod --show-labels
```

Ожидаемая логика:

```text
Service exists
DNS exists
Pods Running
but
Service has no matching endpoints
```

Это хороший пример:

> объект существует ≠ система работает.

---

# 39. Вторая intentional failure simulation

Сломайте readiness path:

```yaml
readinessProbe:
  httpGet:
    path: /wrong
    port: http
```

Ожидайте:

```text
container Running
   ↓
readiness fails
   ↓
Pod Ready=False
   ↓
Service endpoint not ready
```

Проверка:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice
```

---

# 40. Контрольные вопросы

Не переходите к следующей главе, пока не сможете ответить без подсказки.

## Базовые

1. Почему Pod нельзя считать VM?
2. Кто создаёт replacement Pod?
3. Кто выбирает node?
4. Кто запускает container на node?
5. Почему Service не использует Pod IP напрямую как application contract?
6. Чем Running отличается от Ready?
7. Зачем три probes?
8. Где должны храниться business data?
9. Чем ConfigMap отличается от Secret?
10. Почему Secret не равен “зашифрованному паролю”?

## Связи

11. Что должно совпасть между Service selector и Pod?
12. Что связывает `targetPort: http` с container?
13. Как readiness влияет на Service?
14. Почему HPA связан с CPU requests?
15. Почему replicas связаны с DB pool capacity?

## Security

16. Чем NetworkPolicy отличается от JWT?
17. Чем ServiceAccount отличается от application user identity?
18. Что защищает TLS?
19. Что решает authorization?

## Operations

20. Что происходит после `kubectl apply Deployment`?
21. Почему restart — плохой первый шаг диагностики?
22. Что проверить при `Running 0/1`?
23. Что проверить при Service без endpoints?
24. Что означает `CrashLoopBackOff` концептуально?

---

# 41. Что изучать дальше

Следующая глава:

- [01 — Spring Boot configuration and secrets](01-spring-boot-configuration-secrets.md)

Логика перехода:

```text
Глава 00:
"приложение живёт в disposable Pod"

Следующий вопрос:
"откуда тогда Pod получает environment-specific configuration
и как не положить password в image/Git?"
```

Именно это разбирает глава 01.

---

# 42. Связанные материалы

## Учебные стенды

- [01 — Internal REST service](../../showcases/01-internal-rest-service/README.md)
- [01 — annotated manifest](../../showcases/01-internal-rest-service/annotated.yaml)
- [02 — Public API / Ingress / TLS](../../showcases/02-public-api-ingress-tls/README.md)
- [03 — REST + PostgreSQL + HPA + NetworkPolicy](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)
- [18 — StatefulSet + headless Service](../../showcases/18-statefulset-headless-service/README.md)

## Reference

- [Deployment manifest reference](manifests/deployment.md)
- [Pod manifest reference](manifests/pod.md)
- [Service manifest reference](manifests/service.md)
- [ConfigMap manifest reference](manifests/configmap.md)
- [Secret manifest reference](manifests/secret.md)

## Labs

- [Lab 01 — Spring Boot baseline](../../labs/01-spring-boot-baseline/README.md)

---

# 43. Sources

Используйте sources для проверки и углубления после того, как понятна mental model.

- https://kubernetes.io/docs/concepts/overview/components/
- https://kubernetes.io/docs/concepts/workloads/pods/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://docs.spring.io/spring-boot/reference/

