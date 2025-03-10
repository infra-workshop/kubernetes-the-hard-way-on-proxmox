# Deploying the DNS Cluster Add-on

In this lab you will deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery, backed by [CoreDNS](https://coredns.io/), to applications running inside the Kubernetes cluster.

## The DNS Cluster Add-on

Get the CoreDNS yaml:

```bash
wget https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml
```

Edit the `coredns.yaml` file to change CoreDNS configuration to enable DNS resolution for external name:

```bash
sed '/.*prometheus :9153/a \ \ \ \ \ \ \ \ forward . /etc/resolv.conf' coredns.yaml
```

Deploy the `coredns` cluster add-on:

```bash
kubectl apply -f coredns.yaml
```

> Output:

```bash
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

List the pods created by the `kube-dns` deployment:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> Output (you may need to wait a few seconds to see the pods "READY"):

```bash
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

## Verification

Create a `busybox` deployment:

```bash
kubectl run --generator=run-pod/v1 busybox --image=busybox:1.28 --command -- sleep 3600
```

List the pod created by the `busybox` deployment:

```bash
kubectl get pods -l run=busybox
```

> Output (you may need to wait a few seconds to see the pod "READY"):

```bash
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Retrieve the full name of the `busybox` pod:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> Output:

```bash
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [Smoke Test](13-smoke-test.md)
