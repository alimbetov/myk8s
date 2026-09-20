# 05 — Day-2 operations and failure playbook

Проверено: 2026-09-20.

## Investigation order

Не начинайте с restart. Сначала определите слой отказа.

```text
1 desired state
2 scheduling
3 image/container start
4 application startup
5 readiness/liveness
6 Service/EndpointSlice
7 DNS/network policy
8 external dependency
9 storage
10 rollout/migration compatibility
```

## Command set

```bash
kubectl get deploy,rs,pod -o wide
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
kubectl logs <pod> --all-containers
kubectl logs <pod> --previous
kubectl get svc,endpointslice
kubectl get networkpolicy
kubectl get pvc,pv,storageclass
kubectl top pod
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>
kubectl auth can-i --as=system:serviceaccount:<ns>:<sa> get secrets
```

## Failure matrix

| Failure | Symptom | First checks | Fix class |
|---|---|---|---|
| Pod crash | CrashLoopBackOff | logs --previous, describe | app/config/runtime |
| readiness | 0 ready endpoints | /readyz, EndpointSlice | dependency/startup/readiness |
| dependency down | timeouts/errors | DNS, TCP, client metrics | dependency/retry/circuit breaking |
| DNS | UnknownHost | nslookup/getent, CoreDNS | Service/DNS |
| wrong Secret | auth/startup error | secret ref, app logs | secret rotation |
| expired cert | TLS handshake | cert dates/chain/SNI | PKI rotation |
| network denied | connect timeout | NetworkPolicy/CNI | policy |
| node drain | eviction blocked | PDB, topology | availability |
| disk full | eviction/write failure | node/pod storage metrics | capacity/cleanup/expand |
| incompatible DB rollout | new version fails | migration history | expand/contract/rollback plan |

## Node drain

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

Перед drain проверяются:
- PDB;
- replica count;
- topology constraints;
- stateful operator status;
- local storage implications.

## Secret rotation

Rotation — не только заменить object:
1. create/issue new credential;
2. make dependency accept new credential;
3. update workload Secret/reference;
4. rollout/reload safely;
5. verify;
6. revoke old credential;
7. audit.

## Rollback

`kubectl rollout undo` применим к workload revision, но:
- не откатывает DB migrations;
- не возвращает удалённые external secrets;
- не восстанавливает data;
- не отменяет несовместимое message schema.

## Observability minimum

Метрики:
- request rate/errors/latency;
- JVM memory/GC/threads;
- readiness/liveness transitions;
- dependency pool saturation;
- queue lag;
- DB pool usage;
- restart/OOM/eviction;
- PVC capacity and node disk pressure.

Логи должны содержать correlation/trace identifiers, но не secrets/tokens.

## Sources

- https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
- https://kubernetes.io/docs/tasks/debug/debug-application/
- https://kubernetes.io/docs/concepts/storage/

Проверено: **2026-09-20**.
