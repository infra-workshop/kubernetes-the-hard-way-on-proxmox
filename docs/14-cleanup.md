# お掃除

本実習では、チュートリアルで作成した計算資源を削除します。

## 仮想マシン

作成した7台VMを停止します:

```bash
sudo shutdown -h now
```

PromoxのWebUIまたはCLIを用いて、すべてのVMを削除します。

```bash
sudo qm destroy <vmid> --purge
```

## ネットワーク

Delete the private Kubernetes network (`vmbr8`) via the Proxmox WebUI (to avoid fatal misconfiguration).
