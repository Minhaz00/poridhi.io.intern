# Deploying the DNS Cluster Add-on

In this lab you will deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery, backed by [CoreDNS](https://coredns.io/), to applications running inside the Kubernetes cluster.

## The DNS Cluster Add-on

Deploy the `coredns` cluster add-on:

```sh
kubectl apply -f coredns.yml
```

> output

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

List the pods created by the `core-dns` deployment:

```sh
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> output

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-hh7r2   1/1     Running   0          10s
coredns-8494f9c688-zqrj2   1/1     Running   0          10s
```

## Verification (stucked)

Create a `busybox` deployment: `busybox-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  annotations:
    container.apparmor.security.beta.kubernetes.io/busybox: unconfined
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    command: ["sleep", "3600"]
```

```sh
kubectl apply -f busybox-pod.yaml
```

> output

```sh
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Retrieve the full name of the `busybox` pod:

```sh
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```sh
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> output

```sh
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [Smoke Test](13-smoke-test.md)



