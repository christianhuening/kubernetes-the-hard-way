> some explanations are taken from here: https://coreos.com/kubernetes/docs/latest/deploy-master.html

> The older documentation for running the master nodes as services on bare-metal can be found [here](05-kubernetes-controller_legacy.md)

# Bootstrapping H/A Kubernetes Master Nodes

We will now bootstrap a 3 node Kubernetes controller cluster. The following virtual machines will be used:

* icc-ctrl-1
* icc-ctrl-2
* icc-ctrl-3

We will use the frontend load balancers created [earlier](02.5-haproxy.md) to provide a public IP address for remote access to the API servers and H/A.

## Why

The Kubernetes components that make up the control plane include the following components:

* API Server
* Scheduler
* Controller Manager

Each component is being run on the same machine for the following reasons:

* The Scheduler and Controller Manager are tightly coupled with the API Server
* Only one Scheduler and Controller Manager can be active at a given time, but it's ok to run multiple at the same time. Each component will elect a leader via the API Server.
* Running multiple copies of each component is required for H/A
* Running each component next to the API Server eases configuration.

Each of the master components is safe to run on multiple nodes. The apiserver is stateless, but handles recording the results of leader elections to etcd on behalf of other master components. The controller-manager and scheduler use the leader election mechanism to ensure only one of each is active, leaving the inactive master components ready to assume responsibility in case of failure.

We will run all above mentioned services as Containers on the controller nodes. This has the added benefit that
we do not need to install the overlay network on each node directly but let that handle
by the cluster itself. To bring up the containers we will use the Kubernetes Kubelet in a special way (see below).

## Provision the Kubernetes Controller Cluster

Run the following commands on `icc-ctrl-1`, `icc-ctrl-2`, `icc-ctrl-3`:

---

Copy the bootstrap token into place:

```
sudo mkdir -p /var/lib/kubernetes/
```

```
sudo mv token.csv /var/lib/kubernetes/
```

### TLS Certificates

The TLS certificates created in the [Setting up a CA and TLS Cert Generation](02-certificate-authority.md) lab will be used to secure communication between the Kubernetes API server and Kubernetes clients such as `kubectl` and the `kubelet` agent. The TLS certificates will also be used to authenticate the Kubernetes API server to etcd via TLS client auth.

Copy the TLS certificates to the Kubernetes configuration directory:

```
sudo mv ca-key.pem apiserver.pem apiserver-key.pem kubernetes.pem kubernetes-key.pem kubernetes-ldap-client.pem kubernetes-ldap-client.key kube-proxy.pem kube-proxy-key.pem /var/lib/kubernetes/
```


### Install kubectl:

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.7.2/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/bin/
```

### Install the kubelet
The kubelet is the agent on each machine that starts and stops Pods and other machine-level tasks. The kubelet communicates with the API server (also running on the master nodes) with the TLS certificates we placed on disk earlier.

On the master node, the kubelet is configured to communicate with the API server, but not register for cluster work, as shown in the --register-schedulable=false line in the YAML excerpt below. This prevents user pods being scheduled on the master nodes, and ensures cluster work is routed only to task-specific worker nodes.

Since we're using Calico + Flannel as Overlay Network solution, the kubelet is configured to use the Container Networking Interface (CNI) standard for networking. This makes Calico aware of each pod that is created and allows it to network the pods into the flannel overlay. Both flannel and Calico communicate via CNI interfaces to ensure the correct IP range (managed by flannel) is used for each node.

Note that the kubelet running on a master node may log repeated attempts to post its status to the API server. These warnings are expected behavior and can be ignored. Future Kubernetes releases plan to handle this common deployment consideration more gracefully.

Create this file:

#### /etc/systemd/system/kubelet.service
```
cat >  /etc/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/kubelet \
  --api-servers=http://127.0.0.1:8080 \
  --allow-privileged=true \
  --register-schedulable=false \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --cluster-dns=10.32.0.10,10.32.0.11,10.32.0.12 \
  --cluster-domain=cluster.local \
  --container-runtime=docker \
  --network-plugin=cni \
  --tls-cert-file=/var/lib/kubernetes/apiserver.pem \
  --tls-private-key-file=/var/lib/kubernetes/apiserver-key.pem \
  --cert-dir=/var/lib/kubernetes \
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

EOF
```

### Setup Kubernetes API Server Pod

The API server is where most of the magic happens. It is stateless by design and takes in API requests, processes them and stores the result in etcd if needed, and then returns the result of the request.
The API server is where most of the magic happens. It is stateless by design and takes in API requests, processes them and stores the result in etcd if needed, and then returns the result of the request.

We're going to use a unique feature of the kubelet to launch a Pod that runs the API server. Above we configured the kubelet to watch a local directory for pods to run with the --pod-manifest-path=/etc/kubernetes/manifests flag. All we need to do is place our Pod manifest in that location, and the kubelet will make sure it stays running, just as if the Pod was submitted via the API. The cool trick here is that we don't have an API running yet, but the Pod will function the exact same way, which simplifies troubleshooting later on.

If this is your first time looking at a Pod manifest, don't worry, they aren't all this complicated. But, this shows off the power and flexibility of the Pod concept. Create /etc/kubernetes/manifests/kube-apiserver.yaml with the following settings:

#### /etc/kubernetes/manifests/kube-apiserver.yaml
```
cat >  /etc/kubernetes/manifests/kube-apiserver.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
    command:
    - /hyperkube
    - apiserver
    - --bind-address=0.0.0.0
    - --apiserver-count=3
    - --audit-log-maxage=30
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=100
    - --audit-log-path=/var/lib/audit.log
    - --enable-swagger-ui=true
    - --authorization-mode=Node,RBAC
    - --etcd-cafile=/etc/kubernetes/ssl/ca.pem
    - --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem
    - --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem
    - --etcd-servers=https://141.22.30.32:2379,https://141.22.30.33:2379,https://141.22.30.34:2379,https://141.22.30.35:2379,https://141.22.30.36:2379
    - --allow-privileged=true
    - --service-cluster-ip-range=10.32.0.0/24
    - --secure-port=443
    - --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction
    - --kubelet-certificate-authority=/etc/kubernetes/ssl/ca.pem
    - --kubelet-client-certificate=/etc/kubernetes/ssl/apiserver.pem
    - --kubelet-client-key=/etc/kubernetes/ssl/apiserver-key.pem
    - --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem
    - --tls-ca-file=/etc/kubernetes/ssl/ca.pem
    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
    - --authentication-token-webhook-config-file=/etc/kubernetes/ssl/ldap-webhook-config.yaml
    - --authentication-token-webhook-cache-ttl=30m0s
    - --runtime-config=authentication.k8s.io/v1beta1=true
    - --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true
    - --token-auth-file=/etc/kubernetes/ssl/token.csv
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        port: 8080
        path: /healthz
      initialDelaySeconds: 15
      timeoutSeconds: 15
    ports:
    - containerPort: 443
      hostPort: 443
      name: https
    - containerPort: 8080
      hostPort: 8080
      name: local
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /var/lib/kubernetes/
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host


EOF
```

Note that we specify the --tls-cert-file and --tls-private-key-file parameters here.
The Kubelet has the same certificates specified which will allow Kubelet and apiserver
to communicate seemlessly.

Not also how we provide the etcd certificates and all nodes of the already set up
etcd cluster.

### Kubernetes Controller Manager

The controller manager is responsible for reconciling any required actions based on changes to Replication Controllers.

For example, if you increased the replica count, the controller manager would generate a scale up event, which would cause a new Pod to get scheduled in the cluster. The controller manager communicates with the API to submit these events.

Create /etc/kubernetes/manifests/kube-controller-manager.yaml. It will use the TLS certificate placed on disk earlier.

#### /etc/kubernetes/manifests/kube-controller-manager.yaml
```
cat > /etc/kubernetes/manifests/kube-controller-manager.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
    command:
    - /hyperkube
    - controller-manager
    - --master=http://127.0.0.1:8080
    - --allocate-node-cidrs=true
    - --cluster-cidr=10.200.0.0/16
    - --leader-elect=true
    - --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem
    - --root-ca-file=/etc/kubernetes/ssl/ca.pem
    - --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem
    - --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem
    - --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true
    resources:
      requests:
        cpu: 200m
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
      initialDelaySeconds: 15
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /var/lib/kubernetes/
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host

EOF
```


### Set Up the kube-scheduler Pod
The scheduler monitors the API for unscheduled pods, finds them a machine to run on, and communicates the decision back to the API.

Create File /etc/kubernetes/manifests/kube-scheduler.yaml:

#### /etc/kubernetes/manifests/kube-scheduler.yaml
```
cat > /etc/kubernetes/manifests/kube-scheduler.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
    command:
    - /hyperkube
    - scheduler
    - --master=http://127.0.0.1:8080
    - --leader-elect=true
    resources:
      requests:
        cpu: 100m
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
      initialDelaySeconds: 15
      timeoutSeconds: 15
EOF
```
### Start Services

Now that we've defined all of our units and written our TLS certificates to disk, we're ready to start the master components.

#### Load Changed Units
First, we need to tell systemd that we've changed units on disk and it needs to rescan everything:

```
$ sudo systemctl daemon-reload
```

#### Start kubelet
Now that everything is configured, we can start the kubelet, which will also start the Pod manifests for the API server, the controller manager, proxy and scheduler.
```
$ sudo systemctl start kubelet
```
Ensure that the kubelet will start after a reboot:
```
$ sudo systemctl enable kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.
```

### Verification

```
kubectl get componentstatuses
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}
etcd-3               Healthy   {"health": "true"}
etcd-4               Healthy   {"health": "true"}
```

> Note: currently the etcd nodes will report "remote error: tls: bad certificate". This is a known issue:
> see: [issue 156 on Kelseys Version of this guide](https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/156) and
> a [PR to fix it](https://github.com/kubernetes/kubernetes/pull/39716), that will be included in K8s 1.7

> Remember to run these steps on `controller0`, `controller1`, and `controller2`

### Set Up Kube Proxy

We're going to run the proxy as a Daemon Set in the finished cluster.
The proxy is responsible for directing traffic destined for specific services and pods to the correct location. The proxy communicates with the API server periodically to keep up to date.

Both the master and worker nodes in your cluster will run the proxy.

All you have to do is to ```kubectl apply -f https://gitlab.informatik.haw-hamburg.de/icc/kubernetes-the-icc-way/raw/master/deployments/kube-proxy.yaml``` once your cluster is running.
This will deploy a Daemon Set for the kube-proxy resulting in one instance running on every node.
