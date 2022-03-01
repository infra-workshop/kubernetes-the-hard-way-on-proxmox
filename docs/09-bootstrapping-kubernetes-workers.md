# Kubenretesワーカーノードのブートストラップ

本実習では、3つのKubernetesワーカーノードをブートストラップします。各ノードには[runc](https://github.com/opencontainers/runc)、[CNI](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd)、[kubelet](https://kubernetes.io/docs/admin/kubelet)および[kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies)がインストールされます。

## 前提条件

本実習のコマンドは`worker-0`, `worker-1`, and `worker-2`の各ワーカーノード用インスタンスで実行する必要があります。Login to each worker instance using the `ssh` command. Example:

```bash
ssh root@worker-0
```

### tmuxを使った並列なコマンド実行

[tmux](https://github.com/tmux/tmux/wiki)を使用すると複数のインスタンスで同時にコマンドを実行できます。前提条件の[tmuxを使った並列なコマンド実行](01-prerequisites.md#tmuxを使った並列なコマンド実行)セクションを参照してください。

## 単一Kubernetesワーカーノードのプロビジョニング

OSの依存ライブラリをインストールします:

```bash
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
```

> socatバイナリーは`kubectl port-forward`コマンドのサポートを有効にします。

### Swapの無効化

デフォルトで[swap](https://help.ubuntu.com/community/SwapFaq)が有効になっている場合、kubeletの起動は失敗します。Kubernetesが適切なリソース割り当てとサービス品質を提供できるように、swapを無効にすることを[推奨](https://github.com/kubernetes/kubernetes/issues/7294)します。

swapが有効になっているか確認します:

```bash
sudo swapon --show
```

出力が空の場合、swapは有効になっていません。有効になっている場合は、次のコマンドを実行してただちに無効にします:

```bash
sudo swapoff -a
```

> インスタンスの再起動後もswapがオフのままになるようにするには、Linuxディストリビューションのドキュメントを参照してください。You may need to comment the Swap line in the `/etc/fstab` file.

### ワーカーバイナリーのダウンロードとインストール

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kubelet
```

インストール用ディレクトリを作成します:

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

ワーカーバイナリーをインストールします:

```bash
mkdir containerd
tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
sudo mv runc.amd64 runc
chmod +x crictl kubectl kube-proxy kubelet runc
sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
sudo mv containerd/bin/* /bin/
```

### CNIのネットワーク設定

現在作業中のインスタンスが持つPodのCIDR範囲を定義します(各ワーカー違うものを使います)。Replace THE_POD_CIDR by the CIDR network for this node (see network architecture):

```bash
POD_CIDR=THE_POD_CIDR
```

> Example for worker-0: 10.200.0.0/24

Create the `bridge` network configuration file:

```bash
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Create the `loopback` network configuration file:

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### containerdの設定

`containerd`の設定ファイルを作成します:

```bash
sudo mkdir -p /etc/containerd/
```

```bash
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

systemdユニットファイル`containerd.service`を作成します:

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Kubeletの設定

```bash
sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

設定ファイル`kubelet-config.yaml`を作成します:

```bash
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

> `resolvConf`の設定は、`system-resolved`を実行しているシステムのサービスディスカバリにCoreDNSを使用する場合に、ループを回避するために使用されます。

systemdユニットファイル`kubelet.service`を作成します:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetes Proxyの設定

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

設定ファイル`kube-proxy-config.yaml`を作成します:

```bash
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

systemdユニットファイル`kube-proxy.service`を作成します:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### ワーカーサービスの起動

```bash
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

> 上記のコマンドは各コントローラノード`worker-0`、`worker-1`、`worker-2`にて忘れずに実行してください。

## 検証

登録されたKubernetesのノード一覧を表示します:

```bash
ssh root@controller-0 kubectl get nodes --kubeconfig admin.kubeconfig
```

> 出力結果

```bash
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   15s   v1.21.5
worker-1   Ready    <none>   15s   v1.21.5
worker-2   Ready    <none>   15s   v1.21.5
```

Next: [リモートアクセス用のkubectl設定](10-configuring-kubectl.md)
