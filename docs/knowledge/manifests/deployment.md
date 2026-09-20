# Manifest Reference — Deployment

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Назначение

`Deployment` описывает desired state для stateless workload и управляет `ReplicaSet`, который уже создаёт Pods.

```text
Deployment
  -> ReplicaSet revision N
      -> Pods
```

Изменение `.spec.template` создаёт новую revision / ReplicaSet и запускает rollout.

---

## 2. Минимальный manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: app
          image: registry.example/orders:1.0.0
```

---

## 3. Production-oriented manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  labels:
    app.kubernetes.io/name: orders
spec:
  replicas: 3

  revisionHistoryLimit: 10
  minReadySeconds: 10
  progressDeadlineSeconds: 600

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  selector:
    matchLabels:
      app.kubernetes.io/name: orders

  template:
    metadata:
      labels:
        app.kubernetes.io/name: orders
    spec:
      serviceAccountName: orders
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 30

      securityContext:
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: app
          image: registry.example/orders:1.7.3
          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8080
              protocol: TCP

          envFrom:
            - configMapRef:
                name: orders-config
            - secretRef:
                name: orders-secret

          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              memory: 768Mi

          startupProbe:
            httpGet:
              path: /livez
              port: http
            periodSeconds: 5
            failureThreshold: 30

          livenessProbe:
            httpGet:
              path: /livez
              port: http
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            failureThreshold: 2

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "true"]

          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
```

---

## 4. apiVersion / kind

```yaml
apiVersion: apps/v1
kind: Deployment
```

`Deployment` находится в API group `apps`.

---

## 5. metadata

### metadata.name
Имя Deployment в namespace. Используется в командах:

```bash
kubectl rollout status deploy/orders
```

### metadata.namespace
Если не задан, используется namespace текущего context / CLI.

### metadata.labels
Labels самого Deployment. Они не обязаны совпадать с Pod labels, но обычно используется единая naming convention.

### metadata.annotations
Произвольная metadata для tooling/controllers. Semantics annotation зависит от конкретного controller/tool.

---

## 6. spec.replicas

```yaml
replicas: 3
```

**Default: 1.**

Desired number Pods.

Важно при HPA: если HPA управляет Deployment, постоянное declarative управление `replicas` другим инструментом может конфликтовать с autoscaler.

---

## 7. spec.selector

Обязателен:

```yaml
selector:
  matchLabels:
    app: orders
```

Selector должен совпадать с labels Pod template:

```yaml
template:
  metadata:
    labels:
      app: orders
```

Selector Deployment определяет, какие Pods/ReplicaSets относятся к workload.

Selector после создания практически считается immutable contract; изменение selector у существующего Deployment запрещено API validation в типичных случаях и должно проектироваться как новый workload identity.

---

## 8. spec.template

`.spec.template` — PodTemplateSpec.

Любое существенное изменение template, например image/env/probe/resources/labels, создаёт новую rollout revision.

Не создают новую Pod revision изменения полей Deployment вне template, например масштабирование `replicas`.

---

## 9. strategy.type

Возможные основные значения:

- `RollingUpdate`
- `Recreate`

**Default: RollingUpdate.**

### RollingUpdate
Старые и новые ReplicaSets сосуществуют во время rollout.

### Recreate
Старые Pods удаляются перед созданием новых; возможен downtime.

---

## 10. maxUnavailable

```yaml
maxUnavailable: 0
```

Допустим integer или percentage.

**Default для RollingUpdate: 25%.**

Определяет, сколько desired Pods может быть unavailable в rollout.

Не может быть 0 одновременно с `maxSurge=0`.

---

## 11. maxSurge

```yaml
maxSurge: 1
```

Integer или percentage.

**Default: 25%.**

Сколько Pods можно временно создать сверх desired replicas.

В реальном cluster terminating Pods могут некоторое время дополнительно потреблять resources, поэтому capacity planning не должен считать `replicas + maxSurge` абсолютно жёстким максимумом фактического consumption.

---

## 12. minReadySeconds

```yaml
minReadySeconds: 10
```

**Default: 0.**

Сколько секунд newly Ready Pod должен оставаться Ready без container crash, прежде чем считаться available.

Полезно для обнаружения Pods, которые становятся Ready и быстро падают.

---

## 13. progressDeadlineSeconds

```yaml
progressDeadlineSeconds: 600
```

**Default: 600 секунд.**

Если rollout не показывает progress в течение deadline, Deployment получает condition с reason `ProgressDeadlineExceeded`.

Controller не делает automatic rollback только из-за этого condition; CI/CD должен наблюдать status и принимать policy-defined action.

---

## 14. revisionHistoryLimit

```yaml
revisionHistoryLimit: 10
```

**Default: 10.**

Сколько старых ReplicaSets сохранять для history/rollback.

`0` удаляет old revisions после завершения rollout и фактически убирает возможность обычного Deployment rollback к старому ReplicaSet.

---

## 15. paused

```yaml
paused: true
```

Останавливает rollout processing изменений template до resume.

CLI:

```bash
kubectl rollout pause deploy/orders
kubectl rollout resume deploy/orders
```

Во время pause progress deadline не оценивается.

---

## 16. template.spec.serviceAccountName

Определяет ServiceAccount Pod.

Не путать:
- Kubernetes API identity;
- business authentication Spring Security.

Если приложению Kubernetes API не нужен, используйте отдельный SA либо default SA без привилегий и рассмотрите:

```yaml
automountServiceAccountToken: false
```

---

## 17. terminationGracePeriodSeconds

Pod-level field.

**Default: 30 секунд.**

При termination kubelet инициирует graceful shutdown и после grace period может принудительно завершить process.

Spring Boot timeout должен укладываться внутрь этого окна.

---

## 18. containers[].name

Уникально внутри Pod.

Используется:

```bash
kubectl logs <pod> -c app
```

---

## 19. containers[].image

OCI image reference.

Production preference:
- immutable version tag;
- либо digest.

Не полагаться на mutable `latest`.

---

## 20. imagePullPolicy

Основные значения:
- `Always`
- `IfNotPresent`
- `Never`

Default зависит от image reference/tag при первоначальном создании Pod; Kubernetes автоматически выбирает policy по правилам image name/tag. Не использовать это неявное поведение как release strategy.

---

## 21. command / args

Override image ENTRYPOINT/CMD semantics.

Spring Boot example:

```yaml
command: ["java"]
args: ["-jar", "/app/app.jar"]
```

Не переопределять без необходимости: container image должен сам иметь корректный startup contract.

---

## 22. ports[].containerPort

Documentational/network integration metadata о port container.

Само объявление `containerPort` не публикует Pod наружу.

Service использует selector + targetPort.

Named port:

```yaml
- name: http
  containerPort: 8080
```

упрощает Service/probes.

---

## 23. env / envFrom

```yaml
envFrom:
  - configMapRef:
      name: orders-config
  - secretRef:
      name: orders-secret
```

или явный key mapping.

Изменение ConfigMap/Secret не обновляет environment уже запущенной JVM.

---

## 24. resources

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    memory: 768Mi
```

Requests участвуют в scheduling и HPA calculations; limits контролируются cgroups.

Memory limit может привести к OOM kill; CPU limit — throttling.

---

## 25. probes

### startupProbe
Startup gate для медленного старта.

### livenessProbe
Решение о restart container.

### readinessProbe
Решение о допуске Pod к normal Service traffic.

Для Spring Boot обычно использовать Actuator probe groups.

---

## 26. lifecycle

Поддерживает lifecycle hooks, включая `preStop`.

Hook входит в termination lifecycle и потребляет grace period. Не использовать blind sleep без documented need.

---

## 27. container securityContext

Типичные hardening fields:

```yaml
runAsNonRoot: true
allowPrivilegeEscalation: false
capabilities:
  drop: ["ALL"]
readOnlyRootFilesystem: true
```

Последнее требует, чтобы приложение действительно не писало в root filesystem.

---

## 28. Pod securityContext

Например:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

Pod-level settings могут применяться ко всем containers в зависимости от поля.

---

## 29. volumes / volumeMounts

Deployment template может использовать:
- ConfigMap
- Secret
- PVC
- emptyDir
- projected
и другие volume sources.

Stateful data не следует хранить в container writable layer.

---

## 30. nodeSelector / affinity / tolerations / topologySpreadConstraints

Определяют placement Pods.

Используются для:
- dedicated nodes;
- availability zones;
- anti-affinity;
- spreading replicas.

Ошибочная жёсткая affinity может оставить Pods в `Pending`.

---

## 31. Update sequence

```text
spec.template changes
 -> Deployment controller creates new ReplicaSet
 -> new Pods created
 -> readiness determines availability
 -> old ReplicaSet scaled down
 -> rollout completes
```

---

## 32. Immutable / mutable summary

| Field | Typical behavior |
|---|---|
| metadata labels/annotations | mutable |
| replicas | mutable |
| template | mutable; triggers rollout |
| strategy | mutable |
| selector | effectively immutable after creation |
| image | mutable through template; rollout |
| resources | template change normally rolls Pods; modern in-place resize exists through dedicated resize mechanism |
| ServiceAccount | template change; rollout |

---

## 33. Failure scenario — selector mismatch on create

Manifest selector does not match template labels.

API validation rejects Deployment.

Check:

```bash
kubectl apply -f deployment.yaml
```

---

## 34. Failure scenario — image does not exist

Symptoms:
- new ReplicaSet exists;
- Pods `ImagePullBackOff`;
- rollout stalls.

```bash
kubectl get rs,pod
kubectl describe pod <pod>
kubectl rollout status deploy/orders
```

---

## 35. Failure scenario — readiness never succeeds

Symptoms:
- container Running;
- Pod NotReady;
- new ReplicaSet not available;
- rollout may reach progress deadline.

```bash
kubectl describe pod <pod>
kubectl get endpointslice
kubectl rollout status deploy/orders
```

---

## 36. Failure scenario — unschedulable requests

Pod remains Pending because resources/affinity cannot be satisfied.

```bash
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

---

## 37. Day-2 commands

```bash
kubectl get deploy orders
kubectl describe deploy orders
kubectl get rs -l app=orders
kubectl get pod -l app=orders

kubectl scale deploy/orders --replicas=5
kubectl set image deploy/orders app=registry/orders:1.7.4

kubectl rollout status deploy/orders
kubectl rollout history deploy/orders
kubectl rollout undo deploy/orders
kubectl rollout restart deploy/orders
```

---

## 38. Security checklist

- immutable/trusted image;
- non-root;
- least-privilege ServiceAccount;
- disable SA token mount if unused;
- Secret not embedded in manifest/image;
- resource limits/requests;
- seccomp RuntimeDefault where platform supports;
- drop capabilities unless required.

---

## 39. Spring Boot checklist

- `/livez`, `/readyz`;
- graceful shutdown;
- externalized config;
- DB/HTTP timeout;
- resource sizing;
- schema backward compatibility during mixed-version rollout;
- stdout/stderr logs.

---

## 40. CKAD commands

```bash
kubectl create deployment orders --image=nginx
kubectl scale deployment orders --replicas=3
kubectl set image deployment/orders nginx=nginx:1.27
kubectl rollout status deployment/orders
kubectl rollout undo deployment/orders
```

---

## Production-like examples

- [Deployment in internal REST service](../../../showcases/01-internal-rest-service/README.md)
- [Two Deployments in blue/green](../../../showcases/08-blue-green/README.md)
- [Stable + canary Deployments](../../../showcases/09-canary/README.md)

Каждый showcase содержит `WALKTHROUGH.md` с разбором mapping'ов и `all.yaml` с полной композицией.

## Sources

- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
