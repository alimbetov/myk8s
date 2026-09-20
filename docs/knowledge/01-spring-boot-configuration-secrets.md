# 01 — Spring Boot configuration, ConfigMap, Secret и lifecycle секретов

Проверено: 2026-09-20.

Эта глава отвечает на следующий вопрос после platform baseline:

> Если Spring Boot Pod disposable и image должен быть одинаковым для dev/test/prod, **откуда приложение получает environment-specific configuration и как безопасно передавать passwords, tokens и keys?**

Цель — не просто научиться писать `envFrom`. Нужно построить правильную mental model:

```text
configuration source
        ↓
Kubernetes delivery mechanism
        ↓
Spring Environment
        ↓
typed application contract
        ↓
runtime client / datasource
```

А для secrets цепочка длиннее:

```text
credential source
      ↓
Kubernetes Secret / external projection
      ↓
optional encryption at rest
      ↓
Pod delivery
      ↓
Spring Boot
      ↓
credential reload / rotation
```

---

# 1. Что вы должны понимать после главы

После этой главы вы должны уметь объяснить:

1. Почему один и тот же container image должен использоваться в разных environments.
2. Какие значения должны лежать в `application.yaml`.
3. Что нужно вынести в ConfigMap.
4. Что нужно считать Secret.
5. Почему Base64 не является encryption.
6. Почему Kubernetes Secret **не обязательно зашифрован в etcd**.
7. Где хранится encryption key, если cluster использует encryption at rest.
8. Чем local encryption key отличается от KMS.
9. Как Secret попадает в Pod: env, `envFrom`, mounted file, `configtree:`.
10. Почему изменение Secret не означает автоматический reload Spring Boot.
11. Как безопасно rotating DB password/certificate/token.
12. Почему `@ConfigurationProperties` лучше большого набора `@Value`.
13. Как configuration влияет на HTTP clients, pools, retries и observability.
14. Что должен проверять аналитик, разработчик и тестировщик.

---

# 2. Одна большая схема configuration path

```text
                    SOURCE OF CONFIGURATION
                            |
            +---------------+----------------+
            |                                |
            v                                v
       NON-SECRET                         SECRET
     configuration                       material
            |                                |
            v                                v
        ConfigMap                    Vault / cloud secret
            |                         manager / CI secret
            |                                |
            |                                v
            |                         Kubernetes Secret
            |                                |
            +---------------+----------------+
                            |
                            v
                         Pod spec
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
        env              mounted file       configtree
          |                 |                  |
          +-----------------+------------------+
                            |
                            v
                     Spring Environment
                            |
                            v
                 @ConfigurationProperties
                            |
                            v
             Hikari / HTTP client / Kafka / ...
```

Главная идея:

> **Container image хранит code и безопасные defaults. Environment-specific значения приходят во время deployment/runtime.**

---

# 3. Как было раньше на VM

```text
/opt/orders/
├── orders.jar
├── application.properties
├── application-prod.properties
└── secret.properties
```

Внутри:

```properties
spring.datasource.url=jdbc:postgresql://10.20.30.15:5432/orders
spring.datasource.username=orders
spring.datasource.password=SuperSecret123
payment.url=http://10.20.40.12:8080
```

Проблема не в самом properties-файле. Проблема — в смешивании code, environment config и credentials.

```text
code change
  ↓
new JAR

config change
  ↓
manual file edit / drift

password rotation
  ↓
manual operation on servers
```

---

# 4. Целевая модель: build once, configure at runtime

```text
                   SAME IMAGE
            registry/orders:1.4.2
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
       DEV            TEST           PROD
        |              |              |
  ConfigMap dev  ConfigMap test  ConfigMap prod
  Secret dev     Secret test     Secret prod
```

Image один и тот же. Меняются URL, credentials, timeouts, pool size, feature flags и certificates.

---

# 5. Что оставлять в application.yaml

Хорошая роль `application.yaml`:

1. структура configuration;
2. safe defaults;
3. placeholders для external values;
4. documentation runtime contract.

```yaml
spring:
  application:
    name: orders-api

  datasource:
    url: \${DB_URL}
    username: \${DB_USERNAME}
    password: \${DB_PASSWORD}

    hikari:
      maximum-pool-size: \${DB_POOL_SIZE:10}
      connection-timeout: \${DB_POOL_CONNECTION_TIMEOUT:3s}

app:
  customer-api:
    base-url: \${CUSTOMER_API_URL:http://customer-api:8080}
    connect-timeout: \${CUSTOMER_API_CONNECT_TIMEOUT:1s}
    read-timeout: \${CUSTOMER_API_READ_TIMEOUT:3s}
```

Почему URL может иметь safe default, а password — нет:

```text
CUSTOMER_API_URL missing
  -> safe internal Service default possible

DB_PASSWORD missing
  -> should fail fast
```

Плохой default:

```yaml
password: \${DB_PASSWORD:admin123}
```

---

# 6. Spring Environment — центральная точка

Упрощённая модель:

```text
application.yaml
environment variables
system properties
external config imports
configtree
        |
        v
Spring Environment
        |
        v
@ConfigurationProperties / @Value
```

Spring Environment — место, где Kubernetes-delivered values становятся application configuration.

---

# 7. Почему @ConfigurationProperties лучше десятков @Value

Плохо:

```java
@Value("\${app.customer-api.base-url}")
private String baseUrl;

@Value("\${app.customer-api.connect-timeout}")
private Duration connectTimeout;
```

Лучше:

```java
@ConfigurationProperties(prefix = "app.customer-api")
@Validated
public record CustomerApiProperties(
        @NotNull URI baseUrl,
        @NotNull Duration connectTimeout,
        @NotNull Duration readTimeout
) {}
```

Получаем:

- typed contract;
- validation;
- понятную группу settings;
- проще unit tests;
- меньше scattered string keys.

Runtime chain:

```text
ConfigMap/env
     ↓
Spring Environment
     ↓
CustomerApiProperties
     ↓
HTTP client configuration
```

---

# 8. Typed configuration = runtime safety

Requirement:

> Customer API connect timeout = 2 seconds.

Deployment:

```yaml
CUSTOMER_API_CONNECT_TIMEOUT: 2s
```

Spring:

```text
"2s"
  ↓
Duration
  ↓
typed validated value
```

Если:

```text
CUSTOMER_API_CONNECT_TIMEOUT=banana
```

лучше startup binding failure, чем ошибка при первом production request.

---

# 9. ConfigMap: non-secret runtime configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config

data:
  CUSTOMER_API_URL: http://customer-api:8080
  CUSTOMER_API_CONNECT_TIMEOUT: 1s
  CUSTOMER_API_READ_TIMEOUT: 3s
  DB_POOL_SIZE: "10"
  FEATURE_NEW_CHECKOUT: "false"
```

Хорошо хранить:

- service URLs;
- ports;
- timeouts;
- pool settings;
- log levels;
- non-sensitive feature flags;
- tuning.

Не хранить:

- DB password;
- OAuth client secret;
- private key;
- access token;
- API key.

---

# 10. ConfigMap -> Pod -> Spring

```yaml
envFrom:
  - configMapRef:
      name: orders-config
```

```text
ConfigMap
   |
   | envFrom
   v
container environment
   |
   v
Spring Environment
   |
   v
@ConfigurationProperties
```

Mapping example:

```text
ConfigMap.data.CUSTOMER_API_URL
          ↓
environment CUSTOMER_API_URL
          ↓
\${CUSTOMER_API_URL}
          ↓
app.customer-api.base-url
```

---

# 11. envFrom vs explicit env

Compact:

```yaml
envFrom:
  - configMapRef:
      name: orders-config
```

Explicit:

```yaml
env:
  - name: CUSTOMER_API_URL
    valueFrom:
      configMapKeyRef:
        name: orders-config
        key: CUSTOMER_API_URL
```

Explicit mapping:

- делает contract видимым;
- уменьшает accidental injection лишних keys;
- позволяет переименовать source key.

---

# 12. Secret: что это на самом деле

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

Главная мысль:

```text
kind: Secret
   ≠
"значение автоматически надёжно зашифровано"
```

Secret — API primitive для confidential data, access control и безопасной delivery, но не полный secret-management lifecycle.

---

# 13. data vs stringData

```yaml
data:
  password: U3VwZXJTZWNyZXQ=
```

Это Base64 representation.

Decoded:

```text
SuperSecret
```

Следовательно:

```text
Base64
!=
encryption
```

Можно использовать:

```yaml
stringData:
  password: SuperSecret
```

API server преобразует строку в `data` representation.

**Ни `data`, ни `stringData` не делают real secret безопасным для Git.**

---

# 14. Где хранится Secret после kubectl apply

```text
kubectl / CI
      |
      v
kube-apiserver
      |
      v
API storage
      |
      v
etcd
```

Ключевой факт: если administrator не настроил encryption provider, Kubernetes API data at rest не получают дополнительное Kubernetes encryption автоматически.

```text
Secret object
     |
     +--> confidential API semantics
     +--> RBAC/access control
     |
     X--> not automatically encrypted at rest
```

---

# 15. Encryption at rest

Administrator может настроить:

```text
Secret
   ↓
kube-apiserver
   ↓
encryption provider
   ↓
ciphertext
   ↓
etcd
```

Это защищает persistent API resource data в etcd.

Не путать:

```text
etcd encryption
!=
Pod delivery
!=
application credential reload
```

---

# 16. Local encryption key

В self-managed cluster encryption key может находиться в `EncryptionConfiguration` на control-plane host.

```text
control-plane
   |
   +--> kube-apiserver
   |
   +--> EncryptionConfiguration
           |
           +--> local key
```

Плюс:

```text
etcd-only compromise
   ↓
attacker sees ciphertext
```

Минус:

```text
control-plane host compromise
   ↓
attacker may access local encryption key
```

Поэтому local-key encryption защищает etcd лучше, чем отсутствие encryption, но не даёт сильной separation от compromised control-plane host.

---

# 17. KMS и envelope encryption

Более сильная модель:

```text
                     REMOTE KMS
                  +-------------+
                  | KEK         |
                  +------+------+
                         |
                         | protects / unwraps
                         v
Kubernetes API Server -- KMS plugin
          |
          | encrypts API data using DEK
          v
         etcd
```

Термины:

```text
DEK = Data Encryption Key
      encrypts actual API data

KEK = Key Encryption Key
      protects DEK
      managed by remote KMS
```

Это envelope encryption.

Для современных Kubernetes clusters KMS v2 — stable начиная с Kubernetes 1.29. KMS v1 deprecated с 1.28 и не должен быть новым выбором.

---

# 18. Application secret vs Kubernetes encryption key

Не смешивать:

```text
DB_PASSWORD
   |
   | application credential
   v
Kubernetes Secret
```

и:

```text
KMS KEK / local encryption key
   |
   | platform encryption material
   v
protects Secret representation in etcd
```

Типичное ownership:

| Объект | Ответственность |
|---|---|
| DB password contract | app/DB/platform shared |
| Kubernetes Secret | deployment/platform |
| encryption-at-rest policy | platform/security |
| KMS KEK | security/platform |

---

# 19. Как проверять encryption at rest

Нельзя сделать:

```bash
kubectl get secret orders-db -o yaml
```

и по результату определить raw etcd encryption.

API server возвращает authorized caller usable representation.

Нужно проверять:

- kube-apiserver encryption configuration;
- provider order;
- KMS/plugin configuration;
- managed Kubernetes security settings;
- migration/rewrite существующих resources после включения encryption.

Это platform/admin responsibility.

---

# 20. Откуда DB password приходит в Kubernetes

## Model A — protected CI/CD variable

```text
CI secret store
      |
      v
deployment pipeline
      |
      v
Kubernetes Secret
      |
      v
Pod
```

## Model B — external secret manager

```text
Vault / cloud secret manager
        |
        v
integration/controller/CSI
        |
        v
Pod / Kubernetes Secret
        |
        v
Spring Boot
```

## Model C — operator-generated credentials

```text
DB operator
   |
   | creates/manages credential
   v
Secret
   |
   v
Spring Boot
```

Главный принцип:

> Git хранит **reference/template**, а не production credential.

---

# 21. Secret -> environment variable

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: orders-db
        key: password
```

Spring:

```yaml
spring:
  datasource:
    password: \${DB_PASSWORD}
```

Runtime:

```text
Secret
  ↓
container starts
  ↓
DB_PASSWORD copied into process environment
  ↓
Spring reads it
```

Главный недостаток:

```text
Secret changes
   ≠
running process env changes
```

Для env-based delivery нужен rollout/restart или другой application-specific mechanism.

---

# 22. Secret -> mounted file

```yaml
volumeMounts:
  - name: db-secret
    mountPath: /etc/secrets
    readOnly: true

volumes:
  - name: db-secret
    secret:
      secretName: orders-db
```

Получаем:

```text
/etc/secrets/username
/etc/secrets/password
```

Projected Secret volume может обновиться после Secret update.

Но:

```text
file changed
   ≠
application client reloaded
```

---

# 23. Spring Boot configtree

Spring Boot поддерживает configuration trees.

Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-db-configtree
type: Opaque

stringData:
  spring.datasource.username: orders_app
  spring.datasource.password: replace-at-deploy-time
```

Mounted:

```text
/etc/secrets/
├── spring.datasource.username
└── spring.datasource.password
```

Spring:

```yaml
spring:
  config:
    import: configtree:/etc/secrets/
```

или:

```text
SPRING_CONFIG_IMPORT=configtree:/etc/secrets/
```

```text
filename
  ↓
Spring property name

file contents
  ↓
property value
```

Полный пример:

- [Showcase 15 — Secret configtree](../../showcases/15-secret-configtree/README.md)
- [Showcase 15 annotated.yaml](../../showcases/15-secret-configtree/annotated.yaml)

---

# 24. ConfigMap как application.yaml

ConfigMap:

```yaml
data:
  application.yaml: |
    app:
      customer-api:
        base-url: http://customer-api:8080
        connect-timeout: 1s
        read-timeout: 3s
```

Mount:

```text
/etc/spring/application.yaml
```

Spring:

```text
SPRING_CONFIG_ADDITIONAL_LOCATION=file:/etc/spring/
```

Подходит для:

- nested structures;
- lists;
- maps;
- больших configuration groups.

Пример:

- [Showcase 14 — mounted application.yaml](../../showcases/14-configmap-mounted-application-yaml/README.md)
- [Showcase 14 annotated.yaml](../../showcases/14-configmap-mounted-application-yaml/annotated.yaml)

---

# 25. Env vs file vs configtree

| Подход | Хорошо для | Что происходит при update |
|---|---|---|
| env/envFrom | scalar settings | existing process env не меняется |
| mounted ConfigMap | structured file | file может обновиться; app reload отдельно |
| mounted Secret | cert/key/password files | file может обновиться; client reload отдельно |
| configtree | Spring-native file key/value | file update и Spring/client reload — разные вещи |

Главное:

```text
Kubernetes delivery semantics
     ≠
Spring runtime reload semantics
```

---

# 26. ConfigMap update и versioned configuration

Env:

```text
ConfigMap v2
   ↓
existing Pod env still v1
```

Mounted file:

```text
ConfigMap v2
   ↓
file eventually v2
   ↓
Spring bean may still hold v1
```

Простой production pattern:

```text
orders-config-v17
      ↓
Deployment
      ↓ rollout

orders-config-v18
      ↓
new Deployment revision
```

С `immutable: true` это ещё явнее.

---

# 27. Immutable ConfigMap / Secret

```yaml
metadata:
  name: orders-config-v18

immutable: true
```

Преимущества:

- auditability;
- explicit revision;
- predictable rollout;
- accidental mutation запрещена;
- rollback проще.

Цена: изменение требует нового object name/version.

---

# 28. Secret rotation — workflow, а не kubectl edit

Правильная последовательность:

```text
1 issue new credential
      ↓
2 old + new temporarily valid where possible
      ↓
3 update Secret
      ↓
4 rollout/reload
      ↓
5 verify new connections
      ↓
6 revoke old
      ↓
7 audit
```

Плохая последовательность:

```text
revoke old first
      ↓
existing Pods still use old env
      ↓
new connections fail
```

---

# 29. DB password rotation пошагово

Исходное:

```text
Database accepts P1
Pods use P1
```

## Step 1
Создать P2 или equivalent replacement credential.

## Step 2
Сделать P2 валидным на стороне DB.

## Step 3
Update Kubernetes Secret.

## Step 4
Rollout:

```bash
kubectl rollout restart deploy/orders-api
kubectl rollout status deploy/orders-api
```

## Step 5
Проверить:

- new Pods Ready;
- новые DB connections successful;
- error rate не вырос;
- old Pods ушли.

## Step 6
Revoke P1.

---

# 30. Почему Hikari усложняет проверку rotation

```text
Hikari pool
  |
  +--> existing connection authenticated with P1
  +--> existing connection authenticated with P1
```

Existing connection может продолжать работать после password change.

Поэтому тест должен заставить приложение создать **новое connection**.

Иначе:

```text
existing connections work
   ↓
tester thinks P2 is valid
   ↓
later reconnect fails
```

---

# 31. Certificate/key rotation

```text
new certificate/key
   ↓
Secret update
   ↓
mounted file update
   ↓
server/client must reload TLS material
```

Последний шаг зависит от конкретной library/server.

Следовательно:

> Secret volume updated ≠ TLS endpoint already uses new certificate.

---

# 32. RBAC и косвенный доступ к Secret

Кто может читать Secret — очевидный risk.

Но есть ещё:

> Кто может создать Pod/Deployment в namespace, тот может попытаться смонтировать доступный Secret в workload.

Поэтому security model должна учитывать:

```text
direct Secret API access
+
workload creation permissions
```

---

# 33. Не давайте Spring Boot list/get Secrets без причины

Плохая схема:

```text
Spring Boot
   |
   | list secrets
   v
Kubernetes API
```

если приложению нужен только один password.

Лучше:

```text
platform injects exact credential
      ↓
Pod
      ↓
Spring Boot
```

Если Kubernetes API не нужен:

```yaml
automountServiceAccountToken: false
```

---

# 34. Configuration URL dependencies

Плохо:

```text
PAYMENT_URL=http://10.25.10.31:8080
```

Kubernetes:

```text
PAYMENT_URL=http://payment-api:8080
```

Cross-namespace:

```text
http://payment-api.payments:8080
```

Причина:

```text
Pod IP disposable
Service DNS stable
```

---

# 35. Timeout — часть configuration contract

Недостаточно:

```yaml
PAYMENT_API_URL: http://payment-api:8080
```

Нужно:

```yaml
PAYMENT_API_URL: http://payment-api:8080
PAYMENT_CONNECT_TIMEOUT: 1s
PAYMENT_READ_TIMEOUT: 3s
```

```text
dependency hangs
   ↓
threads wait
   ↓
concurrency accumulates
   ↓
orders-api itself degrades
```

---

# 36. Pool size — тоже architecture configuration

```text
DB_POOL_SIZE=10
HPA maxReplicas=20

20 × 10
≈ up to 200 DB client connections
```

Поэтому ConfigMap value `DB_POOL_SIZE` нельзя менять без capacity thinking.

---

# 37. Retry configuration может разрушить dependency

Плохое:

```text
retry=10
```

без semantics.

Нужно определить:

- retryable errors;
- attempts;
- backoff;
- jitter;
- idempotency;
- overall deadline.

Пример:

```text
100 requests/sec
× 5 retries
= up to 500 attempts/sec
```

Это retry storm.

---

# 38. Git policy

## Можно

- property names;
- safe defaults;
- service names;
- ports;
- timeout defaults;
- pool defaults;
- non-secret ConfigMaps;
- Secret templates/references без real values.

## Нельзя

- production passwords;
- OAuth client secrets;
- private keys;
- API tokens;
- kubeconfig credentials;
- raw encryption/KMS keys;
- real confidential Secret manifests.

---

# 39. Для аналитика

Аналитик должен описывать configuration как contract.

| Parameter | Secret | Required | Default | Owner | Change/rotation |
|---|---:|---:|---|---|---|
| CUSTOMER_API_URL | no | yes | internal Service | app | rollout/config |
| CUSTOMER_API_CONNECT_TIMEOUT | no | yes | 1s | app | rollout/config |
| DB_USERNAME | sensitive | yes | none | DB/app | coordinated |
| DB_PASSWORD | yes | yes | none | DB/security | rotation runbook |
| DB_POOL_SIZE | no | yes | 10 | app/DB | capacity review |

Вопросы аналитика:

- что environment-specific?
- что sensitive?
- что mandatory?
- кто owner?
- какой safe default?
- нужен ли reload?
- нужен ли zero-downtime rotation?
- что происходит при missing/wrong value?

---

# 40. Для разработчика

Checklist:

- [ ] один image для environments;
- [ ] safe defaults;
- [ ] secrets без fallback;
- [ ] typed `@ConfigurationProperties`;
- [ ] validation;
- [ ] finite timeouts;
- [ ] pool sizing связан с replicas/HPA;
- [ ] secrets не логируются;
- [ ] понятны env/file/configtree semantics;
- [ ] rotation behavior протестирован;
- [ ] нет лишнего Kubernetes API access.

---

# 41. Для тестировщика

Нужно проверять не только happy path.

## Missing ConfigMap

Ожидание:

```text
required object missing
  ↓
new Pod cannot build expected container configuration
```

## Missing Secret key

```text
Secret exists
but key missing
  ↓
container setup or application startup failure
```

## Invalid typed value

```text
CONNECT_TIMEOUT=banana
  ↓
Spring binding/validation failure
```

## Wrong DB password

```text
DNS OK
network OK
TCP 5432 OK
DB authentication fails
```

## Secret changed without restart

Для env:

```text
Secret=P2
Pod env=P1
```

Это ожидаемо.

---

# 42. Failure lab — missing ConfigMap

```bash
kubectl delete configmap orders-config
kubectl rollout restart deployment/orders

kubectl get pod
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

Ответьте:

1. Pod создан?
2. Container started?
3. Какое event объясняет failure?
4. Почему repeated restart не исправит missing ConfigMap?

---

# 43. Failure lab — wrong DB Secret

```bash
kubectl create secret generic orders-db \
  --from-literal=DB_USERNAME=orders_app \
  --from-literal=DB_PASSWORD=wrong \
  --dry-run=client -o yaml | kubectl apply -f -
```

Для env-based delivery:

```bash
kubectl rollout restart deploy/orders
kubectl rollout status deploy/orders
kubectl logs deploy/orders
```

Expected:

```text
network path works
database auth fails
```

Logs не должны печатать password.

---

# 44. Failure lab — stale env after Secret update

```text
Secret v1 -> Pod starts -> env=P1

Secret changes -> P2

existing Pod env still P1
```

После rollout:

```text
new Pod env=P2
```

Это один из самых важных rotation lessons.

---

# 45. Failure lab — mounted Secret update

```text
Secret P1
   ↓
/etc/secrets/password=P1

Secret becomes P2
   ↓
projected file eventually P2
```

Следующий вопрос:

> Hikari уже использует P2?

Не обязательно.

Это проверяет разницу:

```text
delivery update
vs
application reload
```

---

# 46. Security anti-patterns

1. `password: \${DB_PASSWORD:password}`.
2. Real password в Git Secret manifest.
3. Считать Base64 encryption.
4. Давать application `list secrets` без необходимости.
5. Сначала revoke old credential, потом update application.
6. Класть private key/password в ConfigMap.
7. Считать mounted file update автоматическим Spring reload.
8. Хранить Pod IP как dependency URL.

---

# 47. Responsibility map

```text
Application developer
  -> property contract
  -> validation
  -> safe logging
  -> reload/client behavior

Platform
  -> Secret delivery
  -> RBAC
  -> encryption at rest
  -> KMS integration

Security
  -> key governance
  -> credential policy
  -> audit/rotation rules

DB/API owner
  -> credential issuance/revocation

Tester
  -> negative configuration
  -> rotation
  -> stale config
  -> leakage tests
```

---

# 48. Полная production схема Secret lifecycle

```text
             SECRET SOURCE
                  |
          +-------+-------+
          |               |
          v               v
        Vault        Cloud/CI secret
          |               |
          +-------+-------+
                  |
                  v
          Kubernetes delivery
                  |
                  v
        Kubernetes Secret object
                  |
          +-------+--------+
          |                |
          v                |
        etcd               |
          |                |
 encryption at rest?       |
      /         \          |
    no          yes        |
    |            |         |
 identity      KMS/local   |
                 |         |
                 v         |
             ciphertext    |
                           |
                  +--------+
                  |
                  v
                Pod
          +-------+--------+
          |       |        |
         env    file    configtree
          |       |        |
          +-------+--------+
                  |
                  v
           Spring Environment
                  |
                  v
       @ConfigurationProperties
                  |
                  v
      DB/client/TLS credential
                  |
                  v
              rotation
```

Эта схема заменяет неправильную mental model:

```text
Secret = зашифрованная переменная
```

---

# 49. Контрольные вопросы

## Основы

1. Почему один image используется между environments?
2. Что хранить в ConfigMap?
3. Что считать Secret?
4. Почему password не должен иметь unsafe default?
5. Что делает Spring Environment?
6. Зачем `@ConfigurationProperties`?

## Secrets

7. Чем `data` отличается от `stringData`?
8. Почему Base64 не encryption?
9. Где Kubernetes хранит API resource data?
10. Шифруются ли Secrets at rest автоматически?
11. Что такое encryption provider?
12. Где хранится local encryption key?
13. Почему local key не защищает полностью от host compromise?
14. Что такое KMS?
15. Чем DEK отличается от KEK?
16. Чем DB password отличается от KMS key?

## Delivery

17. Чем `envFrom` отличается от explicit `env`?
18. Обновится ли env running JVM после Secret update?
19. Может ли mounted Secret file обновиться?
20. Означает ли file update Spring/Hikari reload?
21. Что делает `configtree:`?

## Rotation

22. Почему нельзя сначала revoke old credential?
23. Зачем overlap?
24. Почему existing pool connections могут скрыть проблему нового password?
25. Что проверить после rollout?

---

# 50. Что изучать дальше

Следующая глава:

- [02 — Services, DNS and service security](02-services-dns-service-security.md)

Логика перехода:

```text
Глава 01:
мы научились доставлять URL/credentials/timeouts
в Spring Boot

Следующий вопрос:
что означает URL http://customer-api:8080?
кто создаёт этот адрес?
как Service выбирает Pods?
как работает DNS?
как поверх этого строится security?

        ↓

Глава 02
```

---

# 51. Связанные production-like примеры

## Env/config

- [Showcase 01 — Internal REST service](../../showcases/01-internal-rest-service/README.md)
- [Showcase 01 annotated.yaml](../../showcases/01-internal-rest-service/annotated.yaml)

## Mounted application.yaml

- [Showcase 14](../../showcases/14-configmap-mounted-application-yaml/README.md)
- [Showcase 14 annotated.yaml](../../showcases/14-configmap-mounted-application-yaml/annotated.yaml)

## Secret configtree

- [Showcase 15](../../showcases/15-secret-configtree/README.md)
- [Showcase 15 annotated.yaml](../../showcases/15-secret-configtree/annotated.yaml)

## PostgreSQL credential example

- [Showcase 12 — CloudNativePG](../../showcases/12-cloudnativepg-primary-replicas/README.md)

---

# 52. Sources

- https://docs.spring.io/spring-boot/reference/features/external-config.html
- https://docs.spring.io/spring-boot/api/java/org/springframework/boot/context/properties/ConfigurationProperties.html
- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/concepts/security/secrets-good-practices/
- https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/

Актуальные facts, использованные в главе:

- Secret `data` использует Base64 encoding, а не encryption.
- Kubernetes API data at rest не получают encryption provider автоматически, если administrator его не настроил.
- Kubernetes поддерживает at-rest encryption providers для API resources.
- KMS v2 stable с Kubernetes 1.29 и является актуальным вариантом для external key management.
- Spring Boot поддерживает external configuration и `configtree:` для mounted configuration trees.
