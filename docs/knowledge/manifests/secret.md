# Manifest Reference — Secret

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Назначение

`Secret` хранит sensitive data и позволяет доставить её Pods без embedding credentials в image/application manifest.

Secret — storage/delivery primitive, не полный secret-management lifecycle.

---

## 2. Opaque Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-db
type: Opaque
stringData:
  username: orders
  password: replace-at-deploy-time
```

---

## 3. data vs stringData

### data
Values должны быть base64 encoded.

### stringData
Write-only convenience input для plain strings.

При API write `stringData` merge в `data`; если key присутствует в обоих, `stringData` имеет precedence.

`stringData` не возвращается API как отдельное поле.

Важно: `stringData` плохо сочетается с некоторыми Server-Side Apply workflows; учитывать выбранный GitOps/tooling model.

---

## 4. Base64 != encryption

```bash
echo cGFzc3dvcmQ= | base64 -d
```

Base64 только representation.

Security требует:
- RBAC;
- encryption at rest;
- API access protection;
- secret rotation;
- secure backups;
- safe logs.

---

## 5. Secret size

Individual Secret ограничен примерно **1 MiB**.

Secret не предназначен для больших binary artifacts.

---

## 6. type

Common types:
- `Opaque`
- `kubernetes.io/tls`
- `kubernetes.io/dockerconfigjson`
- bootstrap/token-specific built-in types.

Type позволяет API/tooling применять additional conventions/validation.

---

## 7. kubernetes.io/tls

Typical keys:
- `tls.crt`
- `tls.key`

Используется Ingress/Gateway/controller ecosystems для TLS certificates.

Private key остаётся sensitive secret material.

---

## 8. dockerconfigjson

Registry authentication credentials для image pull.

Используется через `imagePullSecrets`, а не как Spring application env.

---

## 9. immutable

```yaml
immutable: true
```

Stable с v1.21.

После установки true:
- secret data нельзя изменить;
- immutable нельзя вернуть в false;
- обычно создаётся new versioned Secret.

Плюс уменьшает watches kubelet на very large clusters.

---

## 10. envFrom.secretRef

```yaml
envFrom:
  - secretRef:
      name: orders-db
```

Все допустимые keys становятся env vars.

---

## 11. secretKeyRef

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: orders-db
        key: password
```

Более явный application contract.

---

## 12. Secret as volume

```yaml
volumes:
  - name: tls
    secret:
      secretName: orders-tls
```

Secret volumes read-only и backed by tmpfs according to Kubernetes volume semantics.

---

## 13. Mounted update behavior

Когда mutable Secret, mounted as normal volume, обновляется, kubelet eventually propagates update.

Это eventual consistency, не atomic application reconfiguration guarantee.

`subPath` mount не получает automatic Secret updates.

---

## 14. Env update behavior

Secret as env var фиксируется при container start.

Secret object update не меняет environment существующей JVM.

Нужен:
- restart;
- reload architecture;
- sidecar/operator mechanism;
в зависимости от delivery model.

---

## 15. Required Secret

По умолчанию Secret references required.

Если required Secret отсутствует, containers не запускаются до появления Secret.

Если Secret существует, но required key отсутствует, Pod startup также fails.

Можно указать `optional: true`, но для credentials это часто плохой default.

---

## 16. Secret exposure model

Любой actor с API permission читать Secret может получить secret material.

RBAC `get/list/watch secrets` должен быть крайне ограничен.

Особенно `list`/broad namespace access опасны.

---

## 17. Kubernetes API access не нужен приложению для injection

Pod может получать Secret через env/volume без того, чтобы Spring Boot имел RBAC `get secrets`.

Не выдавать application unnecessary Kubernetes API permissions.

---

## 18. etcd encryption

Platform должна настраивать encryption at rest для API data согласно security requirements.

Без этого Secret object не следует считать автоматически encrypted at rest только из-за kind Secret.

---

## 19. Rotation — overlap pattern

```text
old credential valid
 -> issue new credential
 -> update Secret / new version
 -> rollout/reload
 -> verify new connections
 -> revoke old credential
```

Лучший pattern, если dependency поддерживает overlap.

---

## 20. Rotation — no overlap

Нужен controlled cutover:
- maintenance window;
- dual account workaround;
- connection drain;
- application restart ordering.

Risk downtime выше.

---

## 21. Versioned Secret pattern

```text
orders-db-v17
orders-db-v18
```

Deployment reference change даёт явный rollout/audit trail.

Trade-off: lifecycle/cleanup old versions.

---

## 22. TLS rotation

Нужно обновлять coherent pair:
- private key;
- certificate;
- chain/trust where needed.

Application/controller должен reload new file/material.

---

## 23. Failure — missing Secret

```bash
kubectl delete secret orders-db
kubectl rollout restart deploy/orders
kubectl describe pod <new-pod>
```

Expected configuration/start failure events.

---

## 24. Failure — missing key

Secret существует, но `password` отсутствует при required secretKeyRef.

Pod container не сможет нормально стартовать.

---

## 25. Failure — wrong credential

Kubernetes считает Secret valid; Spring стартует и DB возвращает auth error.

Это semantic/application dependency failure.

---

## 26. Failure — changed Secret but old process

Secret updated, existing env-based JVM использует old value.

Runbook должен явно инициировать rollout/reload.

---

## 27. Secret in logs

Application exception/log config не должен печатать:
- raw password;
- bearer token;
- Authorization header;
- private key.

Это application responsibility.

---

## 28. Day-2 commands

Lab only:

```bash
kubectl create secret generic demo   --from-literal=username=demo   --from-literal=password=demo
```

Operational:

```bash
kubectl get secret
kubectl describe secret orders-db
kubectl describe pod <pod>
kubectl rollout restart deploy/orders
```

Не использовать `kubectl get secret -o yaml` без необходимости на shared screens/logged terminals.

---

## 29. Security checklist

- no real secrets in Git;
- encryption at rest configured;
- minimal RBAC;
- audit access;
- rotation documented;
- expiry monitoring for certs/tokens;
- backups protected;
- application logs sanitized;
- SA/API access minimized.

---

## 30. CKAD

Must know:
- create generic secret;
- env/secretKeyRef;
- envFrom;
- volume Secret;
- imagePullSecrets;
- TLS Secret basics.

---

## Sources

- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/secret-v1/
- https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
