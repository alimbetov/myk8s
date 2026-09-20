# 07 — Container image + JVM in Kubernetes

Проверено: 2026-09-20.

Цель главы — понять, что именно Kubernetes запускает, как Spring Boot превращается в OCI image, почему Pod нельзя воспринимать как маленькую VM и какие решения в image напрямую влияют на graceful shutdown, security, startup и эксплуатацию.

## 1. Mental model

Kubernetes не знает Maven, Gradle и JAR. Для него runtime artifact — container image.

```text
source code
  -> Maven/Gradle
  -> Spring Boot JAR
  -> container image
  -> registry
  -> kubelet/container runtime
  -> JVM
  -> Spring Boot
```

JAR — Java artifact. Image — инфраструктурный runtime artifact.

## 2. Как было раньше

На VM типичный deploy мог выглядеть так:

```bash
scp target/orders.jar server:/opt/orders/
ssh server
sudo systemctl restart orders
```

JDK, OS libraries, timezone и users могли отличаться на серверах.

Container image фиксирует среду значительно жёстче:

```text
image =
  base filesystem
+ JRE
+ application
+ startup command
```

Это уменьшает drift между environments.

## 3. Минимальный Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/orders.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Работает, но это только стартовая точка.

## 4. Почему image должен быть immutable

Плохая модель:

```text
orders:latest
сегодня = commit A
завтра  = commit B
```

Тогда трудно ответить:
- какой код реально работает;
- что rollback-ить;
- какой artifact прошёл тесты.

Предпочтительно:

```yaml
image: registry.example/orders:1.7.3
```

или digest:

```yaml
image: registry.example/orders@sha256:...
```

## 5. Layered Spring Boot image

Spring Boot executable JAR содержит application и dependencies. Dependencies меняются реже, поэтому их выгодно отделять.

```text
dependencies
spring-boot-loader
snapshot-dependencies
application
```

Изменение одного Java класса тогда не обязательно инвалидирует слой всех библиотек.

Практический результат:
- меньше push/pull traffic;
- быстрее rollout;
- лучше cache reuse.

## 6. Dockerfile или Buildpacks

### Dockerfile

Плюсы:
- полный контроль;
- очевидный runtime;
- легко добавить OS packages.

Минусы:
- нужно сопровождать base image;
- больше responsibility.

### Cloud Native Buildpacks

Spring Boot plugin умеет создавать OCI images без ручного Dockerfile.

Плюсы:
- хорошие defaults;
- standardized build;
- меньше boilerplate.

Минусы:
- меньше low-level контроля;
- platform должна понимать lifecycle buildpacks.

Production-команда может использовать любой подход, если результат reproducible и patchable.

## 7. Non-root

Container не должен работать root без необходимости.

Kubernetes:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

Но manifest не исправит image, который технически требует root.

### Failure practice

Добавьте `runAsNonRoot: true` к несовместимому image:

```bash
kubectl get pod
kubectl describe pod <pod>
```

Важно определить: Spring ещё не стартовал; failure произошёл на container-runtime/security layer.

## 8. ENTRYPOINT и signals

Предпочтительно exec form:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Модель termination:

```text
kubelet
  -> SIGTERM
  -> PID 1 container
  -> JVM
  -> Spring graceful shutdown
```

Shell wrapper без корректного `exec`/signal forwarding может сломать эту цепочку.

## 9. Writable filesystem

Pod replacement создаёт новый writable layer.

Плохая практика:

```java
Files.write(Path.of("/app/invoices/report.pdf"), bytes);
```

если файл должен жить долго.

Используйте:
- object storage;
- PVC, когда нужен filesystem semantic;
- database для structured data.

## 10. Logs

Для Kubernetes удобная модель:

```text
Spring Boot
 -> stdout/stderr
 -> container runtime
 -> node log agent
 -> central logging
```

`kubectl logs` — оперативный инструмент, не архив.

## 11. JVM container awareness

Современная JVM учитывает cgroup CPU/memory constraints, но developer всё равно обязан понимать budget.

Memory container:

```text
heap
+ metaspace
+ code cache
+ thread stacks
+ direct buffers
+ native libraries
+ JVM native structures
```

Поэтому `-Xmx == memory limit` опасен.

## 12. Java threads

Рост thread count влияет не только на CPU. Thread stacks потребляют native memory.

При расследовании OOM учитывайте:
- Tomcat/Jetty threads;
- executors;
- Kafka concurrency;
- scheduler threads;
- virtual/platform thread model.

Virtual threads уменьшают стоимость concurrency, но не увеличивают PostgreSQL connections или downstream capacity автоматически.

## 13. Probes без curl

Не делайте image зависимым от shell utilities только ради health check:

```yaml
livenessProbe:
  exec:
    command: ["curl", "localhost:8080/livez"]
```

Предпочтительно:

```yaml
livenessProbe:
  httpGet:
    path: /livez
    port: http
```

Это позволяет использовать более минимальный runtime image.

## 14. Не запекайте secrets

Плохо:

```dockerfile
ENV DB_PASSWORD=my-prod-password
```

Image может быть:
- cached;
- downloaded;
- scanned;
- replicated.

Secret должен приходить runtime.

## 15. Supply chain minimum

Production pipeline желательно иметь:
- trusted base image;
- fixed JDK line;
- vulnerability scan;
- SBOM;
- immutable image reference;
- registry access control;
- понятный patch process;
- связь image ↔ commit ↔ CI build.

## 16. Что происходит при image update

```text
Deployment template changes
 -> new ReplicaSet
 -> scheduler creates new Pods
 -> node pulls image
 -> container starts
 -> JVM starts
 -> Spring starts
 -> startup/readiness succeed
 -> old Pod may be removed
```

Поэтому startup time image/JVM влияет на rollout duration.

## 17. Failure practice: wrong command

Сломайте command:

```yaml
command: ["java"]
args: ["-jar", "missing.jar"]
```

Проверить:

```bash
kubectl get pod
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

Учитесь отличать ImagePullBackOff от application process crash.

## 18. Failure practice: local data loss

1. Запишите файл внутрь Pod.
2. Удалите Pod.
3. Найдите replacement.
4. Проверьте файл.

Вывод: Deployment восстанавливает compute instance, не local data.

## 19. Developer vs Platform

| Область | Developer | Platform | Shared |
|---|---:|---:|---:|
| JAR/build | ✓ | | |
| image layout | ✓ | | ✓ |
| registry | | ✓ | |
| base-image policy | | | ✓ |
| vulnerability scan | | ✓ | ✓ |
| runtime security | | | ✓ |
| JVM behavior | ✓ | | ✓ |

## 20. CKAD

Нужно понимать:
- image;
- command/args;
- imagePullPolicy;
- securityContext;
- logs;
- container lifecycle.

Production дополнительно: supply chain, layered images, SBOM, patching, JVM runtime behavior.

## 21. Production checklist

- immutable image version;
- non-root;
- correct signal handling;
- no secrets in layers;
- no permanent data in writable layer;
- stdout/stderr logging;
- vulnerability scan;
- reproducible build;
- image traceable to source commit;
- base JRE has update policy.

## Sources

- https://docs.spring.io/spring-boot/reference/packaging/container-images/
- https://docs.spring.io/spring-boot/reference/packaging/container-images/dockerfiles.html
- https://docs.spring.io/spring-boot/reference/packaging/container-images/cloud-native-buildpacks.html

Проверено: **2026-09-20**.
