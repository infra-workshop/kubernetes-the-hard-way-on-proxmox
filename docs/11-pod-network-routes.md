# Podが使うネットワーク経路のプロビジョニング

ノードにスケジュールされたPodは、ノードが持つPod CIDR範囲からIPアドレスを受け取ります。この時点ではネットワーク[経路](https://cloud.google.com/compute/docs/vpc/routes)が欠落しているため、Podは異なるノード上で動作している他のPodと通信できません。

本実習では、ノードのPod CIDR範囲をノードの内部IPアドレスにマップするための経路を各ワーカーノード上に作成します。

> Kubernetesのネットワーキングモデルを実装する方法は[他にも](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)あります。

## ルーティングテーブル

**各ワーカーノード上で**、以下のルールを追加します:

> **WARNING**: 作業中のワーカーノードに直接紐付いたPOD CIDRをルーティングのルールに追加しないでください(例: worker-0上で作業している時に10.200.0.0/24を追加しないでください).

```bash
ip route add 10.200.0.0/24 via 192.168.8.20 # worker-0では追加しない
ip route add 10.200.1.0/24 via 192.168.8.21 # worker-1では追加しない
ip route add 10.200.2.0/24 via 192.168.8.22 # worker-2では追加しない
```

> Don't take care of the `RTNETLINK answers: File exists` message, it appears just when you try to add an existing route, not a real problem.

List the routes in the `kubernetes-the-hard-way` VPC network:

```bash
ip route
```

> Output (example for worker-0):

```bash
default via 192.168.8.1 dev ens18 proto static
10.200.1.0/24 via 192.168.8.21 dev ens18
10.200.2.0/24 via 192.168.8.22 dev ens18
192.168.8.0/24 dev ens18 proto kernel scope link src 192.168.8.21
```

これを永続化するには、ネットワーク設定を編集する必要があります(Linuxディストロにより方法は異なります。

Example for **Ubuntu 18.04** and higher:

```bash
vi /etc/netplan/00-installer-config.yaml
```

> 設定内容 (example for worker-0, **作業中のワーカーノードに直接紐付いたPOD CIDRを指定しないでください**):

```bash
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens18:
      addresses:
      - 192.168.8.10/24
      gateway4: 192.168.8.1
      nameservers:
        addresses:
        - 9.9.9.9
      routes:
      - to: 10.200.1.0/24
        via: 192.168.8.21
      - to: 10.200.2.0/24
        via: 192.168.8.22
  version: 2
```

Next: [DNSクラスターアドオンのデプロイ](12-dns-addon.md)