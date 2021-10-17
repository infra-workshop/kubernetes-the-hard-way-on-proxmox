# リモートアクセス用のkubectl設定

本実習では、`admin`ユーザーの認証情報に基づいた`kubectl`コマンド用のkubeconfigファイルを生成します。

> 本実習で使用するコマンドは、管理クライアント証明書の生成に使用したディレクトリと同じディレクトリから実行してください。

## admin用Kubernetesコンフィグファイル

kubeconfigには接続先のKubernetes APIサーバーが必要です。高可用性をサポートするために、Kubernetes APIサーバの前面に配置した外部ロードバランサに割り当てられたIPアドレスが使用されます。

`admin`ユーザとして認証するのに適したkubeconfigファイルを生成します(replace MY_PUBLIC_IP_ADDRESS with your public IP address on the `gateway-01` VM):

```bash
KUBERNETES_PUBLIC_ADDRESS=MY_PUBLIC_IP_ADDRESS

kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

kubectl config use-context kubernetes-the-hard-way
```

## 検証

リモートにあるKubernetesクラスターのバージョンを確認します:

```bash
kubectl version
```

> Output:

```bash
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.5", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.5", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.5", Compiler:"gc", Platform:"linux/amd64"}
```

リモートにあるKubernetesクラスター上にあるノードの一覧を表示します:

```bash
kubectl get nodes
```

> 出力結果

```bash
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   90s   v1.21.5
worker-1   Ready    <none>   91s   v1.21.5
worker-2   Ready    <none>   90s   v1.21.5
```

Next: [Podが使うネットワーク経路のプロビジョニング](11-pod-network-routes.md)
