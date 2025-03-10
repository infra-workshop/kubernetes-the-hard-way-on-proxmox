# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user (replace MY_PUBLIC_IP_ADDRESS with your public IP address on the `gateway-01` VM):

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

## Verification

Check the health of the remote Kubernetes cluster:

```bash
kubectl get componentstatuses
```

> Output:

```bash
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
```

List the nodes in the remote Kubernetes cluster:

```bash
kubectl get nodes
```

> Output:

```bash
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   90s   v1.21.5
worker-1   Ready    <none>   91s   v1.21.5
worker-2   Ready    <none>   90s   v1.21.5
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
