# Managing the Container Network Routes

As the nodes of the ICC are all positioned within the same layer 3 network, there are no complex routing rules to consider. This may change eventually as the cluster grows to consume the almost 500 available IP addresses within that network. But even for that case the ICC comes prepared, as the VXLAN overlay network used in flannel, can be routed to other networks just like any other Layer 4 package.

In order separate the networks between sets of pods, namespaces and applications, the ICC uses the Calico network policy framework. Together with flannel it forms [Canal](https://github.com/projectcalico/canal) which is deployed below.


## Preparing the Kubelets for CNI ##
Requirements:

- The Kubernetes cluster must be configured to provide serviceaccount tokens to pods.
- kubelets must be started with --network-plugin=cni and have --cni-conf-dir and --cni-bin-dir properly set
- The controller manager must be started with --cluster-cidr=10.244.0.0/16 and --allocate-node-cidrs=true.

The following steps below have to be performed for each of the nodes `icc-node-1`, `icc-node-2` and `icc-node-3`:

Switch the network plugin from `kubenet` to `cni` by changing the kubelet systemd service definition:

```
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/kubelet \\
  --api-servers=https://${PUBLIC_K8S_ADDRESS}:443 \\
  --allow-privileged=true \\
  --cluster-dns=10.32.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=docker \\
  --experimental-bootstrap-kubeconfig=/var/lib/kubelet/bootstrap.kubeconfig \\
  --network-plugin=cni \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --serialize-image-pulls=false \\
  --register-node=true \\
  --tls-cert-file=/var/lib/kubelet/kubelet-client.crt \\
  --tls-private-key-file=/var/lib/kubelet/kubelet-client.key \\
  --cert-dir=/var/lib/kubelet \\
  --v=2
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


## Applying the Network ##

To set up the neccesary roles and permission, the ConfigMap provided by the Canal project will work without changes.

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/canal/master/k8s-install/1.6/rbac.yaml
```

The actual setup of the network elements needs some small changes, mainly due to the changed CIDR for the pod networks (`10.200.0.0/16` instead of the `10.244.0.0/16` used by the original confguration (KTIW_ROOT is the top directory of this document).

```
kubectl apply -f ${KTIW_ROOT}/config_maps/canal_fabric.yaml
```

This will set up a network in each node and configure it to use flannel and calico.

