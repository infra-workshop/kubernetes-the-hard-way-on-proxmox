# スモークテスト

本実習では、Kubernetesクラスターが正常に機能していることを確認するために必要な一連のタスクを実行します。

## データの暗号化

本セクションでは、[保存されている秘密データを暗号化する](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted)機能を確認します。

Generic形式のSecretデータを作成します:

```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

etcdに保存されているsecretデータ`kubernets-the-hard-way`のhexdumpを表示します:

```bash
ssh root@controller-0 \
  sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```

> 出力結果

```bash
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 44 ac 6e ac 11 2f 28  |:v1:key1:D.n../(|
00000050  02 46 3d ad 9d cd 68 be  e4 cc 63 ae 13 e4 99 e8  |.F=...h...c.....|
00000060  6e 55 a0 fd 9d 33 7a b1  17 6b 20 19 23 dc 3e 67  |nU...3z..k .#.>g|
00000070  c9 6c 47 fa 78 8b 4d 28  cd d1 71 25 e9 29 ec 88  |.lG.x.M(..q%.)..|
00000080  7f c9 76 b6 31 63 6e ea  ac c5 e4 2f 32 d7 a6 94  |..v.1cn..../2...|
00000090  3c 3d 97 29 40 5a ee e1  ef d6 b2 17 01 75 a4 a3  |<=.)@Z.......u..|
000000a0  e2 c2 70 5b 77 1a 0b ec  71 c3 87 7a 1f 68 73 03  |..p[w...q..z.hs.|
000000b0  67 70 5e ba 5e 65 ff 6f  0c 40 5a f9 2a bd d6 0e  |gp^.^e.o.@Z.*...|
000000c0  44 8d 62 21 1a 30 4f 43  b8 03 69 52 c0 b7 2e 16  |D.b!.0OC..iR....|
000000d0  14 a5 91 21 29 fa 6e 03  47 e2 06 25 45 7c 4f 8f  |...!).n.G..%E|O.|
000000e0  6e bb 9d 3b e9 e5 2d 9e  3e 0a                    |n..;..-.>.|
```

etcd上のキーのプレフィックスには`k8s:enc:aescbc:v1:key1`が付いているはずです。これは、`aescbc`プロバイダが暗号化されたキー`key1`のデータを暗号化するために使用されたことを示します。

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```bash
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```bash
kubectl get pods -l app=nginx
```

> Output (you may need to wait a few seconds to see the pod "READY"):

```bash
NAME                     READY   STATUS    RESTARTS   AGE
nginx-554b9c67f9-vt5rn   1/1     Running   0          10s
```

## Deployment

本セクションでは、[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)を作成および管理する機能を確認します。

[nginx](https://nginx.org/en/)のDeploymentを作成します:

```
kubectl create deployment nginx --image=nginx
```

Deploymentリソース`nginx`によって作られたPodの一覧を表示します:

```
kubectl get pods -l app=nginx
```

> 出力結果

```
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-kpn5m   1/1     Running   0          10s
```

### ポート転送

本セクションでは、[ポート転送](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)を使用してアプリケーションにリモートでアクセスする機能を確認します。

Podリソース`nginx`のフルネームを取得します:

```bash
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

`nginx`Podの`80`番ポートを、手元のマシンの`8080`番ポートに転送します:

```bash
kubectl port-forward $POD_NAME 8080:80
```

> 出力結果

```bash
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

新しいターミナルを開き、転送済アドレスを使ってHTTPリクエストを発行します:

```bash
curl --head http://127.0.0.1:8080
```

> 出力結果

```bash
HTTP/1.1 200 OK
Server: nginx/1.19.0
Date: Wed, 24 Jun 2020 12:55:15 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 26 May 2020 15:00:20 GMT
Connection: keep-alive
ETag: "5ecd2f04-264"
Accept-Ranges: bytes
```

前のターミナルに戻り、`nginx`Podへのポート転送を停止します:

```bash
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### ログ

本セクションでは、[コンテナのログを取得](https://kubernetes.io/docs/concepts/cluster-administration/logging/)する機能を確認します。

`nginx`Podのログを表示します:

```bash
kubectl logs $POD_NAME
```

> 出力結果

```bash
127.0.0.1 - - [24/Jun/2020:12:55:15 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.64.0" "-"
```

### コマンドの実行

本セクションでは、[コンテナ内でコマンドが実行できる](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container)ことを確認します。

`nginx`コンテナで`nginx-v`コマンドを実行して、nginxバージョンを表示します:

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

> 出力結果

```bash
nginx version: nginx/1.19.0
```

## サービス

本セクションでは、[サービス](https://kubernetes.io/docs/concepts/services-networking/service/)を使用してアプリケーションを公開する機能を確認します。

[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)サービスを使用して`nginx`のDeploymentを公開します。

```bash
kubectl expose deployment nginx --port 80 --type NodePort
```

>クラスターが[クラウドプロバイダーインテグレーション](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider)で構成されていないため、サービスのtype:LoadBalancerを使用できません。クラウドプロバイダーインテグレーションのセットアップについては本チュートリアルの対象外です。

`nginx`サービスに割り当てられたノード上のポートを取得します:

```bash
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Define the Kubernetes network IP address of a worker instance (replace MY_WORKER_IP with the private IP defined on a worker):

```bash
NODE_IP=MY_WORKER_IP
```

> Example for worker-0: 192.168.8.20

設定したワーカーノードのIPアドレスと`nginx`のノードポートを使ってHTTPリクエストを発行します:

```bash
curl -I http://${NODE_IP}:${NODE_PORT}
```

> 出力結果

```bash
HTTP/1.1 200 OK
Server: nginx/1.19.0
Date: Wed, 24 Jun 2020 12:57:37 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 26 May 2020 15:00:20 GMT
Connection: keep-alive
ETag: "5ecd2f04-264"
Accept-Ranges: bytes
```

Next: [お掃除](14-cleanup.md)
