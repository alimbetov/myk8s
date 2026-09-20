# 07 — Container image + JVM: что Kubernetes реально запускает

Проверено: 2026-09-20.

Эта глава отвечает на вопрос:

> Мы умеем создавать Pod и Deployment. **Что именно находится внутри image, как Java process становится container process, кто получает SIGTERM, почему tag и digest — не одно и то же, и как security/runtime decisions в image влияют на Spring Boot в Kubernetes?**

Главная mental model:

```text
source code
   ↓
Maven / Gradle
   ↓
Spring Boot JAR
   ↓
OCI image
   ↓
registry
   ↓
kubelet
   ↓
container runtime
   ↓
container process = PID 1
   ↓
JVM
   ↓
Spring Boot
```

Kubernetes не знает, что у вас “Java-приложение”. Для него есть **image + process + lifecycle contract**.

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь объяснить:

1. Чем JAR отличается от container image.
2. Что такое OCI image.
3. Что такое image layer.
4. Чем tag отличается от digest.
5. Почему mutable tag опасен как release identity.
6. Как работает `imagePullPolicy`.
7. Почему `imagePullPolicy` не меняется автоматически после смены tag.
8. Что делает kubelet при старте container.
9. Что такое ENTRYPOINT и CMD.
10. Как Kubernetes `command`/`args` связаны с ENTRYPOINT/CMD.
11. Почему exec-form ENTRYPOINT важен для signals.
12. Почему PID 1 критичен для graceful shutdown.
13. Что происходит при Pod termination.
14. Что такое non-root container.
15. Почему `runAsNonRoot` не исправляет плохой image.
16. Зачем `allowPrivilegeEscalation: false`.
17. Зачем drop Linux capabilities.
18. Зачем `readOnlyRootFilesystem`.
19. Где приложение может писать временные файлы.
20. Почему local container filesystem не durable.
21. Почему CA certificates/timezone/locale могут ломать Java runtime.
22. Как JVM видит cgroup memory/CPU.
23. Почему heap != RSS.
24. Зачем `MaxRAMPercentage`.
25. Почему thread stacks/direct memory важны.
26. Что дают Spring Boot layers.
27. Когда удобен Dockerfile, а когда Buildpacks.
28. Что такое reproducible image/supply chain traceability.
29. Что проверять QA/SDET.
30. Что должен знать аналитик о runtime artifact.

---

# 2. Одна схема runtime path

```text
Git commit
   |
   v
Maven/Gradle build
   |
   v
orders.jar
   |
   v
image build
   |
   v
registry.example/orders:1.7.3
   |
   | resolves to digest
   v
sha256:abcd...
   |
   v
kubelet asks container runtime to pull
   |
   v
image layers on node
   |
   v
container filesystem + process
   |
   v
PID 1 = java
   |
   v
SpringApplication.run(...)
```

Если вы понимаете эту цепочку, становится проще диагностировать:

```text
ImagePullBackOff
vs
container startup failure
vs
JVM startup failure
vs
Spring startup failure
```

---

# 3. Как было раньше на VM

Классическая схема:

```text
RHEL VM
 |
 +--> JDK installed by admin
 +--> /opt/orders/orders.jar
 +--> /etc/orders/application.properties
 +--> systemd
 +--> /var/log/orders
```

Deploy:

```bash
scp orders.jar host:/opt/orders/
ssh host
sudo systemctl restart orders
```

Между servers могли различаться:

- JDK patch;
- timezone;
- locale;
- CA trust store;
- OS libraries;
- users/groups;
- filesystem permissions.

Image уменьшает этот runtime drift.

---

# 4. Что такое container image

Container image — immutable package с filesystem layers и metadata.

Conceptually:

```text
image
├── base OS/runtime files
├── JRE
├── dependencies
├── application
└── metadata
    ├── entrypoint
    ├── cmd
    ├── env defaults
    └── user
```

Image **не является запущенным process**.

```text
image
  ↓ instantiate
container
  ↓
process
```

---

# 5. JAR vs image

JAR:

```text
Java application artifact
```

Image:

```text
runtime artifact
=
JRE
+ filesystem
+ application
+ OS/runtime dependencies
+ startup metadata
```

Kubernetes Deployment references image, не JAR.

---

# 6. Минимальный Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY target/orders.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Это рабочий baseline.

Но production questions остаются:

```text
Which exact base image?
Which user?
Which CA certificates?
Which timezone behavior?
Which JVM options?
Writable filesystem?
SBOM?
Patch process?
Signal handling?
```

---

# 7. Image layers

OCI/Docker image состоит из layers.

Conceptually:

```text
Layer 1: base filesystem
Layer 2: JRE/runtime
Layer 3: dependencies
Layer 4: application classes/resources
```

Node/cache может reuse unchanged layers.

Это важно для:

- build speed;
- push/pull speed;
- rollout time;
- registry storage;
- vulnerability patching.

---

# 8. Spring Boot layered images

Spring Boot layered archive обычно разделяет:

```text
dependencies
spring-boot-loader
snapshot-dependencies
application
```

Почему:

```text
dependencies change rarely
application changes often
```

Если изменился один Java class:

```text
application layer changes
dependencies layer reused
```

Spring Boot 4.1 продолжает официально поддерживать layered JAR/container image workflow. citeturn439639search0turn439639search2turn439639search9

---

# 9. Multi-stage Dockerfile

Учебный production-oriented pattern:

```dockerfile
FROM eclipse-temurin:21-jre AS builder

WORKDIR /builder

COPY target/orders.jar application.jar

RUN java -Djarmode=tools \
    -jar application.jar \
    extract \
    --layers \
    --destination extracted


FROM eclipse-temurin:21-jre

WORKDIR /application

COPY --from=builder /builder/extracted/dependencies/ ./
COPY --from=builder /builder/extracted/spring-boot-loader/ ./
COPY --from=builder /builder/extracted/snapshot-dependencies/ ./
COPY --from=builder /builder/extracted/application/ ./

ENTRYPOINT ["java", "-jar", "application.jar"]
```

Не копируйте exact base image blindly: organization должна иметь approved base-image policy.

---

# 10. Dockerfile vs Buildpacks

## Dockerfile

Плюсы:

- полный контроль;
- explicit packages;
- explicit users/filesystem;
- удобно для unusual native dependencies.

Минусы:

- вы отвечаете за base image lifecycle;
- больше boilerplate;
- легко сделать insecure defaults.

## Cloud Native Buildpacks

Spring Boot Maven/Gradle plugins умеют строить OCI image напрямую.

Например Maven:

```bash
mvn spring-boot:build-image
```

Spring Boot 4.1.1 Buildpacks создают OCI-compatible image и по security design build/run используют non-root users. citeturn439639search1turn439639search4

Buildpacks особенно хороши, если команда хочет standardized Java image lifecycle.

---

# 11. Image tag

Пример:

```text
orders:1.7.3
```

Tag — human-readable reference.

Но registry технически может переназначить tag на другой image.

```text
orders:1.7.3
   |
   +--> today digest A
   |
   +--> someone overwrites tag
   |
   +--> tomorrow digest B
```

Поэтому tag может быть mutable.

---

# 12. Image digest

Digest:

```text
registry.example/orders@sha256:abc123...
```

Digest uniquely identifies image content.

Mental model:

```text
tag
=
name / pointer

digest
=
content identity
```

Kubernetes docs прямо рекомендуют digest, когда нужна гарантия, что запускается одна и та же версия code независимо от изменения tag в registry. citeturn439639search3

---

# 13. Хороший release contract

Минимум:

```text
Git commit
   ↓
CI build id
   ↓
image tag
   ↓
image digest
```

Например:

```text
commit:
  8f31ab2

tag:
  orders:1.7.3

digest:
  sha256:...
```

На incident должно быть возможно ответить:

> Какой exact image сейчас работает?

---

# 14. Почему latest плох для production release identity

```yaml
image: registry/orders:latest
```

Проблемы:

- identity mutable;
- rollback unclear;
- different nodes can resolve at different times;
- audit weak;
- трудно связать runtime с CI artifact.

Лучше:

```text
semantic/build tag
+
record digest
```

или direct digest pinning.

---

# 15. imagePullPolicy

Возможные values:

```text
Always
IfNotPresent
Never
```

## IfNotPresent

Если image уже есть на node, kubelet может использовать local cached image.

## Always

При каждом container start kubelet просит runtime resolve image reference через registry; уже cached layers/content могут reuse, поэтому “Always” не означает обязательно download всех bytes заново.

## Never

Kubelet не pulls image; image должна уже присутствовать локально.

---

# 16. imagePullPolicy defaults

Если поле omitted при создании object:

```text
image uses digest
  -> IfNotPresent

tag = latest
  -> Always

no tag
  -> Always

explicit non-latest tag
  -> IfNotPresent
```

Это важно знать.

---

# 17. Тонкий нюанс: pull policy фиксируется при creation

Очень частая ошибка.

Создали:

```yaml
image: orders:1.7.3
# imagePullPolicy omitted
```

API установил:

```text
IfNotPresent
```

Позже поменяли image на:

```yaml
image: orders:latest
```

Kubernetes **не обязан автоматически поменять существующее поле** на `Always`.

`imagePullPolicy` устанавливается при первоначальном создании PodTemplate/object и затем не пересчитывается только из-за смены image reference. citeturn439639search3

Поэтому production manifest должен быть explicit.

---

# 18. Recommended release style

Например:

```yaml
containers:
  - name: app
    image: registry.example/orders:1.7.3
    imagePullPolicy: IfNotPresent
```

или digest:

```yaml
image: registry.example/orders@sha256:...
imagePullPolicy: IfNotPresent
```

Policy выбирается platform/release standard, а не случайно.

---

# 19. Что происходит при image pull

```text
Pod assigned to node
      ↓
kubelet
      ↓
container runtime
      ↓
registry auth/DNS/network
      ↓
resolve reference
      ↓
download missing layers
      ↓
unpack image
      ↓
create container
```

Failures здесь дают:

```text
ErrImagePull
ImagePullBackOff
```

Spring Boot ещё не запускался.

---

# 20. imagePullSecrets

Private registry может требовать credentials.

Pod/ServiceAccount:

```yaml
imagePullSecrets:
  - name: registry-credentials
```

Это credential для **registry pull**, а не application DB/API credential.

Не путать:

```text
imagePullSecret
!=
Spring application Secret
```

---

# 21. ENTRYPOINT и CMD

Container image имеет startup metadata.

Docker mental model:

```text
ENTRYPOINT
  = executable

CMD
  = default arguments
```

Например:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

# 22. Kubernetes command / args

Kubernetes naming отличается от Docker naming:

```text
Kubernetes command
  overrides image ENTRYPOINT

Kubernetes args
  overrides image CMD
```

Пример:

```yaml
command: ["java"]
args:
  - "-jar"
  - "app.jar"
```

Это очень важный CKAD/runtime mapping.

---

# 23. Почему лучше не override command без причины

Если image уже имеет правильный ENTRYPOINT:

```text
image owns startup contract
```

Deployment должен задавать runtime configuration, а не повторять startup command.

Плохая duplication:

```text
Dockerfile ENTRYPOINT
+
Deployment command
+
Helm command
```

Каждый слой может разойтись.

---

# 24. Exec form ENTRYPOINT

Предпочтительно:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

а не shell form:

```dockerfile
ENTRYPOINT java -jar app.jar
```

Почему:

```text
exec form
  -> java is container main process

shell form
  -> shell may become PID 1
  -> Java child process
```

Это влияет на signal forwarding и shutdown behavior.

---

# 25. PID 1

Внутри container есть process tree.

Хорошо:

```text
PID 1
  java
```

Плохо без корректного exec/signal handling:

```text
PID 1
  sh
   |
   +--> java
```

Kubernetes termination signal идёт main container process через runtime.

Если shell не forwarding signal правильно, Spring Boot может не получить SIGTERM вовремя.

---

# 26. Wrapper scripts

Иногда нужен script:

```sh
#!/bin/sh
set -e

# prepare...

exec java -jar /app/app.jar
```

Ключевое слово:

```text
exec
```

Оно заменяет shell Java process'ом.

Без `exec`:

```text
shell remains PID 1
Java remains child
```

---

# 27. Termination flow

Актуальная Kubernetes lifecycle model:

```text
Pod termination requested
       ↓
grace period starts
       ↓
preStop runs if configured
       ↓
runtime sends TERM to PID 1
       ↓
Spring Boot receives SIGTERM
       ↓
graceful shutdown
       ↓
process exits
       |
       +--> if grace expires: force termination
```

Kubernetes одновременно обновляет serving/terminating endpoint state during graceful shutdown. citeturn119455search7turn119455search10turn119455search11

---

# 28. preStop входит в тот же grace budget

Очень важный нюанс:

```text
terminationGracePeriodSeconds = 30s

preStop sleep = 20s
Spring graceful shutdown = 20s

20 + 20 > 30
```

Результат:

```text
Spring may be forcibly terminated
```

`preStop` не даёт “дополнительное время”; он расходует общий termination budget. citeturn119455search7turn119455search10

---

# 29. Container stop signal

Обычно runtime sends TERM.

Kubernetes также имеет container lifecycle stop-signal mechanisms/platform support, но для типичного Spring Boot contract следует проектировать корректную обработку SIGTERM.

Application должна:

- перестать брать новую работу;
- завершить in-flight;
- закрыть pools/listeners;
- выйти до grace deadline.

---

# 30. Spring Boot graceful shutdown

Runtime contract:

```text
SIGTERM
   ↓
JVM shutdown
   ↓
Spring context shutdown
   ↓
embedded web server graceful phase
   ↓
in-flight requests
   ↓
beans/pools/resources close
```

Но developer должен отдельно тестировать:

- Kafka;
- Rabbit listeners;
- custom executors;
- scheduled work;
- long DB transaction.

---

# 31. Non-root container

Production default:

```text
application should not require root
```

Kubernetes:

```yaml
securityContext:
  runAsNonRoot: true
```

Это не magically creates user.

Image должна быть совместима.

---

# 32. USER in Dockerfile

Например:

```dockerfile
RUN useradd --system --uid 10001 appuser

USER 10001
```

Но syntax/commands зависят от base distribution.

Buildpacks уже используют non-root runtime model по design. citeturn439639search4

---

# 33. runAsNonRoot: true не исправляет root-only image

Если image metadata/user приводит к root execution, а Pod требует:

```yaml
runAsNonRoot: true
```

container может не стартовать.

Это хороший failure:

```text
Spring Boot never starts
failure at runtime/security layer
```

Inspect:

```bash
kubectl describe pod <pod>
```

---

# 34. allowPrivilegeEscalation

Recommended:

```yaml
allowPrivilegeEscalation: false
```

Это запрещает process получить больше privileges, чем parent process, через mechanisms вроде setuid.

Kubernetes security guidance рекомендует disable privilege escalation для обычных workloads. citeturn119455search0turn119455search1

---

# 35. Linux capabilities

Root privileges разбиты на capabilities.

Typical Spring Boot REST API обычно не требует:

```text
CAP_SYS_ADMIN
CAP_NET_ADMIN
...
```

Production hardening:

```yaml
capabilities:
  drop:
    - ALL
```

Если действительно нужна capability, добавить минимально необходимую.

Restricted Pod Security Standard требует drop ALL, с ограниченным исключением для `NET_BIND_SERVICE`. citeturn119455search2

---

# 36. seccomp

Hardening:

```yaml
seccompProfile:
  type: RuntimeDefault
```

Это использует runtime's default syscall filtering profile.

Не обязательно понимать каждый syscall для начала, но важно знать:

```text
container security
!=
just non-root
```

---

# 37. readOnlyRootFilesystem

Hardening:

```yaml
readOnlyRootFilesystem: true
```

Это быстро выявляет скрытые filesystem assumptions.

Например Spring/app/library хочет писать:

```text
/tmp
/app
/home
```

и fails.

Нужно определить writable directories явно.

---

# 38. Writable temp directories

Если application нужен temp:

```yaml
volumes:
  - name: tmp
    emptyDir: {}

containers:
  - name: app
    volumeMounts:
      - name: tmp
        mountPath: /tmp
```

Mental model:

```text
root filesystem
  read-only

/tmp
  explicit ephemeral writable volume
```

Это более transparent contract.

---

# 39. emptyDir lifecycle

`emptyDir` живёт lifetime Pod.

```text
container restart
  -> emptyDir may remain

Pod removed
  -> emptyDir removed
```

Поэтому подходит для:

- temp files;
- shared scratch between containers;
- caches.

Не подходит для durable business state.

---

# 40. readOnlyRootFilesystem failure drill

Включите:

```yaml
readOnlyRootFilesystem: true
```

Если app fails:

```text
Permission denied
Read-only file system
```

не отключайте hardening сразу.

Сначала выясните:

> Почему application пишет в image layer?

Возможно нужен explicit `emptyDir`/PVC, либо library config.

---

# 41. CA certificates

Java HTTPS call:

```text
Spring Boot
   ↓
JVM TLS
   ↓
trust store
   ↓
server certificate chain
```

Если corporate internal CA отсутствует в runtime trust:

```text
PKIX path building failed
```

На VM admin мог когда-то вручную добавить CA.

В immutable image это должно быть reproducible.

---

# 42. Как управлять corporate CA

Options зависят от organization:

- approved base image уже содержит CA;
- image build добавляет CA deterministically;
- mounted trust material + JVM configuration;
- platform/service mesh handles TLS differently.

Главное:

```text
"на сервере сертификат был установлен"
```

не является reproducible image strategy.

---

# 43. Не путать TLS key/cert и CA trust

```text
CA trust
  -> кому JVM доверяет

client certificate/private key
  -> чем client authenticates itself

server certificate
  -> identity server side
```

Все три могут участвовать в mTLS, но это разные files/secrets.

---

# 44. Timezone

Container OS timezone может отличаться от business timezone.

Best practice в backend logic:

```text
store timestamps as UTC/Instant
convert at boundaries
```

Но libraries/reporting/log formatting могут зависеть от timezone.

Можно задавать JVM:

```text
-Duser.timezone=UTC
```

или application-level ZoneId explicit.

Не рассчитывайте:

> host настроен на Asia/Almaty, значит container тоже “как надо”.

---

# 45. Locale / encoding

Проблемы могут возникнуть с:

- default charset;
- locale-sensitive formatting;
- collation assumptions;
- fonts/report generation.

У Java runtime/environment они должны быть явными там, где business behavior зависит от них.

Особенно для PDF/Excel/report workloads.

---

# 46. Native libraries

Если Java dependency использует JNI/native code:

```text
JAR compiles
but
runtime image missing .so/library
```

Possible symptom:

```text
UnsatisfiedLinkError
```

Container image должен включать exact native dependencies.

---

# 47. Distroless/minimal images

Плюсы:

- меньше packages;
- меньше attack surface;
- меньше image size.

Минусы:

- нет shell;
- нет curl;
- debug harder.

Это не плохо.

Production pattern:

```text
minimal runtime image
+
kubectl debug / ephemeral debug tooling
```

а не “в каждый production image положим curl, ping, vim, tcpdump”.

---

# 48. Probes не должны требовать curl в image

Плохо:

```yaml
livenessProbe:
  exec:
    command:
      - curl
      - http://localhost:8080/livez
```

Лучше:

```yaml
livenessProbe:
  httpGet:
    path: /livez
    port: http
```

Kubelet выполняет HTTP probe сам.

Это позволяет runtime image оставаться минимальным.

---

# 49. Logs: stdout/stderr

Recommended container model:

```text
Spring Boot
   ↓
stdout/stderr
   ↓
container runtime
   ↓
node log files
   ↓
log collector
   ↓
central logging
```

Не проектируйте:

```text
/app/log/orders.log
```

как единственный production log source внутри ephemeral filesystem.

---

# 50. kubectl logs не архив

```bash
kubectl logs <pod>
```

операционный инструмент.

Для production retention нужны:

- centralized collection;
- storage/retention policy;
- searchable fields;
- access/security controls.

---

# 51. JVM container awareness

Современная JVM умеет учитывать container/cgroup constraints при ergonomics.

Но это не означает:

> memory planning больше не нужно.

JVM видит available/container memory и выбирает heap ergonomics, но process RSS включает гораздо больше.

---

# 52. JVM memory model

```text
container memory
   |
   +--> Java heap
   +--> metaspace
   +--> code cache
   +--> thread stacks
   +--> direct buffers
   +--> native libraries
   +--> GC/JVM native structures
```

Следовательно:

```text
-Xmx
!=
container memory usage
```

---

# 53. Почему Xmx = memory limit опасно

Пример:

```text
memory limit = 1Gi
-Xmx = 1Gi
```

Heap может теоретически занять весь budget.

Но process ещё требует native memory.

Результат:

```text
RSS > cgroup limit
   ↓
OOMKill
```

Поэтому нужен headroom.

---

# 54. MaxRAMPercentage

Java поддерживает `-XX:MaxRAMPercentage`.

Conceptually:

```text
heap max
≈
percentage of JVM-recognized max RAM
```

Например:

```text
-XX:MaxRAMPercentage=65
```

может оставить часть memory budget под native memory.

Exact percentage нужно измерять на workload, не копировать blindly. Oracle Java 21 docs продолжают документировать `MaxRAMPercentage` и связанные RAM-percentage options. citeturn439639search7

---

# 55. Fixed Xmx vs percentage

## Fixed

```text
-Xms512m
-Xmx512m
```

Плюсы:

- predictable.

Минусы:

- image/runtime config тесно связан с container limit;
- при разных limits нужен coordinated config.

## Percentage

```text
-XX:MaxRAMPercentage=...
```

Плюсы:

- адаптируется к container memory.

Минусы:

- всё равно требуется workload measurement.

Оба подхода могут быть production-valid.

---

# 56. Xms

Большой `-Xms`:

- резервирует/commits heap behavior earlier;
- может улучшить predictability;
- может увеличить baseline footprint.

Малый Xms:

- меньше startup memory;
- heap grows dynamically.

Выбор связан с workload/GC/latency requirements.

---

# 57. Threads и native memory

Каждый platform thread имеет stack.

Упрощённо:

```text
many threads
   ↓
more native stack memory
```

Sources:

- web server workers;
- custom executors;
- schedulers;
- Kafka listeners;
- Rabbit consumers;
- database/network libraries.

При OOM нужно смотреть не только heap histogram.

---

# 58. Virtual threads

Virtual threads снижают cost большого количества concurrent Java tasks по сравнению с platform threads.

Но:

```text
more virtual concurrency
!=
more PostgreSQL connections
!=
more downstream capacity
!=
unlimited CPU
```

Если 10,000 virtual tasks одновременно хотят DB connection:

```text
Hikari pool still finite
```

Concurrency model и dependency capacity остаются связанными.

---

# 59. Direct memory

Libraries типа Netty/NIO могут использовать off-heap/direct buffers.

Symptoms:

```text
heap looks fine
container memory high
```

Нужно смотреть:

- native/direct usage;
- process RSS;
- library behavior.

Это ещё одна причина, почему heap alone insufficient.

---

# 60. CPU container awareness

CPU requests/limits влияют на scheduling/runtime.

JVM может адаптировать:

- processor count assumptions;
- GC/compiler/thread ergonomics.

Но developer всё равно должен тестировать под **реальными cgroup limits**, а не только запускать JAR на laptop с 16 cores.

---

# 61. ActiveProcessorCount

Java имеет `-XX:ActiveProcessorCount` для override processor count, который JVM использует для thread-pool/GC ergonomics.

Это advanced tool.

Не задавайте его без измерений.

Обычно сначала правильно настройте Kubernetes CPU resources и наблюдайте actual behavior.

---

# 62. Startup time image + JVM

Rollout duration:

```text
image pull
+
container create
+
JVM startup
+
Spring startup
+
Liquibase
+
cache warmup
+
startupProbe
+
readiness
```

Large image и slow JVM startup влияют на:

- rollout;
- autoscaling responsiveness;
- recovery after node loss.

---

# 63. Image size — не единственный показатель

Маленький image полезен, но важнее:

```text
security
patchability
startup
layer reuse
reproducibility
runtime completeness
```

100 MB trustworthy/reproducible image лучше 40 MB image с undocumented runtime hacks.

---

# 64. Reproducible build

Хотим:

```text
same source + same build inputs
   ↓
predictable artifact
```

Нужно контролировать:

- dependency versions;
- build plugins;
- base image;
- timestamps/metadata where relevant;
- CI environment;
- generated resources.

Buildpacks тоже оптимизируют metadata для caching/reproducibility. citeturn439639search1

---

# 65. Supply chain minimum

Production pipeline должен уметь ответить:

```text
Where did image come from?
Which commit?
Which CI run?
Which dependencies?
Which base image?
Which vulnerabilities?
Who pushed it?
Which digest was deployed?
```

Minimum practices:

- trusted registry;
- vulnerability scan;
- SBOM;
- immutable references;
- signed/attested images where organization supports;
- base-image update process;
- provenance.

---

# 66. Base image patching

Даже если application code не менялся:

```text
base JRE/OS vulnerability found
   ↓
rebuild image
   ↓
new digest
   ↓
rollout
```

Production image lifecycle включает regular rebuild.

“Мы JAR не меняли” не значит “image не нужно обновлять”.

---

# 67. Secret leakage into image

Плохой Dockerfile:

```dockerfile
ENV DB_PASSWORD=prod-secret
```

Ещё хуже:

```dockerfile
COPY application-prod.yaml /app/
```

если там real secret.

Image layers могут храниться:

- registry;
- caches;
- CI artifacts;
- developer machines.

Secret injection должна происходить runtime.

---

# 68. Build ARG тоже не secret manager

Плохо:

```dockerfile
ARG DB_PASSWORD
RUN echo "$DB_PASSWORD" > /app/password
```

Build secrets могут оказаться в layers/history/cache depending on workflow.

Используйте dedicated build-secret mechanisms только для build-time credentials, а runtime app secrets передавайте runtime.

---

# 69. Image user and file ownership

Если image запускается UID 10001:

```text
/app files
/tmp mounts
cert files
PVC directories
```

должны быть readable/writable согласно contract.

Типичный failure:

```text
runAsNonRoot enabled
   ↓
application cannot write directory
   ↓
Permission denied
```

Это filesystem ownership issue, не Spring logic bug.

---

# 70. fsGroup и volume permissions

Для mounted volume может использоваться Pod security context:

```yaml
securityContext:
  fsGroup: 10001
```

Но semantics зависят от volume/driver.

Не используйте fsGroup как universal permission fix без понимания storage behavior.

---

# 71. Production-oriented Pod fragment

```yaml
spec:
  automountServiceAccountToken: false

  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: registry.example/orders@sha256:REPLACE_ME
      imagePullPolicy: IfNotPresent

      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL

      ports:
        - name: http
          containerPort: 8080

      volumeMounts:
        - name: tmp
          mountPath: /tmp

  volumes:
    - name: tmp
      emptyDir: {}
```

Это reference skeleton, не universal production manifest.

Kubernetes application security guidance рекомендует non-root, disabled privilege escalation, read-only root filesystem where possible, and minimal capabilities. citeturn119455search0turn119455search2

---

# 72. Что происходит при Deployment image update

```text
Deployment image changes
      ↓
Pod template hash changes
      ↓
new ReplicaSet
      ↓
new Pod scheduled
      ↓
image pull
      ↓
container created
      ↓
PID 1 starts
      ↓
JVM starts
      ↓
Spring starts
      ↓
startup/readiness
      ↓
traffic
```

Image behavior напрямую влияет на rollout behavior.

---

# 73. Failure lab — nonexistent image

Set:

```yaml
image: registry.example/orders:does-not-exist
```

Observe:

```bash
kubectl get pod
kubectl describe pod <pod>
```

Expected:

```text
ImagePullBackOff
```

Spring logs отсутствуют, потому что process не стартовал.

---

# 74. Failure lab — private registry auth

Remove/break imagePullSecret.

Expected:

```text
image pull authorization error
   ↓
ErrImagePull / ImagePullBackOff
```

Question:

> Это application Secret problem?

Нет. Это image distribution/supply-chain credential.

---

# 75. Failure lab — wrong command

Override:

```yaml
command: ["java"]
args:
  - "-jar"
  - "/app/missing.jar"
```

Expected:

```text
image pulled
container process starts
java exits with error
kubelet restarts
CrashLoopBackOff
```

Compare with ImagePullBackOff.

---

# 76. Failure lab — shell wrapper without exec

Create wrapper:

```sh
#!/bin/sh
java -jar /app/app.jar
```

Then terminate Pod and inspect shutdown logs/timing.

Compare with:

```sh
#!/bin/sh
exec java -jar /app/app.jar
```

Goal:

> understand PID 1 and signal propagation.

---

# 77. Failure lab — read-only root filesystem

Set:

```yaml
readOnlyRootFilesystem: true
```

If app tries to write:

```text
/app/tmp
```

observe failure.

Then fix explicitly:

```text
mount emptyDir at required temp path
```

Do not simply disable hardening without investigation.

---

# 78. Failure lab — missing corporate CA

Use HTTPS endpoint with corporate/private CA unavailable in image.

Expected:

```text
PKIX / trust failure
```

Diagnosis:

```text
DNS OK
TCP OK
TLS trust fails
```

Repair through approved trust-store/base-image mechanism.

---

# 79. Failure lab — memory headroom

Lab:

```text
memory limit = 512Mi
-Xmx = 512m
```

Apply load.

Observe:

- heap;
- RSS;
- OOMKilled;
- restartCount.

Then reduce heap budget or use measured percentage/headroom.

Goal:

```text
heap != process memory
```

---

# 80. Failure lab — local file disappears

1. Write file into container writable layer.
2. Delete Pod.
3. Wait replacement.
4. Check file.

Expected:

```text
file absent
```

Repeat with explicit PVC/object storage if persistence is required.

---

# 81. Failure lab — mutable tag

In lab registry, point same tag to new image without changing Deployment manifest.

Observe behavior across restarts/nodes depending on pull policy.

Goal:

```text
same YAML tag
can resolve to different content
```

Then compare digest pinning.

---

# 82. Analyst view

Analyst does not need Docker expertise, but should understand release artifact contract.

Useful fields:

| Requirement | Example |
|---|---|
| application version | 1.7.3 |
| image reference | registry/orders:1.7.3 |
| immutable digest | sha256:... |
| runtime Java | 21 |
| required CA | corporate-root-ca |
| timezone contract | UTC internally |
| filesystem persistence | none |
| startup SLO | < 60s |
| graceful shutdown | < 20s |
| security | non-root / no privilege escalation |

This helps separate:

```text
code version
runtime artifact
runtime config
platform behavior
```

---

# 83. Developer view

Developer checklist:

- [ ] image reproducible;
- [ ] JRE version explicit;
- [ ] ENTRYPOINT signal-safe;
- [ ] no secret baked into image;
- [ ] non-root compatible;
- [ ] read-only root compatible where feasible;
- [ ] writable temp paths explicit;
- [ ] CA requirements documented;
- [ ] timezone/locale assumptions explicit;
- [ ] native libraries explicit;
- [ ] heap/native memory budget measured;
- [ ] stdout/stderr logs;
- [ ] graceful SIGTERM tested;
- [ ] image traceable to commit/digest.

---

# 84. Tester view

QA/SDET should test:

- image pull failure;
- private registry auth;
- wrong command;
- non-root;
- read-only root;
- temp filesystem;
- SIGTERM;
- graceful shutdown;
- missing CA;
- wrong timezone if business-sensitive;
- memory limit/OOM;
- mutable tag behavior in lab;
- Pod replacement/local data loss.

Important:

> Tester should distinguish image/runtime failure from Spring/application failure.

---

# 85. Platform/SRE view

Platform owns/shared:

- registry;
- base-image policy;
- image scanning;
- admission controls;
- Pod Security;
- runtime/containerd;
- node image cache;
- registry credentials;
- CA distribution standards;
- SBOM/provenance policy;
- vulnerability remediation.

Developer owns application/runtime compatibility.

---

# 86. Failure classification

```text
Image cannot be downloaded
 -> supply/registry layer

Image downloaded, process cannot start
 -> runtime/security/command layer

Java starts, exits
 -> JVM/application startup layer

Spring starts, NotReady
 -> readiness/application dependency layer
```

Эта classification помогает не смешивать incidents.

---

# 87. Anti-patterns

1. `latest` как production release identity.
2. Mutable version tag overwrite.
3. Secrets in Dockerfile/image layer.
4. Shell-form ENTRYPOINT without understanding signals.
5. Wrapper script without `exec`.
6. Root container без необходимости.
7. `allowPrivilegeEscalation: true` by default.
8. Full Linux capabilities.
9. Writable root filesystem “потому что проще”.
10. Durable files in container writable layer.
11. Curl installed only for probes.
12. `Xmx == memory limit`.
13. Debug tools everywhere in runtime image.
14. Manual CA changes on running Pod.
15. Undocumented native OS packages.
16. No link image → commit → CI build → digest.
17. Never rebuilding image when base JRE has CVE.

---

# 88. CKAD mapping

Нужно быстро понимать:

- image;
- imagePullPolicy;
- command;
- args;
- container state;
- logs;
- securityContext;
- volumes;
- lifecycle;
- resource limits.

Useful:

```bash
kubectl get pod
kubectl describe pod
kubectl logs
kubectl logs --previous
kubectl exec
```

Но production слой добавляет:

```text
supply chain
SBOM
digest
base-image patching
PID 1
JVM ergonomics
CA
timezone
non-root/read-only
memory headroom
```

---

# 89. Связь с предыдущей главой

Глава 06:

```text
kubectl creates/manages container workloads quickly
```

Глава 07 раскрывает:

```text
what exactly runs inside that container
```

Теперь цепочка полная:

```text
Deployment
  ↓
Pod
  ↓
image
  ↓
container
  ↓
PID 1
  ↓
JVM
  ↓
Spring Boot
```

---

# 90. Что изучать дальше

Следующая глава:

- [08 — Resources, JVM memory and CPU](08-resources-jvm-memory-cpu.md)

Логика перехода:

```text
Глава 07:
мы поняли, как JVM оказывается внутри cgroup/container

Следующий вопрос:
сколько CPU/memory ей дать?
что такое request и limit?
почему Xmx нельзя приравнять к limit?
как GC, threads, direct memory,
OOMKilled и HPA связаны между собой?

       ↓

Глава 08
```

---

# 91. Production-like examples

- [Showcase 01 — Internal REST service](../../showcases/01-internal-rest-service/README.md)
- [Showcase 01 annotated.yaml](../../showcases/01-internal-rest-service/annotated.yaml)
- [Showcase 16 — initContainer + main](../../showcases/16-initcontainer-main/README.md)
- [Showcase 17 — native sidecar](../../showcases/17-native-sidecar/README.md)

---

# 92. Control questions

## Artifact

1. Чем JAR отличается от image?
2. Что такое image layer?
3. Чем tag отличается от digest?
4. Почему digest immutable identity?
5. Почему latest плох как release contract?

## Pull

6. Какие есть imagePullPolicy?
7. Какой default у non-latest tag?
8. Какой default у latest?
9. Меняется ли existing imagePullPolicy автоматически после смены tag?
10. Что означает ImagePullBackOff?

## Process

11. Чем ENTRYPOINT отличается от CMD?
12. Что Kubernetes command overrides?
13. Что Kubernetes args overrides?
14. Почему exec form важен?
15. Что такое PID 1?
16. Почему wrapper script должен использовать exec?
17. Как SIGTERM доходит до Spring?

## Security

18. Что делает runAsNonRoot?
19. Почему image всё равно должна поддерживать non-root?
20. Что делает allowPrivilegeEscalation?
21. Зачем drop ALL capabilities?
22. Что делает readOnlyRootFilesystem?
23. Зачем RuntimeDefault seccomp?

## Filesystem/runtime

24. Когда использовать emptyDir?
25. Почему local filesystem не durable?
26. Как read-only root выявляет скрытые assumptions?
27. Почему CA должна быть частью reproducible runtime contract?
28. Почему timezone нельзя оставлять случайной?

## JVM

29. Почему heap != RSS?
30. Почему Xmx=limit опасен?
31. Что делает MaxRAMPercentage?
32. Почему threads влияют на native memory?
33. Почему virtual threads не увеличивают DB capacity?
34. Что такое direct memory?

## Build/supply chain

35. Что дают Spring Boot layers?
36. Чем Dockerfile отличается от Buildpacks?
37. Почему base image нужно регулярно rebuild?
38. Что такое SBOM/provenance?
39. Почему image должен быть traceable до commit?

---

# 93. Sources

## Kubernetes

- https://kubernetes.io/docs/concepts/containers/images/
- https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/concepts/security/application-security-checklist/
- https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
- https://kubernetes.io/docs/concepts/security/pod-security-standards/

## Spring Boot

- https://docs.spring.io/spring-boot/reference/packaging/container-images/
- https://docs.spring.io/spring-boot/reference/packaging/container-images/dockerfiles.html
- https://docs.spring.io/spring-boot/reference/packaging/container-images/cloud-native-buildpacks.html
- https://docs.spring.io/spring-boot/reference/packaging/container-images/efficient-images.html

## JVM

- https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html

Актуальные facts, использованные в главе:

- Kubernetes image references могут использовать tags или immutable digests; digest pins exact image content.
- `imagePullPolicy` defaults зависят от initial image reference; значение поля не пересчитывается автоматически позже только из-за изменения tag.
- `Always` всё равно может использовать cached image layers/content after registry resolves digest.
- Kubernetes termination sends TERM to container process and `preStop` consumes тот же termination grace budget.
- Kubernetes security guidance рекомендует non-root, disable privilege escalation, read-only root filesystem where possible, seccomp RuntimeDefault and dropping unnecessary capabilities.
- Spring Boot 4.1.1 supports Dockerfiles, layered images and Cloud Native Buildpacks; buildpack images run non-root.
- Java 21 exposes RAM percentage options such as `MaxRAMPercentage` for heap ergonomics.
