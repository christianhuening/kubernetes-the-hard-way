# Setting up a Certificate Authority and Creating TLS Certificates

In this lab you will setup the necessary PKI infrastructure to secure the Kubernetes components. This lab will leverage CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), to bootstrap a Certificate Authority and generate TLS certificates to secure the following Kubernetes components:

* etcd
* kube-apiserver
* kubelet
* kube-proxy

After completing this lab you should have the following TLS keys and certificates:

```
admin.pem
admin-key.pem
ca-key.pem
ca.pem
etcd-key.pem
etcd.pem
apiserver-key.pem
apiserver.pem
kubernetes-key.pem
kubernetes.pem
kube-proxy.pem
kube-proxy-key.pem
```

## Install CFSSL

This lab requires the `cfssl` and `cfssljson` binaries. Download them from the [cfssl repository](https://pkg.cfssl.org).

### OS X

```
wget https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
chmod +x cfssl_darwin-amd64
sudo mv cfssl_darwin-amd64 /usr/local/bin/cfssl
```

```
wget https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64
chmod +x cfssljson_darwin-amd64
sudo mv cfssljson_darwin-amd64 /usr/local/bin/cfssljson
```

### Linux

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
```

```
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

## Set up a Certificate Authority

Create a CA configuration file:

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Create a CA certificate signing request:


```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "DE",
      "L": "Hamburg",
      "O": "NAME OF YOUR CLOUD SOLUTION",
      "OU": "CA",
      "ST": "Hamburg"
    }
  ]
}
EOF
```

Generate a CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
ca-key.pem
ca.pem
```

## Generate client and server TLS certificates

In this section we will generate TLS certificates for each Kubernetes component and a client certificate for the admin user.

### Create the Admin client certificate

Create the admin client certificate signing request:

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "DE",
      "L": "Hamburg",
      "O": "system:masters",
      "OU": "Cluster",
      "ST": "Hamburg"
    }
  ]
}
EOF
```

Generate the admin client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Results:

```
admin-key.pem
admin.pem
```

### Create the kube-proxy client certificate

Create the kube-proxy client certificate signing request:

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "DE",
      "L": "Hamburg",
      "O": "system:node-proxier",
      "OU": "Cluster",
      "ST": "Hamburg"
    }
  ]
}
EOF
```

Generate the kube-proxy client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

Results:

```
kube-proxy-key.pem
kube-proxy.pem
```

### Create the Kubernetes API-Server certificate

The Kubernetes public IP address will be included in the list of subject alternative names for the Kubernetes server certificate. This will ensure the TLS certificate is valid for remote client access.


Create the Kubernetes API-Server certificate signing request:

```yaml
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "10.32.0.1",
    "10.240.0.10",
    "10.240.0.11",
    "10.240.0.12",
    "YOUR_API_LOADBALANCER_NODE_IP",
    "YOUR_API_SERVER_NODE_IP_1",
    "YOUR_API_SERVER_NODE_IP_1",
    "YOUR_API_SERVER_NODE_IP_1",
    "YOUR_API_SERVER_NODE_HOSTNAME_1",
    "YOUR_API_SERVER_NODE_HOSTNAME_2",
    "YOUR_API_SERVER_NODE_HOSTNAME_3,
    "127.0.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "DE",
      "L": "Hamburg",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "Hamburg"
    }
  ]
}

EOF
```

Generate the Kubernetes certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Results:

```
kubernetes-key.pem
kubernetes.pem
```

### Create the etcd server certificate

The IP addresses of each of the etcd nodes will be included in the list of subject alternative names for the etcd server certificate. This will ensure the TLS certificate is valid for remote client access.

The certificate needs to include each of the names of the etcd peers

```
KUBERNETES_PUBLIC_ADDRESS=$(dig +short icc-k8s-api.informatik.haw-hamburg.de)
```

Create the etcd server certificate signing request:

```yaml
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "YOUR_ETCD_SERVER_NODE_HOSTNAME_1",
    "YOUR_ETCD_SERVER_NODE_HOSTNAME_2",
    "YOUR_ETCD_SERVER_NODE_HOSTNAME_3",
    "YOUR_ETCD_SERVER_NODE_HOSTNAME_4",
    "YOUR_ETCD_SERVER_NODE_HOSTNAME_5"
  ],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "DE",
      "L": "Hamburg",
      "O": "Etcd",
      "OU": "Cluster",
      "ST": "Hamburg"
    }
  ]
}
EOF
```

Generate the etcd certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd
```

Results:

```
etcd-key.pem
etcd.pem
```

