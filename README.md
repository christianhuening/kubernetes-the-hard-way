# Kubernetes The HAW-Hamburg Way

This tutorial is the adpation of the original tutorial by Kelsey Hightower to in install a Kubernetes cluster by hand. It containes changes and annotations that specific to the way the Informatik Computer Cluster (ICC) is set up. Furthermore it will contain tags of those parts have been completed and those that are still pending further research and trials.


This tutorial is optimized for learning, which means taking the long route to help people understand each task required to bootstrap a Kubernetes cluster.

> The results of this tutorial should not be viewed as production ready, and may receive limited support from the community, but don't let that prevent you from learning!

## Target Audience

The target audience for this tutorial is someone planning to support a production Kubernetes cluster and wants to understand how everything fits together. After completing this tutorial I encourage you to automate away the manual steps presented in this guide.

## Cluster Details

* Kubernetes 1.7.2
* Docker 1.12.6
* etcd 3.1.4
* [CNI Based Networking](https://github.com/containernetworking/cni)
* Secure communication between all components (etcd, control plane, workers)
* Default Service Account and Secrets
* [RBAC authorization enabled](https://kubernetes.io/docs/admin/authorization)
* [TLS client certificate bootstrapping for kubelets](https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping)
* High-Availability DNS add-on
* [Logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
* [Monitoring]

### What's Missing

The resulting cluster will be missing the following features:

* Cloud Provider Integration

## Labs

This cluster's pet components are installed on a set of VMs provided by a VMWare-based infrastructure, while the worker nodes use a number
 of bare-metal hosts. Details can be found in the infrastructure document below.

* [Infrastructure Provisioning](docs/01-infrastructure-icc.md)
* [Setting up a CA and TLS Cert Generation](docs/02-certificate-authority.md)
* [Setting up a HAProxy to provide reliable IP](docs/02.5-haproxy.md)
* [Setting up TLS Client Bootstrap and RBAC Authentication](docs/03-auth-configs.md)
* [Configuring Nodes to use Private Docker Registry](docs/private-docker-registry.md)
* [Bootstrapping a H/A etcd cluster](docs/04-etcd.md)
* [Bootstrapping H/A Kubernetes Master Nodes](docs/05-kubernetes-controller.md)
* [Bootstrapping Kubernetes Workers](docs/06-kubernetes-worker.md)
* [Configuring the Kubernetes Client - Remote Access](docs/07-kubectl.md)
* [Managing the Container Network Routes](docs/08-network.md)
* [Deploying the Cluster DNS Add-on](docs/09-dns-addon.md)
* [Smoke Test](docs/10-smoke-test.md)
* [Cleaning Up](docs/11-cleanup.md)
* [Managing Access](docs/12-managing-access.md)
* [Cluster Monitoring](docs/13-cluster-monitoring.md)
