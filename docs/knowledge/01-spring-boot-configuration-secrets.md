# 01 — Spring Boot configuration, ConfigMap and Secret

## Учебная карта темы

### Место конфигурации в системе

```text
Git / deployment values
        |
        +--> ConfigMap ------+
        |                   |
        +--> Secret ---------+--> Pod
                                 |
                                 v
                         Spring Environment
                                 |
                                 v
                      @ConfigurationProperties
```

Spring Boot image должен быть один и тот же между environments. Меняются **runtime values**, а не JAR/image.

### Три способа доставки

```text
ConfigMap/Secret
   |
   +--> env / envFrom
   |
   +--> mounted files
   |
   +--> configtree
```

### Annotated fragment

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: orders-db      # exact Secret object reference
        key: password        # key внутри Secret

# Spring:
# spring.datasource.password=${DB_PASSWORD}
```

> **Важно:** изменение Secret object не меняет environment уже работающей JVM. Для env-based delivery обычно нужен rollout/restart.

Полные варианты: [mounted application.yaml](../../showcases/14-configmap-mounted-application-yaml/README.md) и [Secret configtree](../../showcases/15-secret-configtree/README.md).

Проверено: 2026-09-20.

Цель главы — понять не только «как передать env», а **как построить конфигурационный контракт**, чтобы один image безопасно работал в dev/test/prod.

## 1. Как было раньше

На VM часто существовали:

```text
application.properties
application-test.properties
application-prod.properties
```

и в них же могли лежать реальные URL, usernames и passwords.

Проблемы:
- production credentials попадают в Git;
- для другого environment начинают собирать другой artifact;
- configuration и code lifecycle смешиваются;
- rotation credentials превращается в ручную операцию;
- platform team трудно безопасно управлять runtime config.

## 2. Современная модель

```text
application.yaml
  -> безопасные defaults и структура

ConfigMap
  -> environment-specific non-secret values

Secret / external secret provider
  -> credentials

Spring Environment
  -> объединяет property sources

@ConfigurationProperties
  -> typed contract внутри Java
```

Главный принцип: **build once, configure at runtime**.

---

## 3. Что оставлять в application.yaml

```yaml
app:
  customer-api:
    base-url: ${CUSTOMER_API_URL:http://customer-api:8080}
    connect-timeout: ${CUSTOMER_API_CONNECT_TIMEOUT:2s}
    read-timeout: ${CUSTOMER_API_READ_TIMEOUT:5s}

spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      connection-timeout: 3000
```

Здесь структура настроек хранится рядом с code, safe defaults — в Git, environment-specific values приходят снаружи, а secrets не имеют fallback defaults.

### Почему password без default

Плохо:

```yaml
password: ${DB_PASSWORD:admin123}
```

Если Secret забыли, приложение может тихо стартовать с небезопасным default.

Лучше:

```yaml
password: ${DB_PASSWORD}
```

и fail-fast при отсутствии значения.

---

## 4. Relaxed binding

Spring Boot умеет связать:

```text
app.customer-api.base-url
APP_CUSTOMER_API_BASE_URL
app.customerApi.baseUrl
```

с одной property. Это удобно в Kubernetes, где environment variables обычно uppercase с underscore.

---

## 5. Почему @ConfigurationProperties лучше большого набора @Value

Плохо масштабируется:

```java
@Value("${app.customer-api.base-url}")
private String baseUrl;

@Value("${app.customer-api.connect-timeout}")
private Duration connectTimeout;
```

Лучше:

```java
@ConfigurationProperties(prefix = "app.customer-api")
@Validated
public record CustomerApiProperties(
        @NotNull URI baseUrl,
        @NotNull Duration connectTimeout,
        @NotNull Duration readTimeout) {
}
```

Преимущества:
- типы;
- validation;
- одна группа настроек = один Java contract;
- metadata/tooling;
- проще unit tests;
- меньше scattered string keys.

**Практика:** обязательный URL лучше провалидировать при startup, чем получить NullPointerException при первом production request.

---

## 6. ConfigMap

ConfigMap предназначен для **несекретной** configuration.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
data:
  CUSTOMER_API_URL: http://customer-api:8080
  CUSTOMER_API_CONNECT_TIMEOUT: 2s
  CUSTOMER_API_READ_TIMEOUT: 5s
  DB_POOL_SIZE: "10"
```

Подключение:

```yaml
envFrom:
  - configMapRef:
      name: orders-config
```

Хорошо хранить:
- service URL;
- timeout;
- pool size;
- non-sensitive feature flags;
- tuning;
- application mode.

Не хранить passwords/tokens/private keys.

---

## 7. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-db
type: Opaque
stringData:
  DB_USERNAME: replace-at-deploy-time
  DB_PASSWORD: replace-at-deploy-time
```

Подключение:

```yaml
envFrom:
  - secretRef:
      name: orders-db
```

### Base64 не является encryption

``data`` использует base64 encoding. Это легко декодируется.

Security строится на:
- RBAC;
- encryption at rest;
- ограничении доступа;
- rotation;
- external secret manager, когда нужен полноценный lifecycle.

---

## 8. Env variable или mounted file

### Env

Плюсы:
- просто;
- естественно для Spring;
- удобно для scalar values.

Минус:
- изменение ConfigMap/Secret **не меняет environment уже работающего process**.

Нужен restart/rollout.

### Mounted file

Плюсы:
- удобно для certificates/keys/config files;
- projected content может обновляться.

Минус:
- приложение должно уметь reload;
- изменение файла не означает автоматический Spring context refresh.

**Практика:** passwords и simple settings часто удобны как env; certificates/private keys — как files.

---

## 9. Secret rotation — процесс, а не kubectl edit

Правильная последовательность:

```text
1 dependency принимает новый credential
2 Kubernetes Secret обновляется
3 workload reload/restart
4 connectivity проверяется
5 старый credential revoke
6 audit
```

Если сначала отключить старый credential, а Pods ещё используют старое env значение, возникнет outage.

---

## 10. URL зависимостей

VM-подход:

```text
http://10.10.20.45:8080
```

Kubernetes:

```text
http://customer-api:8080
```

Cross-namespace:

```text
http://customer-api.crm:8080
```

Pod IP хранить нельзя: Pod disposable.

---

## 11. Timeouts обязательны

Без timeout:

```text
dependency зависла
-> application thread ждёт
-> threads заканчиваются
-> request queue растёт
-> сервис сам становится unavailable
```

Нужно различать:
- connect timeout;
- response/read timeout;
- connection acquisition timeout;
- общий request deadline.

```yaml
app:
  customer-api:
    connect-timeout: 2s
    read-timeout: 5s
```

---

## 12. Connection pools

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      connection-timeout: 3s
```

Pool size нужно считать на **все replicas**:

```text
20 replicas × pool 20 = до 400 application DB connections
```

Плюс admin, migrations, monitoring.

Следовательно, scaling Deployment влияет на database capacity.

---

## 13. Retry: почему больше не всегда лучше

```text
100 requests/sec
dependency unavailable
5 retry на каждый request
=> до 500 attempts/sec
```

Получается retry storm.

Практика:
- finite retries;
- exponential backoff;
- jitter;
- retry только safe/idempotent operations;
- общий deadline;
- metrics.

Не нужно бесконтрольно делать retry одновременно в application client, service mesh и broker layer.

---

## 14. Git policy

### Можно

- property names;
- safe defaults;
- service names;
- ports;
- timeout defaults;
- pool defaults;
- ConfigMap manifests;
- Secret templates без values;
- resources.

### Нельзя

- production passwords;
- tokens;
- private keys;
- kubeconfig с credentials;
- OAuth client secret;
- реальный Secret manifest.

---

## 15. envFrom или явный env

Компактно:

```yaml
envFrom:
  - secretRef:
      name: orders-db
```

Явно:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: orders-db
        key: DB_PASSWORD
```

``envFrom`` проще, но explicit mapping делает contract заметнее и уменьшает accidental injection лишних keys.

---

## 16. Failure practice: отсутствует ConfigMap

```bash
kubectl delete configmap orders-config
kubectl rollout restart deployment/orders

kubectl get pod
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

Если reference mandatory, новый Pod не сможет нормально сформировать container configuration.

---

## 17. Failure practice: wrong Secret

```bash
kubectl create secret generic orders-db \
  --from-literal=DB_USERNAME=orders \
  --from-literal=DB_PASSWORD=wrong \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deploy/orders
kubectl logs deploy/orders
```

Ожидаем authentication failure.

**Security requirement:** log сообщает причину, но не печатает password.

---

## 18. Failure practice: Secret изменён, а Pod использует старое env

```text
Secret v1 -> Pod env=password-v1
Secret changed to v2
Pod process env всё ещё password-v1
```

Исправление:

```bash
kubectl rollout restart deploy/orders
kubectl rollout status deploy/orders
```

Именно поэтому rotation runbook обязан учитывать reload semantics.

---

## 19. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| property contract | ✓ |  |  |
| timeout semantics | ✓ |  |  |
| pool sizing | ✓ |  | ✓ |
| ConfigMap values |  |  | ✓ |
| secret storage |  | ✓ | ✓ |
| credential rotation |  |  | ✓ |
| etcd encryption |  | ✓ |  |
| safe logging | ✓ |  | ✓ |

---

## 20. Anti-patterns

- secrets в ``application-prod.yaml``;
- password default;
- Pod IP как URL;
- infinite timeout;
- retry без limit/backoff;
- десятки unrelated ``@Value``;
- считать base64 encryption;
- ожидать live env update после изменения Secret;
- выдавать приложению list/get Secrets в Kubernetes API, когда достаточно injection.

---

## 21. CKAD mapping

Нужно быстро уметь:

```bash
kubectl create configmap
kubectl create secret generic
kubectl set env
kubectl describe pod
kubectl get events
```

И понимать ``env``, ``envFrom``, ``configMapKeyRef``, ``secretKeyRef``, volume mounts.

Production дополнительно требует rotation, external secret management, validation, safe logging, timeout/pool/retry design.

---

## Sources: для проверки, а не вместо материала

- https://docs.spring.io/spring-boot/reference/features/external-config.html — property sources, relaxed binding, ``@ConfigurationProperties``.
- https://docs.spring.io/spring-boot/api/java/org/springframework/boot/context/properties/ConfigurationProperties.html — typed binding.
- https://kubernetes.io/docs/concepts/configuration/configmap/ — ConfigMap.
- https://kubernetes.io/docs/concepts/configuration/secret/ — Secret и security considerations.

Проверено: **2026-09-20**.
