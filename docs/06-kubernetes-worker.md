# Bootstrapping Kubernetes Workers

In this lab you will bootstrap 3 Kubernetes worker nodes. The following virtual machines will be used:

* icc-node-1
* icc-node-2
* icc-node-3

## Why

Kubernetes worker nodes are responsible for running your containers. All Kubernetes clusters need one or more worker nodes. We are running the worker nodes on dedicated machines for the following reasons:

* Ease of deployment and configuration
* Avoid mixing arbitrary workloads with critical cluster components. We are building machine with just enough resources so we don't have to worry about wasting resources.

Some people would like to run workers and cluster services anywhere in the cluster. This is totally possible, and you'll have to decide what's best for your environment.

## Prerequisites


Enable TLS bootstrapping by binding the `kubelet-bootstrap` user to the `system:node-bootstrapper` cluster role:

```
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

The boostrapping of Kubelet client and server certificates is done automatically via the new 'Kubelet TLS bootstrapping'
feature (https://github.com/kubernetes/features/issues/43), which is currently deployed in ALPHA status and thus requires
the ALPHA feature-gates to be enabled as can be seen below. The docoumentation for the TLS Bootstrapping in K8s can 
be found here: https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/ .


## Provision the Kubernetes Worker Nodes

Git-Clone the https://gitlab.informatik.haw-hamburg.de/ail/puppet-ail_role repo and checkout the icc_prep branch.
Git-Clone the https://gitlab.informatik.haw-hamburg.de/ail/puppet-ail_feature and checkout the icc_setup repo.

This documentation will replace your actual path to the repository with AIL_ROLE and AIL_FEATURE respectively.


Run the following commands on `worker0`, `worker1`, `worker2`:

#### Create directories

```
sudo mkdir -p /var/lib/{kubelet,kube-proxy,kubernetes}
```

```
sudo mkdir -p /var/run/kubernetes
```

```
sudo mkdir -p /etc/kubernetes/manifests
```

```
sudo mkdir -p /etc/ceph
```

#### Install CEPH and copy CEPH config files

```
sudo apt-get install ceph-common
```

```
sudo cp AIL_ROLE/icc/auth/ceph.client.admin.keyring /etc/ceph/keyring
```

```
sudo cp AIL_ROLE/icc/auth/ceph.conf /etc/ceph/ceph.conf
```


#### Copy configs for the kubelet and kube-proxy

```
sudo cp AIL_ROLE/icc/auth/bootstrap.kubeconfig /var/lib/kubelet/
```


```
sudo cp AIL_ROLE/icc/auth/proxy-kubeconfig-yaml /var/lib/kubelet/proxy-kubeconfig.yaml
```

```
sudo cp AIL_ROLE/icc/auth/worker-kubeconfig.yaml /var/lib/kubelet/worker-kubeconfig.yaml
```

#### Move the TLS certificates in place

```
sudo cp AIL_FEATURE/icc/CA/ca.pem /var/lib/kubernetes/
```

```
sudo cp AIL_FEATURE/icc/CA/kube-proxy.pem /var/lib/kubernetes/
```

```
sudo cp AIL_FEATURE/icc/CA/kube-proxy-key.pem /var/lib/kubernetes/
```

### Install Docker

```
apt-get install docker.io
```
> make sure that your installed Ubuntu version contains Docker >= 0.12.2. Ubuntu 17.04+ will do so.

Create the Docker systemd unit file:

Ubuntu will create a service that will only listen on the TCP Port. But we need a docker servvice that will only listen on the socket.
```
cat > docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
ExecStart=/usr/bin/docker daemon \\
  --iptables=false \\
  --ip-masq=false \\
  --host=unix:///var/run/docker.sock \\
  --log-level=error \\
  --storage-driver=overlay
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start the docker service:

```
sudo mv docker.service /etc/systemd/system/docker.service
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable docker
```

```
sudo systemctl start docker
```

```
sudo docker version
```

### Install the kubelet

The Kubelet can now use [CNI - the Container Network Interface](https://github.com/containernetworking/cni) to manage machine level networking requirements.

Download and install CNI plugins

```
sudo mkdir -p /opt/cni/bin
```

```
wget https://github.com/containernetworking/cni/releases/download/v0.5.2/cni-amd64-v0.5.2.tgz
```

```
sudo tar -xvf cni-amd64-v0.5.2.tar.gz -C /opt/cni/bin
```

Download and install the Kubernetes worker binaries:

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.2/bin/linux/amd64/kubectl
```

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.2/bin/linux/amd64/kubelet
```

```
chmod +x kubectl kubelet
```

```
sudo mv kubectl kubelet /usr/bin/
```

Create the kubelet systemd unit file:

```
export KUBERNETES_PUBLIC_ADDRESS=$(dig +short icc-k8s-api.informatik.haw-hamburg.de | tail -1)
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/kubelet \
  --allow-privileged=true \
  --cluster-dns=10.32.0.10,10.32.0.11,10.32.0.12 \
  --cluster-domain=cluster.local \
  --container-runtime=docker \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --require-kubeconfig \
  --bootstrap-kubeconfig=/var/lib/kubelet/bootstrap.kubeconfig \
  --network-plugin=cni \
  --kubeconfig=/var/lib/kubelet/worker-kubeconfig.yaml \
  --serialize-image-pulls=false \
  --register-node=true \
  --cert-dir=/var/lib/kubernetes \
  --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

EOF
```

```
sudo mv kubelet.service /etc/systemd/system/kubelet.service
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable kubelet
```

```
sudo systemctl start kubelet
```

```
sudo systemctl status kubelet --no-pager
```

#### kube-proxy

```
cat > kube-proxy.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
    command:
    - /hyperkube
    - proxy
    - --master=https://YOURMASTERIP:443
    - --cluster-cidr=10.200.0.0/16
    - --kubeconfig=/etc/kubernetes/proxy-kubeconfig.yaml
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: "ssl-certs"
    - mountPath: /etc/kubernetes/proxy-kubeconfig.yaml
      name: "kubeconfig"
      readOnly: true
    - mountPath: /var/lib/kubernetes/
      name: "etc-kube-ssl"
      readOnly: true
  volumes:
  - name: "ssl-certs"
    hostPath:
      path: "/usr/share/ca-certificates"
  - name: "kubeconfig"
    hostPath:
      path: "/var/lib/kubelet/proxy-kubeconfig.yaml"
  - name: "etc-kube-ssl"
    hostPath:
      path: "/var/lib/kubernetes"
EOF
```

```
sudo mv kube-proxy.yaml /etc/kubernetes/manifests/kube-proxy.yaml
```


> Remember to run these steps on `worker0`, `worker1`, and `worker2`

## Approve the TLS certificate requests

Each worker node will submit a certificate signing request which must be approved before the node is allowed to join the cluster.
This approval will be handled by the 'csrapproval' controller which is included with the kube-management-controller since
Kubernetes 1.7.x. According to the documentation (https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/#approval-controller) 
this features has been enabled in our cluster by providing ClusterRoleBindings (see https://gitlab.informatik.haw-hamburg.de/icc/kubernetes-the-icc-way/tree/master/deployments/ApprovalController for details).

Once all certificate signing requests have been approved all nodes should be registered with the cluster:

```
kubectl get nodes
```

```
NAME      STATUS    AGE       VERSION
worker0   Ready     7m        v1.6.2
worker1   Ready     5m        v1.6.2
worker2   Ready     2m        v1.6.2
```
