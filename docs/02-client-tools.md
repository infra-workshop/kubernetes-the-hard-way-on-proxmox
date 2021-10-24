# クライアントツールのインストール

本実習では、チュートリアルの実行に必要なコマンドである[cfssl](https://github.com/cloudflare/cfssl)、[cfssljson](https://github.com/cloudflare/cfssl)、[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)をインストールします。


## cfsslのインストール

[公開鍵基盤](https://ja.wikipedia.org/wiki/%E5%85%AC%E9%96%8B%E9%8D%B5%E5%9F%BA%E7%9B%A4)のプロビジョニングとTLS証明書の生成には、`cfssl`および`cfssljson`コマンドラインユーティリティが使用されます。

On the **gateway-01** VM, download and install `cfssl` and `cfssljson`:

```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssljson
```

```bash
chmod +x cfssl cfssljson
```

```bash
sudo mv cfssl cfssljson /usr/local/bin/
```

### 検証

インストールされた`cfssl`と`cfssljson`のバージョンが1.4.1以上であることを検証します:

```
cfssl version
```

> 出力結果

```
Version: 1.4.1
Runtime: go1.12.12
```

```
cfssljson --version
```
```
Version: 1.4.1
Runtime: go1.12.12
```

## kubectlのインストール

`kubectl`コマンドは、KubernetesのAPIサーバーとの対話に使用されます。On the **gateway-01** VM, 公式リリースバイナリーから`kubectl`をダウンロードしてインストールします:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kubectl
```

```bash
chmod +x kubectl
```

```bash
sudo mv kubectl /usr/local/bin/
```

### 検証

インストールされた`kubectl`のバージョンが1.21.5以上であることを検証します:

```
kubectl version --client
```

> 出力結果

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.5", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [計算資源のプロビジョニング](03-compute-resources.md)
