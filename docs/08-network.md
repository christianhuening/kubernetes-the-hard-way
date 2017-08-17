# Managing the Container Network Routes

As the nodes of the ICC are all positioned within the same layer 3 network, there are no complex routing rules to consider. This may change eventually as the cluster grows to consume the almost 500 available IP addresses within that network. But even for that case the ICC comes prepared, as the VXLAN overlay network used in flannel, can be routed to other networks just like any other Layer 4 package.

In order separate the networks between sets of pods, namespaces and applications, the ICC uses the Calico network policy framework. Together with flannel it forms [Canal](https://github.com/projectcalico/canal) which is deployed below.


## Preparing the Kubelets for CNI ##
Requirements:

- The Kubernetes cluster must be configured to provide serviceaccount tokens to pods.
- kubelets must be started with --network-plugin=cni and have --cni-conf-dir and --cni-bin-dir properly set
- The controller manager must be started with --cluster-cidr=10.244.0.0/16 and --allocate-node-cidrs=true.

The following steps below have to be performed for each of the nodes `icc-node-1`, `icc-node-2` and `icc-node-3`:


## Applying the Network ##

To set up the neccesary roles and permission, the ConfigMap provided by the Canal project will work without changes.

```
kubectl apply -f ./network/rbac_rules.yaml
```

The actual setup of the network elements needs some small changes, mainly due to the changed CIDR for the pod networks (`10.200.0.0/16` instead of the `10.244.0.0/16` used by the original configuration.

```
kubectl apply -f ./network/canal_fabric.yaml
```

This will set up a network in each node and configure it to use flannel and calico.

## Hint to Tolerations
In case you're using the Taints and Tolerations Beta Feature (https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#taints-and-tolerations-beta-feature)
and have nodes with taints, you most likely want to add a Toleration to your Canal DaemonSet deployment in order to allow
it to also run the Canal Pods on tainted nodes.

## Check the Network Adresses ##

```
kubectl get nodes \
  --output=jsonpath='{range .items[*]}{.status.addresses[?(@.type=="InternalIP")].address} {.spec.podCIDR} {"\n"}{end}'
```

shoud provide a list similar to this:

```
141.22.10.209 10.200.2.0/24
141.22.10.214 10.200.1.0/24
141.22.10.216 10.200.0.0/24
```

