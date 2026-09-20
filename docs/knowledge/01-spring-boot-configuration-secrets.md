# 01 — Spring Boot configuration, ConfigMap and Secret

Проверено: 2026-09-20.

## 1. Mental model

Image должен быть одинаковым для dev/test/prod. Environment-specific config поступает через ConfigMap/Secret/external secret provider.

```text
Git -> non-secret defaults/manifests
ConfigMap -> non-confidential runtime config
Secret -> confidential runtime material
External secret manager -> source of truth for secret lifecycle
Spring Environment -> merges property sources
@ConfigurationProperties -> typed application contract
```

## 2. Spring Boot view

Предпочтительная модель:

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

Typed binding:

```java
@ConfigurationProperties(prefix = "app.customer-api")
public record CustomerApiProperties(
        URI baseUrl,
        Duration connectTimeout,
        Duration readTimeout) {}
```

Spring relaxed binding позволяет `APP_CUSTOMER_API_BASE_URL` связывать с `app.customer-api.base-url`.

## 3. Git policy

Можно хранить:
- service DNS names;
- ports;
- timeout defaults;
- feature flag defaults;
- resource requests/limits;
- ConfigMap templates.

Нельзя хранить:
- passwords;
- private keys;
- OAuth client secrets;
- production tokens;
- kubeconfigs с credentials;
- реальные Secret manifests с `stringData`.

Base64 в `Secret.data` — encoding, не encryption.

## 4. Kubernetes

ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
data:
  CUSTOMER_API_URL: http://customer-api:8080
  DB_POOL_SIZE: "10"
```

Secret placeholder:

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

Injection:

```yaml
envFrom:
  - configMapRef:
      name: orders-config
  - secretRef:
      name: orders-db
```

Environment variables не обновляются внутри уже запущенного process после изменения ConfigMap/Secret: нужен restart/rollout. Mounted files обновляются eventually-consistent, но приложение должно уметь reload.

## 5. Security

Kubernetes Secret по умолчанию нельзя считать достаточной защитой. Production:
- encryption at rest for etcd;
- least-privilege RBAC;
- external secret manager/CSI where appropriate;
- rotation process;
- no secret values in logs;
- namespace boundaries;
- disable unnecessary ServiceAccount token mounting.

## 6. Failure lab checklist

Wrong Secret:
```bash
kubectl create secret generic orders-db   --from-literal=DB_USERNAME=orders   --from-literal=DB_PASSWORD=wrong
kubectl rollout restart deploy/orders
kubectl logs deploy/orders
```

Проверить, что diagnostics показывают authentication failure, но не печатают password.

## Anti-patterns

- один огромный `application-prod.yaml` с паролями в Git;
- dependency URL как Pod IP;
- infinite HTTP/DB timeout;
- `@Value` для десятков связанных параметров вместо typed `@ConfigurationProperties`;
- выдавать приложению Kubernetes Secret read/list API permissions, если достаточно mounted value.

## Sources

- https://docs.spring.io/spring-boot/reference/features/external-config.html
- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/configuration/secret/

Проверено: **2026-09-20**.
