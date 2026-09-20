# Manifest Reference — ConfigMap

## Учебная схема объекта

### Роль ConfigMap

```text
ConfigMap
   |
   +--> env/envFrom
   |
   +--> mounted files
   |
   v
Spring Environment
   |
   v
@ConfigurationProperties
```

ConfigMap хранит **non-secret configuration**, а не application state.

### Annotated fragment

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config

data:
  # Значения здесь строки.
  CUSTOMER_API_URL: http://customer-api:8080
  DB_POOL_SIZE: "10"
```

Pod:

```yaml
envFrom:
  - configMapRef:
      name: orders-config   # exact object reference
```

> env-based configuration фиксируется при старте container. Изменение ConfigMap не меняет env работающей JVM.

Практика: [mounted application.yaml](../../../showcases/14-configmap-mounted-application-yaml/annotated.yaml).

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Назначение

`ConfigMap` хранит non-confidential configuration отдельно от container image.

Для Spring Boot это один из источников externalized runtime configuration.

---

## 2. Manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-config
data:
  SPRING_PROFILES_ACTIVE: k8s
  CUSTOMER_API_URL: http://customer-api:8080
  CUSTOMER_API_CONNECT_TIMEOUT: 2s
  feature.yaml: |
    feature:
      newCheckout: false
```

---

## 3. apiVersion / kind

```yaml
apiVersion: v1
kind: ConfigMap
```

Core API resource.

---

## 4. metadata

- name
- namespace
- labels
- annotations

ConfigMap names должны соответствовать Kubernetes object naming constraints.

---

## 5. data

Map `string -> string`.

Keys должны использовать допустимые символы: alphanumeric, `-`, `_`, `.`.

Values — UTF-8 strings.

Пример scalar:

```yaml
data:
  DB_POOL_SIZE: "10"
```

Число quoted, потому что ConfigMap data values — strings.

---

## 6. binaryData

Для binary/non-UTF8 content в base64-encoded representation.

Keys `data` и `binaryData` не должны пересекаться.

---

## 7. immutable

```yaml
immutable: true
```

Feature stable с Kubernetes 1.21.

Если true:
- `data` / `binaryData` больше нельзя изменить;
- metadata можно изменить;
- immutable нельзя вернуть обратно в mutable;
- для нового content обычно создаётся новый ConfigMap/name.

Плюс: kubelet не обязан держать watch на immutable object, что уменьшает control-plane load в больших clusters.

---

## 8. envFrom.configMapRef

```yaml
envFrom:
  - configMapRef:
      name: orders-config
```

Все допустимые keys становятся env vars container.

Риск: ConfigMap может со временем получить key, который приложение не ожидало.

---

## 9. env.valueFrom.configMapKeyRef

Явный contract:

```yaml
env:
  - name: CUSTOMER_API_URL
    valueFrom:
      configMapKeyRef:
        name: orders-config
        key: CUSTOMER_API_URL
```

Плюс: видны конкретные dependencies Pod.

---

## 10. Environment update behavior

ConfigMap value, переданное как env var, **не обновляет environment уже работающего container**.

```text
ConfigMap changed
 -> existing JVM env unchanged
 -> restart/rollout needed
```

---

## 11. Volume mount

```yaml
volumes:
  - name: config
    configMap:
      name: orders-config

containers:
  - volumeMounts:
      - name: config
        mountPath: /etc/orders
        readOnly: true
```

Каждый key может стать file.

---

## 12. Mounted update behavior

Для обычного ConfigMap volume kubelet eventually updates projected files после ConfigMap changes.

Update не мгновенный; behavior eventually consistent.

Application должна отдельно уметь detect/reload file.

Spring Boot сам по себе не гарантирует автоматический refresh ApplicationContext от изменившегося mounted file.

---

## 13. subPath limitation

Если ConfigMap file mounted через `subPath`, автоматические ConfigMap updates в container не отражаются обычным механизмом projected volume update.

---

## 14. optional

References могут быть optional.

```yaml
configMapRef:
  name: optional-config
  optional: true
```

или volume/configMap key refs.

Для обязательной application configuration обычно лучше fail-fast, а не silently continue.

---

## 15. Missing ConfigMap

Если Pod ссылается на required ConfigMap, которого нет, container startup блокируется/Pod events показывают configuration error.

```bash
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

---

## 16. Missing key

Required `configMapKeyRef` key отсутствует -> container configuration cannot be completed.

Optional key может быть пропущен.

---

## 17. Spring Boot relaxed binding

```yaml
data:
  APP_CUSTOMER_API_BASE_URL: http://customer-api:8080
```

может bind к property:

```text
app.customer-api.base-url
```

Через Spring relaxed binding.

---

## 18. @ConfigurationProperties

Предпочтительный typed contract:

```java
@ConfigurationProperties("app.customer-api")
public record CustomerApiProperties(
    URI baseUrl,
    Duration connectTimeout,
    Duration readTimeout) {}
```

ConfigMap transport не заменяет application validation.

---

## 19. ConfigMap size

ConfigMap предназначен для configuration, не для больших datasets/files.

Для больших assets используйте object storage/volume/image artifact в зависимости от semantics.

---

## 20. Security boundary

ConfigMap не для secret values.

Не хранить:
- DB password;
- API token;
- client secret;
- private key.

---

## 21. Versioned immutable pattern

```text
orders-config-v17
 -> Deployment references v17

orders-config-v18
 -> update Deployment reference
 -> rollout
```

Плюсы:
- auditability;
- rollback;
- immutable config.

Минус:
- cleanup old objects/tooling required.

---

## 22. Mutable fixed-name pattern

```text
orders-config
 -> update data
 -> rollout restart
```

Проще, но change history нужно обеспечивать GitOps/version control.

---

## 23. Failure scenario — wrong URL

ConfigMap syntactically valid, application starts, downstream calls fail.

Это semantic config failure, Kubernetes API не может его обнаружить.

Нужны:
- validation;
- logs;
- readiness/metrics as appropriate.

---

## 24. Failure scenario — changed config not observed

ConfigMap env value изменён, Pods не restarted.

Symptom: old behavior.

Check:

```bash
kubectl get configmap orders-config -o yaml
kubectl rollout history deploy/orders
kubectl get pod
```

---

## 25. Failure scenario — immutable ConfigMap update

Попытка изменить `data` immutable ConfigMap rejected.

Решение: новый ConfigMap object/name + workload update.

---

## 26. Day-2 commands

```bash
kubectl create configmap demo --from-literal=KEY=value
kubectl get configmap
kubectl describe configmap orders-config
kubectl get configmap orders-config -o yaml
kubectl rollout restart deploy/orders
```

---

## 27. Anti-patterns

- secrets in ConfigMap;
- giant application files;
- no typed validation;
- expect live env refresh;
- use subPath expecting update;
- mutable prod config outside audit/GitOps process.

---

## 28. CKAD

Must know:
- create ConfigMap;
- envFrom;
- configMapKeyRef;
- volume mount;
- optional references.

---

## Production-like examples

- [ConfigMap via envFrom](../../../showcases/01-internal-rest-service/README.md)
- [ConfigMap as mounted application.yaml](../../../showcases/14-configmap-mounted-application-yaml/README.md)

Каждый showcase содержит `WALKTHROUGH.md` с разбором mapping'ов и `all.yaml` с полной композицией.

## Sources

- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/config-map-v1/
- https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/
