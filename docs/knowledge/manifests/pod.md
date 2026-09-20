# Manifest Reference — Pod

Проверено: 2026-09-20. Kubernetes baseline: 1.37.

Статус: **I2 FULL TECHNICAL REFERENCE**.

## 1. Назначение

Pod — минимальная schedulable workload unit Kubernetes. Containers внутри Pod разделяют Pod network namespace и могут совместно использовать volumes.

Обычно Spring Boot Deployment создаёт Pods через ReplicaSet; standalone Pod используется главным образом для labs/debug/special cases.

---

## 2. Минимальный manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
    - name: app
      image: nginx:1.27
```

---

## 3. Production-oriented PodTemplate fragment

```yaml
spec:
  serviceAccountName: orders
  automountServiceAccountToken: false
  restartPolicy: Always
  terminationGracePeriodSeconds: 30

  securityContext:
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: registry.example/orders:1.7.3

      ports:
        - name: http
          containerPort: 8080

      resources:
        requests:
          cpu: 250m
          memory: 512Mi
        limits:
          memory: 768Mi

      readinessProbe:
        httpGet:
          path: /readyz
          port: http

      securityContext:
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

---

## 4. apiVersion / kind

```yaml
apiVersion: v1
kind: Pod
```

Pod — core API object.

---

## 5. metadata

- `name`
- `namespace`
- `labels`
- `annotations`

Labels используются Service selectors, NetworkPolicy, observability selectors и workload controllers.

---

## 6. spec.containers

Обязательный список application containers.

Каждый container имеет собственные:
- image;
- command/args;
- env;
- resources;
- probes;
- securityContext;
- volumeMounts.

Containers Pod разделяют network namespace: `localhost` работает между containers одного Pod.

---

## 7. initContainers

Запускаются до regular app containers и должны завершиться успешно.

Use cases:
- подготовка files;
- schema/config generation;
- permissions preparation;
- ожидание strictly controlled prerequisite, хотя бесконечные wait loops являются anti-pattern.

Init container не должен заменять нормальную dependency retry/readiness architecture.

---

## 8. restartPolicy

Pod-level field.

Допустимые значения:
- `Always`
- `OnFailure`
- `Never`

**Default: Always.**

Deployment требует behavior, совместимый с постоянно работающим workload; Jobs обычно используют `Never` или `OnFailure`.

---

## 9. serviceAccountName

ServiceAccount identity Pod для Kubernetes API.

Если поле не задано, используется `default` ServiceAccount namespace.

---

## 10. automountServiceAccountToken

Контролирует автоматический mount credentials ServiceAccount.

Для Spring Boot backend без Kubernetes API:

```yaml
automountServiceAccountToken: false
```

уменьшает unnecessary credential exposure.

---

## 11. terminationGracePeriodSeconds

**Default: 30 seconds.**

При deletion:
- termination starts;
- hooks/signals process;
- grace period идёт;
- после deadline remaining processes могут быть force killed.

Spring graceful shutdown должен завершаться раньше hard deadline.

---

## 12. securityContext Pod

Pod-level security settings могут включать:
- runAsUser/runAsGroup;
- fsGroup;
- seccompProfile;
- supplementalGroups.

Container-level securityContext может уточнять часть параметров.

---

## 13. volumes

Pod-level volume declarations.

Часто:
- `configMap`
- `secret`
- `persistentVolumeClaim`
- `emptyDir`
- `projected`

Container использует volume через `volumeMounts`.

---

## 14. emptyDir

Создаётся для Pod и живёт до удаления Pod.

Container restart не удаляет `emptyDir`, но Pod replacement удаляет.

Не использовать как durable business storage.

---

## 15. nodeSelector

Простой equality-based placement по labels node.

Если подходящего node нет — Pod Pending.

---

## 16. affinity

Поддерживает:
- nodeAffinity;
- podAffinity;
- podAntiAffinity.

Required rules могут сделать scheduling невозможным; preferred rules — preference.

---

## 17. tolerations

Позволяют Pod быть scheduled/оставаться на nodes с matching taints.

Toleration не заставляет scheduler выбрать node, а только снимает соответствующий запрет.

---

## 18. topologySpreadConstraints

Управляет распределением Pods по topology domains: nodes/zones и т.п.

Полезно для HA replicas.

---

## 19. imagePullSecrets

References registry credentials для private image pull.

Это credentials kubelet/image pull path, а не environment Spring Boot.

---

## 20. dnsPolicy

Common values:
- `ClusterFirst`
- `Default`
- `ClusterFirstWithHostNet`
- `None`

Для обычного Pod default — **ClusterFirst**.

При `hostNetwork: true` часто требуется `ClusterFirstWithHostNet`, если нужен cluster DNS.

---

## 21. hostNetwork / hostPID / hostIPC

Подключают Pod к host namespaces.

Это powerful/privileged behavior и обычно не нужно Spring Boot workload.

Увеличивает security blast radius.

---

## 22. containers[].imagePullPolicy

- Always
- IfNotPresent
- Never

Автоматический default зависит от image tag/reference на момент создания object.

Production release должен явно понимать desired behavior.

---

## 23. command / args

Override image process.

Если image ENTRYPOINT корректный, лучше не дублировать startup logic в Deployment.

---

## 24. workingDir

Рабочая директория process внутри container.

Если не задана, зависит от image.

---

## 25. ports

`containerPort` не открывает firewall и не создаёт Service. Это declaration usable by named references/tooling.

---

## 26. env / envFrom

ConfigMap/Secret injection.

Environment values фиксируются при start container.

---

## 27. resources

Requests -> scheduling/resource accounting.

Limits -> runtime cgroup constraints.

Memory OOM и CPU throttling имеют разную semantics.

---

## 28. startup/liveness/readiness

- startup = initial startup gate;
- liveness = restart decision;
- readiness = traffic eligibility.

Readiness condition Pod отражается в Service EndpointSlice readiness.

---

## 29. lifecycle

Hooks:
- postStart
- preStop

Execution timing имеет race/timeout semantics; hook должен быть idempotent и bounded.

---

## 30. volumeMounts

Container mount path, readOnly, subPath и др.

Важно: ConfigMap/Secret mounted через `subPath` не получают автоматические updates обычным projected update mechanism.

---

## 31. Pod phase

Основные:
- Pending
- Running
- Succeeded
- Failed
- Unknown

`Running` не означает `Ready`.

---

## 32. Pod conditions

Важные conditions включают scheduling/readiness related state.

Для service traffic наиболее практична `Ready`.

---

## 33. containerStatuses

Содержит:
- ready;
- restartCount;
- state;
- lastState;
- image/imageID;
и другую runtime status информацию.

---

## 34. Container states

### Waiting
Например `ImagePullBackOff`, `CrashLoopBackOff` related waiting reason.

### Running
Process запущен.

### Terminated
Process завершился с exitCode/reason/signal.

---

## 35. Restart count

`RESTARTS` показывает restart containers в пределах lifecycle Pod.

При replacement создаётся новый Pod object с собственной history.

---

## 36. Pod immutability

Большая часть Pod spec после создания immutable или ограниченно mutable.

Поэтому controllers обычно не «редактируют Pod на месте», а создают replacement Pods.

Современный Kubernetes имеет dedicated in-place resize CPU/memory mechanism; это специальное исключение, не общая mutability Pod spec.

---

## 37. Failure — Pending

Причины:
- insufficient resources;
- PVC pending;
- impossible affinity;
- taints;
- node constraints.

```bash
kubectl describe pod <pod>
```

---

## 38. Failure — ImagePullBackOff

Причины:
- wrong image/tag;
- registry auth;
- registry/DNS/network.

```bash
kubectl describe pod <pod>
```

---

## 39. Failure — CrashLoopBackOff

Это backoff state, а не root cause.

```bash
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

---

## 40. Failure — Running NotReady

```bash
kubectl get pod
kubectl describe pod <pod>
kubectl get endpointslice
```

Причина часто readiness probe/application dependency.

---

## 41. Failure — OOMKilled

```bash
kubectl describe pod <pod>
kubectl logs <pod> --previous
```

Проверить container state/last state и memory sizing.

---

## 42. Termination runtime

При Pod deletion kubelet начинает graceful shutdown. Одновременно control plane обновляет EndpointSlices для terminating Pod; terminating endpoint не должен продолжать обслуживать обычный traffic.

Если `preStop` требует больше времени, grace period должен быть увеличен соответственно.

---

## 43. Day-2 commands

```bash
kubectl get pod -o wide
kubectl get pod <name> -o yaml
kubectl describe pod <name>
kubectl logs <name>
kubectl logs <name> --previous
kubectl exec -it <name> -- sh
kubectl delete pod <name>
```

---

## 44. Security checklist

- non-root;
- no host namespaces;
- no privileged unless justified;
- drop capabilities;
- seccomp;
- SA token off if unused;
- read-only filesystem if compatible;
- no secrets in command/args/logs.

---

## 45. CKAD

Must know:
- Pod YAML;
- command/args;
- env;
- volumes;
- probes;
- resources;
- securityContext;
- troubleshooting status/logs.

---

## Production-like examples

- [initContainer + main Pod composition](../../../showcases/16-initcontainer-main/README.md)
- [Native sidecar Pod](../../../showcases/17-native-sidecar/README.md)
- [Stateful Pod identity](../../../showcases/18-statefulset-headless-service/README.md)

Каждый showcase содержит `WALKTHROUGH.md` с разбором mapping'ов и `all.yaml` с полной композицией.

## Sources

- https://kubernetes.io/docs/concepts/workloads/pods/
- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/
- https://kubernetes.io/docs/concepts/storage/volumes/
