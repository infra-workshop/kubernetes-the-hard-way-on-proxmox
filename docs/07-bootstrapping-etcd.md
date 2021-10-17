# etcdクラスターのブートストラップ

Kubernetesのコンポーネントはステートレスで、クラスターの状態を[etcd](https://github.com/etcd-io/etcd)に格納します。本実習では、3ノードのetcdクラスターをブートストラップし、高可用性と安全なリモートアクセスを実現するように構成します。

## 前提条件

本実習のコマンドは`controller-0`、`controller-1`、`controller-2`の各コントロールプレーン用インスタンスで実行する必要があります。Login to each controller instance using the `ssh` command. Example for `controller-0`:

```bash
ssh root@controller-0
```

### tmuxを使った並列なコマンド実行

[tmux](https://github.com/tmux/tmux/wiki)を使用すると複数のインスタンスで同時にコマンドを実行できます。前提条件の[tmuxを使った並列なコマンド実行](01-prerequisites.md#tmuxを使った並列なコマンド実行)セクションを参照してください。

## etcdクラスター内単一メンバーのブートストラップ

### etcdバイナリーのダウンロードとインストール

[etcd](https://github.com/etcd-io/etcd)のGitHubプロジェクトから、公式のetcdリリースのバイナリーをダウンロードします:

```bash
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```

`etcd`サーバと`etcdctl`コマンドを展開してインストールします:

```bash
tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
```

### etcdサーバーの設定

```bash
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

クライアントリクエストの処理とetcdクラスター・ピアとの通信にはインスタンスの内部IPアドレスが使用されます。以下のコマンドで現在作業中のインスタンスが持つ内部IPアドレスを取得します。Define the INTERNAL_IP (replace MY_NODE_INTERNAL_IP by the value):

```bash
INTERNAL_IP=MY_NODE_INTERNAL_IP
```

> Example for controller-0 : 192.168.8.10

各etcdメンバーには、etcdクラスタ内で一意の名前を付ける必要があります。現在作業中のインスタンスが持つホスト名と一致するようにetcd名を設定します:

```bash
ETCD_NAME=$(hostname -s)
```

systemdユニットファイル`etcd.service`を作成します:

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://192.168.8.10:2380,controller-1=https://192.168.8.11:2380,controller-2=https://192.168.8.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### etcdサーバーの起動

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

## 検証

etcdのクラスターメンバーの一覧を表示します:

```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> 出力結果:

```bash
3a57933972cb5131, started, controller-2, https://192.168.8.12:2380, https://192.168.8.12:2379
f98dc20bce6225a0, started, controller-0, https://192.168.8.10:2380, https://192.168.8.10:2379
ffed16798470cab5, started, controller-1, https://192.168.8.11:2380, https://192.168.8.11:2379
```

Next: [Kubernetesコントロールプレーンのブートストラップ](08-bootstrapping-kubernetes-controllers.md)
