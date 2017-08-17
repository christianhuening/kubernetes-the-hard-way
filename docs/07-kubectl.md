# Configuring the Kubernetes Client - Remote Access

Run the following commands from the machine which will be your Kubernetes Client

## Download and Install kubectl
### Windows
```
1. Get kubectl from there: https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/windows/amd64/kubectl.exe
2. Put it into your PATH
```

### OS X

```
curl -O https://storage.googleapis.com/kubernetes-release/release/v1.7.2/bin/darwin/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin
```

### Linux

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.2/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin
```

## Configure Kubectl

In this section you will configure the kubectl client to point to the [Kubernetes API Server Frontend Load Balancer](04-kubernetes-controller.md#setup-kubernetes-api-server-frontend-load-balancer).

```
KUBERNETES_PUBLIC_ADDRESS=$(dig +short YOUR_API_LOADBALANCER_SERVER_HOSTNAME | tail -1)
```

Also be sure to locate the CA certificate [created earlier](02-certificate-authority.md). Since we are using self-signed TLS certs we need to trust the CA certificate so we can verify the remote API Servers.

### Build up the kubeconfig entry

The following commands will build up the default kubeconfig file used by kubectl.

> if you are using gitbash on windows to run kubectl, you need to be on the same drive as your `~/.kube` folder. The easiest way is to copy the needed certs (`ca.pem`, `admin.pem`, `admin-key.pem`) into `~/.kube/certs/icc` and provide relative paths to there to the following commands.

```
kubectl config set-cluster informatik-compute-cloud \
  --certificate-authority=~/.kube/certs/icc/ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443
```

```
kubectl config set-credentials admin \
  --client-certificate=~/.kube/certs/icc/admin.pem \
  --client-key=~/.kube/certs/icc/admin-key.pem
```


```
kubectl config set-context informatik-compute-cloud \
  --cluster=informatik-compute-cloud \
  --user=admin
```

```
kubectl config use-context informatik-compute-cloud
```

At this point you should be able to connect securly to the remote API server:

```
kubectl get componentstatuses
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

```
kubectl get nodes
```

```
NAME      STATUS    AGE       VERSION
worker0   Ready     7m        v1.6.0-rc.1
worker1   Ready     5m        v1.6.0-rc.1
worker2   Ready     2m        v1.6.0-rc.1
```
