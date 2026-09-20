# Walkthrough — Spring Boot + S3-compatible object storage

## Архитектура

```text
client
  |
  v
Spring Boot file-api
  |
  | S3 API
  v
object storage
  |
  +--> uploads bucket
  +--> reports bucket

PostgreSQL
  |
  +--> object metadata
```

## Файлы

- [README](README.md)
- [Kubernetes manifests](all.yaml)
- [Spring storage config](application.yaml)

## Почему S3, а не PVC

Если приложение оперирует объектами:
- upload;
- attachment;
- report;
- export;

то object storage отделяет storage lifecycle от Pod/node.

```text
Pod replacement
   ↓
application starts elsewhere
   ↓
objects remain accessible through API
```

## Config vs Secret

ConfigMap:

```text
S3_ENDPOINT
S3_REGION
S3_BUCKET
```

Secret:

```text
S3_ACCESS_KEY
S3_SECRET_KEY
```

Не храните credentials в ConfigMap.

## Spring mapping

```yaml
app:
  storage:
    endpoint: ${S3_ENDPOINT}
    bucket: ${S3_BUCKET}
    access-key: ${S3_ACCESS_KEY}
```

Дальше лучше bind через `@ConfigurationProperties`.

## Object key design

Не используйте original filename как唯一 identity.

Production-like key:

```text
tenant/42/uploads/2026/09/uuid.pdf
```

Metadata отдельно:
- original filename;
- MIME;
- size;
- checksum;
- uploader;
- expiresAt;
- status.

## DB + S3 consistency

PostgreSQL transaction и S3 write не одна ACID transaction.

Надёжный flow:

```text
DB row PENDING
 -> upload object
 -> verify
 -> DB row READY
```

Failure cleanup:
- stale PENDING rows;
- orphan objects;
- retry-safe object keys.

## Presigned upload

Для large file:

```text
browser
 -> file-api requests authorization
 -> file-api returns presigned URL
 -> browser uploads directly to S3
 -> callback/confirm
 -> metadata READY
```

Так Spring Pod не прокачивает весь file body через heap/network.

## Failure scenarios

### Wrong endpoint
DNS/connect timeout.

### Wrong credentials
S3 auth failure.

### Upload succeeded, DB update failed
Orphan object.

### DB row created, upload failed
PENDING object record.

## Production

Добавьте:
- encryption;
- lifecycle/retention;
- bucket policy;
- malware scan if required;
- checksum;
- multipart upload;
- observability.

## Главное

Object storage решает storage API, но consistency между DB и object storage остаётся application responsibility.
