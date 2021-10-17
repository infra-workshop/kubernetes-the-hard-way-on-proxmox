# データ暗号化の設定とキーの生成

Kubernetesはクラスタの状態、アプリケーションの構成、機密情報など、さまざまなデータを保存します。Kubernetesはクラスタデータを[暗号化](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data)する機能をサポートします。

本実習では、Kubernetes Secretsの暗号化に適したキーと[コンフィグ](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration)を生成します。

## 暗号化キー

暗号化キーを生成します:

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## 暗号化コンフィグファイル

`encryption-config.yaml`という名前の暗号化コンフィグファイルを生成します:

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

暗号化コンフィグファイル`encryption-config.yaml`を各コントロールプレーン用インスタンスにコピーします:

```bash
for instance in controller-0 controller-1 controller-2; do
  scp encryption-config.yaml root@${instance}:~/
done
```

Next: [etcdクラスターのブートストラップ](07-bootstrapping-etcd.md)
