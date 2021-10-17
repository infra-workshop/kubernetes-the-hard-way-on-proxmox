# Kubernetesコントロールプレーンのブートストラップ

本実習では、3つのインスタンスでコントロールプレーンをブートストラップして可用性の高い構成を実現します。また、KubernetesのAPIサーバーをリモートクライアントに公開する外部ロードバランサも作成します。各ノードには、Kubernetes API Server、Scheduler、Controller Managerの各コンポーネントがインストールされます。

## 前提条件

本実習のコマンドは`controller-0`、`controller-1`、`controller-2`の各コントロールプレーン用インスタンスで実行する必要があります。Login to each controller instance using the `ssh` command. Example:

```bash
ssh root@controller-0
```

### tmuxを使った並列なコマンド実行

[tmux](https://github.com/tmux/tmux/wiki)を使用すると複数のインスタンスで同時にコマンドを実行できます。前提条件の[tmuxを使った並列なコマンド実行](01-prerequisites.md#tmuxを使った並列なコマンド実行)セクションを参照してください。

## Kubernetesコントロールプレーンのプロビジョニング

Kubenretesのコンフィグ用ディレクトリを作成します:

```bash
sudo mkdir -p /etc/kubernetes/config
```

### Kubernetesコントローラー用バイナリーのダウンロードとインストール

Kubernetesの公式リリースバイナリーをダウンロードします:

```bash
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kubectl"
```

Kubernetesバイナリーをインストールします:

```bash
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### Kubernetes APIサーバーの設定

```bash
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

クラスターのメンバーにAPIサーバーを通知するためにインスタンスの内部IPアドレスが使用されます。Define the INTERNAL_IP (replace MY_NODE_INTERNAL_IP by the value):

```bash
INTERNAL_IP=MY_NODE_INTERNAL_IP
```

> Example for controller-0 : 192.168.8.10

systemdユニットファイル`kube-apiserver.service`を作成します:

```bash
KUBERNETES_PUBLIC_ADDRESS=MY_PUBLIC_IP_ADDRESS

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.8.10:2379,https://192.168.8.11:2379,https://192.168.8.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetesコントローラーマネージャーの設定

kubeconfig`kube-controller-manager`を以下の場所に配置します:

```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

systemdユニットファイル`kube-controller-manager.service`を作成します:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetesスケジューラーの設定

kubeconfig`kube-scheduler`を以下の場所に配置します:

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

設定ファイル`kube-scheduler.yaml`を作成します:

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

systemdユニットファイル`kube-scheduler.service`を作成します:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### コントローラーのサービスを起動

```bash
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> KubernetesのAPIサーバーが完全に初期化されるまでには最大10秒かかります。

### 検証

```bash
kubectl cluster-info --kubeconfig admin.kubeconfig
```

```bash
Kubernetes control plane is running at https://127.0.0.1:6443
```

Test the HTTPS health check :

```bash
curl -kH "Host: kubernetes.default.svc.cluster.local" -i https://127.0.0.1:6443/healthz
```

```bash
HTTP/2 200
content-type: text/plain; charset=utf-8
x-content-type-options: nosniff
content-length: 2
date: Wed, 24 Jun 2020 12:24:52 GMT

ok
```

> 上記のコマンドは各コントローラノード`controller-0`、`controller-1`、`controller-2`にて忘れずに実行してください。

## RBACを使ったKubeletの認可

本セクションでは、RBACのアクセス権を設定して、KubernetesのAPIサーバーが各ワーカーノード上のKubeletにアクセスできるようにします。メトリクスやログを取得したり、Pod内でコマンドを実行するためにはKubelet APIへのアクセスが必要です。

> 本チュートリアルでは、Kubeletの`--authorization-mode`フラグを`Webhook`に設定します。Webhookモードでは、[SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) APIを使用して認可を判定します。

本セクションのコマンドはクラスター全体に影響するため、1つのコントローラーノードから1回実行するだけで大丈夫です。

```bash
ssh root@controller-0
```

`system:kube-apiserver-to-kubelet`という名前の[ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole)を作成し、Kubelet APIにアクセスしたり、Podの管理に関連する一般的なタスクを実行したりするための権限を付与します:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Kubernetes APIサーバーは、`--kubelet-client-certificate`フラグで定義されているクライアント証明書を使用して、`kubernetes`ユーザーとしてKubeletに認証します。

ClusterRole`system:kube-apiserver-to-kubelet`を`kubernetes`ユーザーにバインドします:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## Kubernetesのフロントエンドロードバランサー

In this section you will provision an Nginx load balancer to front the Kubernetes API Servers. The load balancer will listen on the private and the public IP address (on the `gateway-01` VM).

### Provision an Nginx Load Balancer

Install the Nginx Load Balancer:

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

As **root** user, Create the Nginx load balancer network configuration:

```bash
cat <<EOF >> /etc/nginx/nginx.conf
stream {
    upstream controller_backend {
        server 192.168.8.10:6443;
        server 192.168.8.11:6443;
        server 192.168.8.12:6443;
    }
    server {
        listen     6443;
        proxy_pass controller_backend;
        # health_check; # Only Nginx commercial subscription can use this directive...
    }
}
EOF
```

Restart the service:

```bash
sudo systemctl restart nginx
```

Enable the service:

```bash
sudo systemctl enable nginx
```

### Load Balancer Verification

Define the static public IP address (replace MY_PUBLIC_IP_ADDRESS with your public IP address on the `gateway-01` VM):

```bash
KUBERNETES_PUBLIC_ADDRESS=MY_PUBLIC_IP_ADDRESS
```

Make a HTTP request for the Kubernetes version info:

```bash
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> output

```bash
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.5",
  "gitCommit": "c96aede7b5205121079932896c4ad89bb93260af",
  "gitTreeState": "clean",
  "buildDate": "2020-06-17T11:33:59Z",
  "goVersion": "go1.16.5",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Kubenretesワーカーノードのブートストラップ](09-bootstrapping-kubernetes-workers.md)
