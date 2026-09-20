# 17 — Secrets lifecycle and rotation

## Учебная карта темы

### Secret lifecycle шире Kubernetes Secret object

```text
credential created
   ↓
stored/delivered
   ↓
application consumes
   ↓
rotated
   ↓
new connection verified
   ↓
old credential revoked
   ↓
audit / expiry monitoring
```

Kubernetes Secret покрывает только часть:

```text
storage + API object + Pod delivery
```

### Rotation with overlap

```text
old works
   ↓
issue new
   ↓
update Secret
   ↓
rollout/reload
   ↓
verify
   ↓
revoke old
```

### Delivery variants

```text
env
mounted Secret file
Spring configtree
CSI/external secret integration
```

### Annotated fragment

```yaml
volumes:
  - name: db-secret
    secret:
      secretName: orders-db

# mounted file может обновиться,
# но Hikari/JVM не обязаны автоматически
# перечитать credential.
```

Практика: [Secret configtree](../../showcases/15-secret-configtree/README.md).

Проверено: 2026-09-20.

Secret — не YAML type, а lifecycle: creation, delivery, use, rotation, revocation, audit and recovery.

## 1. Что является secret

- DB password;
- OAuth client secret;
- API token;
- TLS private key;
- signing key;
- registry credential.

URL/timeout/port обычно не secret.

## 2. Kubernetes Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: orders-db
type: Opaque
stringData:
  username: orders
  password: example
```

Base64 encoding в `data` не encryption.

## 3. Threat model

Нужно защищать secret:
- в Git;
- в CI logs;
- в etcd;
- через Kubernetes API;
- внутри Pod;
- в application logs;
- при rotation;
- backup/restore.

## 4. Git

Никогда real secrets.

Допустимо:

```yaml
# secret.example.yaml
stringData:
  password: replace-at-deploy-time
```

Лучше pipeline/external manager создаёт value.

## 5. etcd encryption

Platform должна включить подходящую encryption-at-rest configuration и защитить etcd access/backups.

Secret object сам по себе не гарантирует encrypted storage.

## 6. RBAC

Минимизируйте access.

Application обычно не нужно `get secrets`: kubelet может inject value в Pod based on spec.

## 7. Env delivery

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: orders-db
        key: password
```

Изменение Secret не меняет env уже работающей JVM.

Rotation требует restart/reload strategy.

## 8. File delivery

Secrets можно mount as files.

Подходит для:
- TLS keys/certs;
- config files.

Application должна уметь reload, если нужен zero-restart rotation.

## 9. External secret managers

Production может использовать:
- cloud secret manager;
- Vault-like systems;
- CSI Secret Store;
- external synchronization operator.

Trade-off:
- stronger lifecycle/central governance;
- additional dependency/operator complexity.

## 10. Rotation pattern with overlap

Лучший сценарий dependency поддерживает два credentials:

```text
old valid
 -> create new valid
 -> deploy new
 -> verify
 -> revoke old
```

Так cutover безопаснее.

## 11. Rotation without overlap

Требует координированное maintenance/cutover и может иметь outage risk.

Это нужно документировать отдельно.

## 12. TLS certificate rotation

Нужно учитывать:
- cert;
- private key;
- CA chain;
- trust store;
- reload behavior;
- expiry monitoring.

## 13. Secret versioning

Полезно иметь distinct immutable Secret names:

```text
orders-db-v17
orders-db-v18
```

Deployment change становится явным rollout event.

Другой подход — fixed name + rollout restart. Trade-off зависит от tooling.

## 14. Failure practice: missing key

Deployment ожидает key `password`, Secret содержит `dbPassword`.

```bash
kubectl describe pod <pod>
kubectl get events
```

## 15. Failure practice: wrong value

Pod стартует, но DB auth fails.

Application log должен показывать auth failure, но не password.

## 16. Failure practice: expired TLS

Network/DNS работают, TLS handshake fails.

Проверяйте expiry/chain/SAN.

## 17. Emergency rotation

Runbook:
1. identify compromised credential;
2. issue replacement;
3. deploy/verify;
4. revoke;
5. inspect logs/audit;
6. rotate related credentials if blast radius uncertain.

## 18. Backup

Backup of Kubernetes/etcd/secret manager содержит secrets и требует соответствующей защиты.

## 19. Anti-patterns

- sealed-looking base64 considered secure;
- Secret in Git;
- credentials in command line/log;
- eternal password;
- broad list secrets RBAC;
- revoke old before new deployed;
- no expiry alerting.

## 20. Developer vs Platform/Security

| Область | Developer | Platform/Security | Shared |
|---|---:|---:|---:|
| no secret logging | ✓ | | |
| Secret delivery | | | ✓ |
| encryption at rest | | ✓ | |
| manager/CSI | | ✓ | |
| rotation semantics | | | ✓ |
| app reload | ✓ | | ✓ |

## 21. CKAD

Secret creation/consumption — CKAD. Encryption, external managers, rotation/audit — production.

## Связанные production-like примеры

- [Database credentials](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)
- [Secret mounted as Spring configtree](../../showcases/15-secret-configtree/README.md)

Для каждого стенда откройте `README.md` → `WALKTHROUGH.md` → `all.yaml`.

## Sources

- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

Проверено: **2026-09-20**.
