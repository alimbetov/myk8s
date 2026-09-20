# 06 — CKAD map: от production mental model к экзаменационной скорости

Проверено: 2026-09-20.

Эта глава отвечает на вопрос:

> Мы уже понимаем, **почему** Kubernetes-механизмы существуют и как они ломаются. Как теперь превратить это понимание в быстрые, уверенные действия на CKAD — не скатываясь в зубрёжку YAML?

Главная mental model:

```text
production understanding
        ↓
minimal Kubernetes primitive
        ↓
fast command / manifest
        ↓
verify expected state
        ↓
break intentionally
        ↓
diagnose
        ↓
repair
        ↓
repeat on time
```

CKAD — это не отдельный “мир”. Это **speed layer поверх правильной mental model**.

---

# 1. Что вы должны понимать после главы

После главы вы должны уметь:

1. Объяснить разницу между production competence и exam speed.
2. Читать задачу и сразу определять нужный Kubernetes primitive.
3. Быстро выбирать между imperative command и YAML edit.
4. Создать минимальный Pod, Deployment, Service, ConfigMap, Secret, Job, CronJob, PVC.
5. Добавить probes, resources, env, volumes.
6. Проверить состояние object без долгого поиска.
7. Понять по status/conditions, на каком слое failure.
8. Диагностировать Pending, CrashLoopBackOff, NotReady, Service/DNS problem.
9. Использовать `kubectl logs --previous` осознанно.
10. Быстро работать с namespaces и context.
11. Не путать exam-minimal manifest с production manifest.
12. Использовать официальный docs как reference, а не как замену пониманию.
13. Строить speed drills вокруг break/fix.
14. Понимать, какие темы CKAD проверяет глубже, а какие production темы остаются за рамками экзамена.

---

# 2. Актуальная модель CKAD

На дату проверки:

```text
Exam type:
  online
  proctored
  performance-based

Duration:
  2 hours

Kubernetes:
  v1.35
```

Linux Foundation указывает, что environment CKAD обновляется под актуальный minor Kubernetes примерно через несколько недель после release, поэтому baseline нужно перепроверять перед реальным экзаменом.

Экзамен проверяет не выбор ответа из списка, а действия в Kubernetes environment.

---

# 3. Production и CKAD: где граница

CKAD может закончить задачу на уровне:

```text
create object
configure required fields
verify expected behavior
```

Production добавляет:

```text
security
capacity
observability
SLO
rotation
backup
RPO/RTO
operator lifecycle
mixed-version compatibility
incident response
```

Пример.

CKAD:

```text
Create Deployment with readinessProbe.
```

Production:

```text
Why this readiness?
What dependency enters it?
What timeout?
How does rollout behave?
What does Service see?
What happens when DB is down?
```

Обе компетенции нужны, но это разные уровни глубины.

---

# 4. Как читать exam task

Каждую задачу разберите на четыре части:

```text
OBJECT
  Что создать/изменить?

STATE
  Какое конечное состояние требуется?

CONSTRAINTS
  namespace, names, ports, image, labels, paths...

VERIFY
  Как быстро доказать, что задача выполнена?
```

Пример:

> Create a Deployment named orders with 3 replicas, image nginx:1.27 and expose port 8080.

Разбор:

```text
OBJECT:
Deployment

STATE:
replicas=3
image=nginx:1.27
containerPort=8080

VERIFY:
kubectl get deploy
kubectl get pod
kubectl get deploy orders -o yaml
```

---

# 5. Сначала primitive, потом syntax

Не начинайте с вопроса:

> Как выглядел YAML?

Начинайте:

> Какой controller/object решает задачу?

Примеры:

| Requirement | Primitive |
|---|---|
| long-running replicated app | Deployment |
| one-shot task | Job |
| scheduled task | CronJob |
| stable network identity | Service |
| non-secret config | ConfigMap |
| credential | Secret |
| persistent filesystem request | PVC |
| restrict pod traffic | NetworkPolicy |
| API identity | ServiceAccount |
| autoscale replicas | HPA |

Если primitive выбран неправильно, идеальный YAML не поможет.

---

# 6. Imperative command или YAML?

## Imperative удобно для skeleton

Например:

```bash
kubectl create deployment orders \
  --image=nginx:1.27 \
  --replicas=3 \
  --dry-run=client -o yaml
```

Это быстро создаёт правильную структуру.

## YAML/edit нужен для сложных fields

Например:

- probes;
- multiple containers;
- volumes;
- NetworkPolicy;
- advanced securityContext;
- complex env mappings.

Хорошая exam strategy:

```text
generate skeleton
   ↓
edit only required fields
   ↓
apply
   ↓
verify
```

---

# 7. --dry-run=client -o yaml как speed tool

Пример:

```bash
kubectl create deployment orders \
  --image=nginx:1.27 \
  --dry-run=client -o yaml > /tmp/orders.yaml
```

Потом:

```bash
vi /tmp/orders.yaml
kubectl apply -f /tmp/orders.yaml
```

Плюсы:

- API structure не пишется с нуля;
- меньше syntax errors;
- быстрее переход к нужному field.

Но понимать generated YAML всё равно нужно.

---

# 8. Namespace discipline

Одна из самых дорогих экзаменационных ошибок:

```text
правильный object
в неправильном namespace
```

Перед задачей:

```bash
kubectl config set-context --current --namespace=<ns>
```

или явно:

```bash
kubectl -n <ns> ...
```

Verify:

```bash
kubectl config view --minify | grep namespace
```

Mental habit:

> namespace — часть identity object.

---

# 9. Pod speed drill

Минимум:

```bash
kubectl run web \
  --image=nginx:1.27 \
  --restart=Never
```

Verify:

```bash
kubectl get pod web
kubectl describe pod web
```

Break:

```text
wrong image
```

Expected:

```text
ImagePullBackOff
```

Diagnose:

```bash
kubectl describe pod web
```

Это уже mini incident drill.

---

# 10. Pod command/args drill

Нужно различать:

```text
container image ENTRYPOINT
container command
container args
```

Kubernetes:

```yaml
command: ["sh", "-c"]
args:
  - echo hello && sleep 3600
```

Exam mistake:

> перепутать command/args и получить immediate exit.

Verify:

```bash
kubectl logs <pod>
kubectl describe pod <pod>
```

---

# 11. Multi-container Pod mental model

```text
Pod
├── main container
├── sidecar/helper
└── shared network + optional volumes
```

Важно:

```text
containers in one Pod
share localhost/network namespace
```

Но у каждого:

- own image;
- own process;
- own resources;
- own logs.

Verify:

```bash
kubectl logs <pod> -c <container>
kubectl exec <pod> -c <container> -- ...
```

---

# 12. Init container drill

Mental flow:

```text
init container
   ↓ success
main containers start
```

Если init fails:

```text
main app never starts
```

Inspect:

```bash
kubectl get pod
kubectl describe pod
kubectl logs <pod> -c <init-container>
```

Production extension:

- finite wait;
- idempotent preparation;
- no endless “wait for DB” loops.

---

# 13. Deployment speed drill

Create:

```bash
kubectl create deployment orders \
  --image=nginx:1.27 \
  --replicas=3
```

Verify:

```bash
kubectl get deploy
kubectl get rs
kubectl get pod
```

Mental model:

```text
Deployment
  ↓
ReplicaSet
  ↓
Pods
```

---

# 14. Scale drill

```bash
kubectl scale deployment orders --replicas=5
```

Verify:

```bash
kubectl get deploy orders
kubectl get pod -l app=orders
```

Important:

```text
scale
!=
new rollout revision
```

---

# 15. Image update drill

```bash
kubectl set image deployment/orders \
  nginx=nginx:1.28
```

Здесь имя container должно совпасть с actual container name.

Verify:

```bash
kubectl rollout status deploy/orders
kubectl rollout history deploy/orders
kubectl get rs
```

Production extension:

- readiness;
- maxSurge;
- DB compatibility;
- graceful shutdown.

---

# 16. Rollback drill

```bash
kubectl rollout undo deployment/orders
```

Verify:

```bash
kubectl rollout status deployment/orders
kubectl rollout history deployment/orders
```

Exam meaning:

```text
previous Pod template revision
```

Production warning:

```text
does not rollback DB/events/external side effects
```

---

# 17. Probe drill

Typical fragment:

```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 2
  periodSeconds: 5
```

Exam task:

- add liveness/readiness/startup;
- path;
- port;
- thresholds/timings.

Mental check:

```text
startup -> startup gate
liveness -> restart?
readiness -> traffic?
```

Break:

```text
wrong readiness path
```

Observe:

```text
Running
Ready=False
```

---

# 18. Resources drill

Fragment:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

Remember:

```text
request
  -> scheduling / HPA denominator

limit
  -> runtime ceiling
```

Exam may ask exact values.

Production adds JVM sizing/capacity reasoning.

---

# 19. ConfigMap drill

Create literal:

```bash
kubectl create configmap orders-config \
  --from-literal=MODE=prod \
  --from-literal=TIMEOUT=3s
```

From file:

```bash
kubectl create configmap app-config \
  --from-file=application.yaml
```

Verify:

```bash
kubectl get configmap orders-config -o yaml
```

Use via env:

```yaml
envFrom:
  - configMapRef:
      name: orders-config
```

---

# 20. Secret drill

Create:

```bash
kubectl create secret generic db-secret \
  --from-literal=username=orders \
  --from-literal=password=test123
```

В exam lab это нормально как exercise.

Production:

> real credentials не должны попадать в shell history/Git без controlled secret workflow.

Use:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

---

# 21. ConfigMap/Secret failure drill

Break reference:

```text
name: missing-config
```

Observe:

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

Question:

> Pod created? Container started?

This teaches object dependency resolution.

---

# 22. Service speed drill

```bash
kubectl expose deployment orders \
  --name=orders \
  --port=80 \
  --target-port=8080
```

Verify:

```bash
kubectl get svc orders
kubectl get endpointslice \
  -l kubernetes.io/service-name=orders
```

Mental model:

```text
Service selector
  ↓
Pod labels
  ↓
EndpointSlice
```

---

# 23. Service selector break/fix

Break selector.

Observe:

```text
Service exists
DNS exists
EndpointSlice no usable backend
```

Commands:

```bash
kubectl get svc orders -o yaml
kubectl get pod --show-labels
kubectl get endpointslice \
  -l kubernetes.io/service-name=orders
```

Repair minimal selector mismatch.

---

# 24. DNS drill

Inside temporary/debug Pod:

```bash
nslookup orders
```

Cross namespace:

```bash
nslookup orders.sales
```

Mental model:

```text
short name
  -> namespace search path

service.namespace
  -> explicit namespace
```

Exam speed comes from knowing what name you expect before typing command.

---

# 25. NetworkPolicy drill

Think in three questions:

```text
TARGET:
which Pods are protected?

SOURCE:
who is allowed?

PORT:
where may traffic go?
```

Skeleton:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-orders
spec:
  podSelector:
    matchLabels:
      app: customer-api

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: orders-api

      ports:
        - protocol: TCP
          port: 8080
```

Production extension:

- namespace boundaries;
- default deny;
- DNS egress;
- CNI enforcement.

---

# 26. Job drill

```bash
kubectl create job hello \
  --image=busybox:1.36 \
  -- echo hello
```

Verify:

```bash
kubectl get job
kubectl get pod
kubectl logs job/hello
```

Mental model:

```text
Job
  ↓
Pod
  ↓
process exits 0
  ↓
Job Complete
```

---

# 27. Job failure drill

Command:

```text
exit 1
```

Observe:

- failed Pod;
- retry according to Job policy;
- Job conditions.

Production extension:

- idempotency;
- backoff;
- external side effects;
- Spring Batch restartability.

---

# 28. CronJob drill

```bash
kubectl create cronjob hello \
  --image=busybox:1.36 \
  --schedule="*/5 * * * *" \
  -- echo hello
```

Mental model:

```text
CronJob
  ↓ schedule
Job
  ↓
Pod
```

Verify:

```bash
kubectl get cronjob
kubectl get job
```

Production extension:

- timezone;
- concurrency policy;
- missed runs;
- idempotency.

---

# 29. PVC drill

Skeleton:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Verify:

```bash
kubectl get pvc
kubectl describe pvc data
```

Use in Pod:

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data
```

Then:

```yaml
volumeMounts:
  - name: data
    mountPath: /data
```

Remember mapping:

```text
volumeMount.name
=
volume.name

claimName
=
PVC.metadata.name
```

---

# 30. SecurityContext drill

Typical fields:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

Pod-level and container-level securityContext are related but not identical scopes.

Exam task may ask:

- runAsUser;
- runAsGroup;
- fsGroup;
- capabilities;
- readOnlyRootFilesystem.

Production adds Pod Security and image compatibility.

---

# 31. ServiceAccount / RBAC drill

ServiceAccount:

```bash
kubectl create serviceaccount orders
```

Check authorization:

```bash
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:<namespace>:orders
```

Mental model:

```text
Pod
  ↓ serviceAccountName
ServiceAccount
  ↓
RoleBinding
  ↓
Role
```

Do not confuse with business JWT identity.

---

# 32. HPA mental drill

Even if specific task varies, remember:

```text
metric
  ↓
HPA
  ↓
Deployment replica count
```

For CPU utilization:

```text
actual CPU
-----------
request CPU
=
utilization
```

Production extension:

```text
more Pods
 -> more pools
 -> more downstream load
```

---

# 33. Troubleshooting speed map

Use symptom to choose the shortest first command.

| Symptom | First command |
|---|---|
| Pending | `kubectl describe pod` |
| ImagePullBackOff | `kubectl describe pod` |
| CrashLoopBackOff | `kubectl logs --previous` |
| Running 0/1 | `kubectl describe pod` |
| Service broken | `kubectl get endpointslice` |
| DNS problem | `nslookup` inside Pod |
| rollout stalled | `kubectl describe deploy` |
| Job failed | `kubectl describe job` + Pod logs |
| PVC Pending | `kubectl describe pvc` |
| permission issue | `kubectl auth can-i` |

Не всегда это единственная команда, но это хороший first move.

---

# 34. get, describe, logs, exec: четыре разных вопроса

## get

```text
What exists?
What is current summary state?
```

## describe

```text
Why is Kubernetes reporting this state?
What events/conditions/configuration matter?
```

## logs

```text
What does application/container say?
```

## exec

```text
What does runtime environment look like from inside Pod?
```

Не используйте `exec` первым без причины.

---

# 35. Output formats

Полезно:

```bash
kubectl get pod -o wide
kubectl get pod -o yaml
kubectl get pod -o json
kubectl get pod -o name
```

Для script-like extraction:

```bash
kubectl get pod <name> \
  -o jsonpath='{.status.podIP}'
```

Главная цель — быстро достать required field, а не писать сложный one-liner ради красоты.

---

# 36. Editing existing objects

Иногда быстрее:

```bash
kubectl edit deployment orders
```

Иногда безопаснее:

```bash
kubectl get deployment orders -o yaml > /tmp/orders.yaml
# edit
kubectl apply -f /tmp/orders.yaml
```

На exam выбирайте путь с меньшим risk/time для конкретной задачи.

---

# 37. Verify immediately

После каждого significant action:

```text
create/change
   ↓
verify
```

Например:

```bash
kubectl apply -f /tmp/x.yaml
kubectl get ...
```

Не делайте 10 изменений подряд и только потом проверку.

Так проще локализовать свою ошибку.

---

# 38. YAML validation habits

Перед apply визуально проверьте:

- apiVersion;
- kind;
- metadata.name;
- namespace;
- indentation;
- selector/labels;
- container name;
- image;
- ports;
- references;
- field nesting.

Если API rejects manifest — сообщение validation часто быстрее перечитать, чем искать глазами всё подряд.

---

# 39. Names matter

Exam tasks часто чувствительны к exact names.

Проверьте:

```text
object name
container name
ConfigMap key
Secret key
port name
namespace
label key/value
volume name
claim name
```

Большая часть Kubernetes mappings — exact references или exact labels.

---

# 40. Selector thinking

Перед apply мысленно проговорите:

```text
Deployment selector
  =
Pod labels

Service selector
  =
Pod labels

NetworkPolicy podSelector
  =
target Pod labels
```

Это снижает количество самых неприятных “всё Running, но не работает” ошибок.

---

# 41. Reference thinking

Другие связи:

```text
secretKeyRef.name
  =
Secret.metadata.name

configMapRef.name
  =
ConfigMap.metadata.name

PVC claimName
  =
PVC.metadata.name

serviceAccountName
  =
ServiceAccount.metadata.name
```

Exam speed резко растёт, когда вы видите YAML как graph references, а не текст.

---

# 42. Named port thinking

```yaml
ports:
  - name: http
    containerPort: 8080
```

Service:

```yaml
targetPort: http
```

Probe:

```yaml
port: http
```

Один logical port name связывает несколько objects/fields.

---

# 43. Time management

Не тратьте одинаковое время на все задачи.

Mental strategy:

```text
read requirement
   ↓
estimate complexity
   ↓
solve clear tasks quickly
   ↓
verify
   ↓
mark harder task mentally
   ↓
return later
```

Не застревайте на одном obscure field, если остальные задачи доступны.

---

# 44. When to use docs

Docs полезны, когда:

- забыли exact field nesting;
- редкий option;
- NetworkPolicy syntax;
- probe field;
- Job/CronJob structure;
- securityContext detail.

Docs не должны быть нужны для вопроса:

```text
Which object solves this problem?
```

Mental model должна отвечать сразу.

---

# 45. Что нужно знать наизусть

Не полный YAML.

Нужно автоматически узнавать:

```text
Pod
Deployment
Service
ConfigMap
Secret
Job
CronJob
PVC
NetworkPolicy
ServiceAccount
resources
probes
volumes
env
selectors
```

И знать базовый `kubectl` workflow.

---

# 46. Что не нужно зубрить полностью

Неэффективно запоминать каждый uncommon field:

- редкие affinity combinations;
- сложные Job policies;
- все NetworkPolicy edge cases;
- каждый SecurityContext option.

Нужно помнить:

```text
что механизм решает
где примерно поле находится
как быстро проверить official reference
```

---

# 47. Speed drill format

Для каждой темы используйте одинаковую карточку.

## Example: Service

```text
CONCEPT
stable service identity

CREATE
kubectl expose deployment ...

VERIFY
kubectl get svc
kubectl get endpointslice

BREAK
wrong selector

SYMPTOM
no usable endpoints

FIX
restore selector

PRODUCTION NOTE
Service != authentication
```

Вот так и нужно готовиться.

---

# 48. Drill: Deployment

```text
CONCEPT
replicated long-running workload

CREATE
kubectl create deployment

VERIFY
get deploy,rs,pod

BREAK
bad image

SYMPTOM
ImagePullBackOff / stalled rollout

FIX
set correct image

PRODUCTION
readiness + capacity + schema compatibility
```

---

# 49. Drill: ConfigMap

```text
CONCEPT
non-secret runtime config

CREATE
kubectl create configmap

VERIFY
get -o yaml

BREAK
wrong reference

SYMPTOM
Pod config/startup failure

FIX
correct name/key

PRODUCTION
versioned config / rollout semantics
```

---

# 50. Drill: Secret

```text
CONCEPT
confidential runtime value

CREATE
kubectl create secret generic

VERIFY
object/reference

BREAK
missing key

SYMPTOM
container config/startup/auth failure

FIX
correct key/reference

PRODUCTION
Base64 != encryption
rotation + external secret lifecycle
```

---

# 51. Drill: readiness

```text
CONCEPT
traffic eligibility

CONFIGURE
readinessProbe

VERIFY
kubectl get pod
kubectl get endpointslice

BREAK
wrong path

SYMPTOM
Running but NotReady

FIX
correct readiness

PRODUCTION
dependency semantics matter
```

---

# 52. Drill: Job

```text
CONCEPT
workload must complete

CREATE
kubectl create job

VERIFY
get job/pod/logs

BREAK
exit 1

SYMPTOM
failed/retried Pod

FIX
command/application

PRODUCTION
idempotency + external side effects
```

---

# 53. Drill: NetworkPolicy

```text
CONCEPT
L3/L4 Pod traffic restriction

CREATE
manifest

VERIFY
controlled connectivity test

BREAK
deny required path

SYMPTOM
timeout

FIX
allow exact source/port

PRODUCTION
DNS + CNI + auth are separate
```

---

# 54. One scenario in two modes

## Production mode

Problem:

> orders-api must call payment-api securely.

You think:

```text
Service DNS
timeouts
NetworkPolicy
TLS/JWT
readiness
observability
failure semantics
```

## CKAD mode

Task:

> Create Service payment-api on port 8080 selecting app=payment.

You think:

```text
Service
selector
port
targetPort
verify endpoints
```

Production understanding narrows the exam problem instead of slowing you down.

---

# 55. Break/fix ladder

Train in this order:

```text
1 write healthy object
2 verify
3 break one thing
4 predict symptom
5 observe
6 repair
7 verify
8 recreate from scratch faster
```

Do not break five fields simultaneously.

---

# 56. 10-minute micro-drill

Example:

```text
Minute 0-2:
create Deployment

2-3:
add Service

3-5:
add ConfigMap env

5-6:
add readinessProbe

6-7:
verify

7-8:
break selector

8-9:
diagnose

9-10:
repair
```

This builds both syntax speed and causal thinking.

---

# 57. 30-minute integrated drill

Build:

```text
Deployment
+ ConfigMap
+ Secret
+ Service
+ readiness
+ resources
+ PVC or Job
```

Then inject two failures.

Goal:

```text
correctness first
then speed
```

---

# 58. Error budget for practice

During learning, record error type.

Example table:

| Error | Count |
|---|---:|
| wrong namespace | 3 |
| indentation | 1 |
| selector mismatch | 4 |
| wrong field path | 5 |
| forgot verify | 2 |
| slow docs search | 6 |

This shows where training time actually belongs.

---

# 59. Practice log

After session write:

```text
Task:
Create Service + readiness.

Time:
7m 20s

Error:
targetPort mismatch

Diagnosis:
2m

Lesson:
always compare targetPort with container named port
```

This is more useful than “studied Kubernetes 1 hour”.

---

# 60. Analyst view

Analyst does not need CKAD speed.

But this chapter helps read technical artifacts.

Analyst should recognize:

```text
Deployment
Service
ConfigMap
Secret
Job
NetworkPolicy
PVC
```

and ask:

- what requirement does this object implement?
- what is its failure effect?
- who owns it?
- what NFR depends on it?

---

# 61. Developer view

Developer should use CKAD preparation to gain operational fluency.

Goal:

```text
feature developer
   ↓
can inspect runtime
   ↓
can diagnose basic platform/application boundary
   ↓
can communicate with platform team precisely
```

Not:

```text
memorized kubectl aliases
only
```

---

# 62. Tester view

Tester benefits strongly from exam primitives.

If QA understands:

```text
selector
readiness
Secret
Service
NetworkPolicy
Job
PVC
```

they can design infrastructure-aware tests.

Example:

```text
break readiness
expect no Service traffic

break Secret
expect startup/auth failure

break selector
expect no endpoints
```

---

# 63. Production vs CKAD comparison

| Topic | CKAD focus | Production extension |
|---|---|---|
| Deployment | create/update | rollout SLO/schema compatibility |
| Probe | configure | correct health semantics |
| Service | expose/select | DNS/security/traffic architecture |
| ConfigMap | consume | config lifecycle/versioning |
| Secret | consume | rotation/KMS/external manager |
| PVC | mount | CSI/topology/backup |
| Job | completion | idempotency/restartability |
| NetworkPolicy | basic allow/deny | default deny/DNS/CNI |
| Resources | requests/limits | JVM/HPA/downstream capacity |
| Troubleshooting | get/describe/logs | observability/incident process |

---

# 64. Anti-patterns подготовки

1. Зубрить полный YAML.
2. Учить только imperative commands.
3. Учить только declarative YAML.
4. Не проверять namespace.
5. Не делать verify после apply.
6. Путать Running и Ready.
7. После каждой ошибки delete Pod.
8. Учить advanced Helm раньше primitives.
9. Использовать production chart как exam template.
10. Запоминать команду без понимания object relation.
11. Игнорировать labels/selectors.
12. Бояться official docs вместо тренировки быстрого поиска.

---

# 65. Personal study cycle

Рекомендуемый цикл на одну тему:

```text
Day N
  |
  +--> 15 min concept
  +--> 15 min minimal manifest
  +--> 15 min break/fix
  +--> 10 min speed repeat
  +--> 5 min notes
```

Для вашего k3s lab это особенно удобно: одна тема — один short practical session.

---

# 66. Mastery levels

## Level 1 — recognize

```text
понимаю, какой object нужен
```

## Level 2 — build

```text
могу создать без copy/paste
```

## Level 3 — verify

```text
могу доказать expected state
```

## Level 4 — break/fix

```text
могу предсказать symptom и исправить
```

## Level 5 — production reasoning

```text
могу объяснить trade-offs и failure consequences
```

CKAD требует сильного Level 2–4.
Работа senior engineer — Level 5.

---

# 67. Definition of mastery

Тема считается реально освоенной, когда вы можете:

```text
explain mechanism
+
identify correct primitive
+
create minimal version
+
verify
+
break
+
diagnose
+
repair
+
explain production caveat
```

Это и есть общий стандарт `myk8s`.

---

# 68. Integrated final drill

Сценарий:

> Создайте internal orders service.

Requirements:

```text
namespace: sales
Deployment: orders
replicas: 2
image: nginx:1.27
container port: 8080
ConfigMap: MODE=prod
Secret: TOKEN=demo
readiness: TCP/HTTP suitable check
Service: orders:8080
resource requests
```

После healthy state:

1. сломать image;
2. исправить;
3. сломать Service selector;
4. диагностировать EndpointSlice;
5. исправить;
6. сломать readiness;
7. исправить;
8. удалить Pod и проверить replacement.

Цель:

```text
build + diagnose
without losing mental model
```

---

# 69. Что изучать дальше

Следующая глава:

- [07 — Container image and JVM](07-container-image-jvm.md)

Логика перехода:

```text
Глава 06:
мы научились быстро работать с Kubernetes primitives

Следующий вопрос:
что именно находится внутри container image?
как Java process становится PID 1?
как image tag/digest, signals, non-root,
filesystem и JVM связаны с Kubernetes?

       ↓

Глава 07
```

---

# 70. Связанные материалы

## Manifest references

- [Pod](manifests/pod.md)
- [Deployment](manifests/deployment.md)
- [Service](manifests/service.md)
- [ConfigMap](manifests/configmap.md)
- [Secret](manifests/secret.md)
- [NetworkPolicy](manifests/networkpolicy.md)
- [PVC](manifests/pvc.md)
- [Job/CronJob](manifests/job-cronjob.md)
- [HPA](manifests/hpa.md)
- [Ingress](manifests/ingress.md)

## Showcases

- [Showcase catalog](../../showcases/README.md)
- [Internal REST](../../showcases/01-internal-rest-service/README.md)
- [REST + HPA + NetworkPolicy](../../showcases/03-rest-postgres-hpa-networkpolicy/README.md)

## Labs

- [Lab 01 — baseline](../../labs/01-spring-boot-baseline/README.md)
- [Lab 02 — JVM/resources/probes](../../labs/02-jvm-resources-probes/README.md)
- [Lab 03 — networking/security](../../labs/03-networking-security/README.md)
- [Lab 04 — rollout/HPA](../../labs/04-rollout-hpa/README.md)
- [Lab 05 — Jobs/operations](../../labs/05-jobs-operations/README.md)

---

# 71. Control questions

## Exam model

1. Почему CKAD speed не заменяет production understanding?
2. Какой текущий Kubernetes baseline экзамена?
3. Почему performance-based format меняет способ подготовки?
4. Что вы выделяете из task до команды?

## Workflow

5. Когда imperative command быстрее?
6. Зачем `--dry-run=client -o yaml`?
7. Почему namespace — часть identity?
8. Почему verify нужно делать сразу?

## Core primitives

9. Когда Pod?
10. Когда Deployment?
11. Когда Job?
12. Когда CronJob?
13. Что решает Service?
14. Что решает ConfigMap?
15. Что решает Secret?
16. Что решает PVC?
17. Что решает NetworkPolicy?

## Troubleshooting

18. First command при Pending?
19. First useful evidence при CrashLoopBackOff?
20. Что проверить, если Service не работает?
21. Что проверить при PVC Pending?
22. Что проверить при permission denied?
23. Чем get отличается от describe?
24. Когда нужен exec?

## Relationships

25. Что связывает Deployment selector и Pods?
26. Что связывает Service и Pods?
27. Что связывает targetPort и containerPort?
28. Что связывает volumeMount и volume?
29. Что связывает volume и PVC?
30. Что связывает Pod и ServiceAccount?

## Mastery

31. Почему break/fix лучше простого повторения healthy YAML?
32. Какие ошибки вы должны логировать в practice journal?
33. Что означает Level 5 mastery?
34. Почему production chart — плохая первая учебная форма?

---

# 72. Sources

- https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/
- https://kubernetes.io/docs/reference/kubectl/
- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands
- https://kubernetes.io/docs/tasks/debug/debug-application/
- https://kubernetes.io/docs/concepts/

Актуальная exam baseline на дату проверки:

- CKAD — online proctored performance-based exam;
- длительность — 2 часа;
- exam environment — Kubernetes v1.35;
- Linux Foundation указывает, что exam environment обновляется вслед за Kubernetes minor releases, поэтому версию следует перепроверить перед экзаменом.
