# Bootstrapping a H/A etcd cluster

In this lab you will bootstrap a 5 node etcd cluster. The following virtual machines will be used:

* icc-etcd-1
* icc-etcd-2
* icc-etcd-3
* icc-etcd-4
* icc-etcd-5

## Why

All Kubernetes components are stateless which greatly simplifies managing a Kubernetes cluster. All state is stored
in etcd, which is a database and must be treated specially. To support this in production, the etcd cluster is run on a dedicated set of machines for the following reasons:

* The etcd lifecycle is not tied to Kubernetes. We should be able to upgrade etcd independently of Kubernetes.
* Scaling out etcd is different than scaling out the Kubernetes Control Plane.
* Prevent other applications from taking up resources (CPU, Memory, I/O) required by etcd.

However, all the e2e tested configurations within the Kubernetes development project, currently run etcd on the master nodes.

## Provision the etcd Cluster

Run the following commands on `icc-etcd-1`, `icc-etcd-2`, `icc-etcd-3`, `icc-etcd-4`, `icc-etcd-5`:

For execution in production, this ist supported by the appropriate puppet configuration.

### TLS Certificates

The TLS certificates created in the [Setting up a CA and TLS Cert Generation](02-certificate-authority.md) lab will be used to secure communication between the Kubernetes API server and the etcd cluster. The TLS certificates will also be used to limit access to the etcd cluster using TLS client authentication. Only clients with a TLS certificate signed by a trusted CA will be able to access the etcd cluster.

Copy the TLS certificates to the etcd configuration directory:

```
sudo mkdir -p /etc/etcd/
```

```
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

### Install the etcd binaries

Since the ICC is mainly based on machines running Ubuntu, a package is available for etcd. But since the ICC requires etcd > 3.0.0, Ubuntu starting with 17.04 is required.

```
apt-get install etcd
```

All etcd data is stored under the etcd data directory. In a production cluster the data directory should be backed by a persistent disk. This directory is created by installing the package.


### Set The Internal IP Address

The internal IP address will be used by etcd to serve client requests and communicate with other etcd peers.

```
INTERNAL_IP=$(facter networking.ip)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name:

```
ETCD_NAME=$(hostname -s)
```

The etcd server will be started and managed by systemd. An appropriate systemd unit file comes with the Ubuntu package. But a number of parameters need to be set in the appropriate defaults file:



```
cat > etcd.default <<EOF
ETCD_NAME="$(hostname -s)"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://<%= $internal_ip %>:2380"
ETCD_LISTEN_CLIENT_URLS="https://<%= $internal_ip %>:2379,https://localhost:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://<%= $internal_ip %>:2380"
ETCD_INITIAL_CLUSTER="icc-etcd-1=https://141.22.30.32:2380,icc-etcd-2=https://141.22.30.33:2380,icc-etcd-3=https://141.22.30.34:2380,icc-etcd-4=https://141.22.30.352380,icc-etcd-5=https://141.22.30.36:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-0"
ETCD_ADVERTISE_CLIENT_URLS="https://<%= $internal_ip %>:2379"
ETCD_CERT_FILE="/etc/etcd/kubernetes.pem"
ETCD_KEY_FILE="/etc/etcd/kubernetes-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/kubernetes.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/kubernetes-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ca.pem"
EOF
```

Once the etcd systemd unit file is ready, move it to the systemd system directory:

```
sudo mv etcd.default /etc/default/etcd
```

Start the etcd server:

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable etcd
```

```
sudo systemctl start etcd
```

```
sudo systemctl status etcd --no-pager
```

> Remember to run these steps on all five nodes of the etcd cluster

## Verification

Once all 5 etcd nodes have been bootstrapped verify the etcd cluster is healthy:

* On one of the controller nodes run the following command:

```
sudo etcdctl \
  --ca-file=/etc/etcd/ca.pem \
  --cert-file=/etc/etcd/etcd.pem \
  --key-file=/etc/etcd/etcd-key.pem \
  --insecure-transport=false \
  --endpoints=141.22.30.36:2379 \
  -w table member list
```

```
+------------------+---------+------------+---------------------------+---------------------------+
|        ID        | STATUS  |    NAME    |        PEER ADDRS         |       CLIENT ADDRS        |
+------------------+---------+------------+---------------------------+---------------------------+
| 3d0fa8ceae1b7a61 | started | icc-etcd-4 | https://141.22.30.35:2380 | https://141.22.30.35:2379 |
| 3d9ce7c381e7f003 | started | icc-etcd-3 | https://141.22.30.34:2380 | https://141.22.30.34:2379 |
| ca225d371e604f6c | started | icc-etcd-2 | https://141.22.30.33:2380 | https://141.22.30.33:2379 |
| f15071bb2bcc0510 | started | icc-etcd-1 | https://141.22.30.32:2380 | https://141.22.30.32:2379 |
| f65430d6c44d7a2a | started | icc-etcd-5 | https://141.22.30.36:2380 | https://141.22.30.36:2379 |
+------------------+---------+------------+---------------------------+---------------------------+
```
