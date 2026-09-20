# Lab 03 — Service discovery, NetworkPolicy and RBAC

## Цель

Научиться разделять:
- DNS;
- Service/EndpointSlice;
- TCP reachability;
- NetworkPolicy;
- authentication;
- Kubernetes RBAC.

## 1. Namespace

```bash
kubectl create namespace net-lab
kubectl config set-context --current --namespace=net-lab
```

Разверните два workloads:
- caller;
- backend.

Создайте Service `backend`.

## 2. DNS

```bash
kubectl exec caller -- nslookup backend
```

## 3. HTTP

```bash
kubectl exec caller --   curl -v http://backend:8080/readyz
```

## 4. Failure — selector mismatch

Измените Service selector.

Проверить:

```bash
kubectl get svc backend
kubectl get pod --show-labels
kubectl get endpointslice   -l kubernetes.io/service-name=backend
```

DNS может работать, endpoint отсутствовать.

## 5. Default deny

```bash
kubectl apply -f ../../examples/security/networkpolicy/default-deny.yaml
```

Повторите nslookup/curl.

Если DNS перестал работать, вы увидели реальный эффект deny egress.

## 6. Разрешения

Создайте allow rules только для необходимых flows:
- DNS;
- caller -> backend:8080.

Проверьте снова.

## 7. RBAC

```bash
kubectl apply -f ../../examples/security/rbac/serviceaccount-role-binding.yaml

kubectl auth can-i get configmaps   --as=system:serviceaccount:net-lab:config-reader

kubectl auth can-i list secrets   --as=system:serviceaccount:net-lab:config-reader
```

Ожидайте:
- specific ConfigMap permission according to Role;
- отсутствие broad secrets access.

## 8. Контрольный алгоритм

При проблеме:

```text
DNS?
 -> TCP?
 -> TLS?
 -> HTTP status?
 -> auth?
```

Не называйте timeout «JWT problem».

## Cleanup

```bash
kubectl delete namespace net-lab
```
