# DNSクラスターアドオンのデプロイ

本実習では、[CoreDNS](https://coredns.io/)によってサポートされるDNSベースのサービスディスカバリを提供する[アドオン](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)を、Kubernetesクラスター内で稼働するアプリケーションに導入します。

## DNSクラスターアドオン

CoreDNS yamlを取得します:

```bash
wget https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```

Edit the `coredns.yaml` file to change CoreDNS configuration to enable DNS resolution for external name:

```bash
sed '/.*prometheus :9153/a \ \ \ \ \ \ \ \ forward . /etc/resolv.conf' coredns.yaml
```

クラスターアドオン`coredns`をデプロイします:

```bash
kubectl apply -f coredns.yaml
```

> 出力結果

```bash
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

Deploymentリソース`kube-dns`によって作られたPodの一覧を表示します:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 出力結果(you may need to wait a few seconds to see the pods "READY"):

```bash
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

## 検証

Deploymentリソース`busybox`をデプロイします:

```bash
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

Deploymentリソース`busybox`によって作られたPodの一覧を表示します:

```bash
kubectl get pods -l run=busybox
```

> 出力結果(you may need to wait a few seconds to see the pod "READY"):

```bash
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Podリソース`busybox`のフルネームを取得します:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

`busybox`の中で`kubernetes`のサービスに対するDNSルックアップを実行します:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 出力結果

```bash
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [スモークテスト](13-smoke-test.md)
