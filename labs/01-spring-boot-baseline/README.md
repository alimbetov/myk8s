# Lab 01 — Spring Boot Kubernetes baseline

## Goal

Научиться отличать Deployment, Pod, Service, readiness, ConfigMap/Secret и сетевую диагностику.

## Prerequisites

- Kubernetes/k3s cluster;
- kubectl;
- namespace permission;
- любой HTTP Spring Boot image с Actuator probes либо адаптация manifests под собственный image.

## Prepare

```bash
kubectl create namespace k8s-lab
kubectl config set-context --current --namespace=k8s-lab
kubectl apply -f ../../examples/spring-boot/base/serviceaccount.yaml
kubectl apply -f ../../examples/spring-boot/base/configmap.yaml
```

Создайте runtime Secret вручную, не из Git:

```bash
kubectl create secret generic spring-app-secret   --from-literal=DB_USERNAME=demo   --from-literal=DB_PASSWORD=demo
```

Адаптируйте image в deployment и примените:

```bash
kubectl apply -f ../../examples/spring-boot/base/deployment.yaml
kubectl apply -f ../../examples/spring-boot/base/service.yaml
kubectl apply -f ../../examples/spring-boot/base/pdb.yaml
kubectl get pod -w
```

## Expected

```bash
kubectl get deploy,pod,svc,endpointslice
kubectl rollout status deploy/spring-app
```

Deployment должен иметь available replica(s), Service — Ready endpoints.

## Break 1: readiness

Измените readiness path на несуществующий `/broken-readyz`.

```bash
kubectl edit deploy spring-app
kubectl get pod
kubectl get endpointslice -l kubernetes.io/service-name=spring-app -o yaml
kubectl describe pod <pod>
```

Ожидаемо container может оставаться Running, но Pod станет NotReady и исчезнет из ready endpoints.

Исправьте path обратно на `/readyz`.

## Break 2: wrong Secret

```bash
kubectl delete secret spring-app-secret
kubectl rollout restart deploy/spring-app
kubectl describe pod <new-pod>
```

Pod не должен успешно стартовать, пока required Secret отсутствует.

Восстановить:

```bash
kubectl create secret generic spring-app-secret   --from-literal=DB_USERNAME=demo   --from-literal=DB_PASSWORD=demo
kubectl rollout restart deploy/spring-app
```

## Break 3: DNS / Service

Создайте debug Pod:

```bash
kubectl run net-debug --image=curlimages/curl -- sleep 3600
kubectl exec net-debug -- nslookup spring-app
kubectl exec net-debug -- curl -v http://spring-app:8080/readyz
```

Затем временно измените selector Service так, чтобы endpoints исчезли. Сравните:
- DNS resolution;
- Service existence;
- EndpointSlice contents;
- HTTP result.

## CKAD focus

Нужно быстро уметь:
- `get/describe/logs/exec`;
- отличить Running от Ready;
- найти selector mismatch;
- создать ConfigMap/Secret;
- понять Event при missing Secret.

## Cleanup

```bash
kubectl delete namespace k8s-lab
```
