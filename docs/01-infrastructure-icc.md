# Cloud Supporting Infrastructure Provisioning - 

This guide will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 1+3+5+n machines will be created.
Also an arbitrary number of runner machines will be set up.

After completing this guide you should have the following virtual machines:


````
NAME         INTERNAL_IP  EXTERNAL_IP      STATUS
icc-frontend-1  ---       141.22.10.222    RUNNING

icc-etcd-1      ---       141.22.30.32     RUNNING
icc-etcd-2      ---       141.22.30.33     RUNNING
icc-etcd-3      ---       141.22.30.34     RUNNING
icc-etcd-4      ---       141.22.30.35     RUNNING
icc-etcd-5      ---       141.22.30.36     RUNNING

icc-ctrl-1   10.240.0.10  XXX.XXX.XXX.XXX  RUNNING
icc-ctrl-2   10.240.0.11  XXX.XXX.XXX.XXX  RUNNING
icc-ctrl-3   10.240.0.12  XXX.XXX.XXX.XXX  RUNNING

icc-worker-1 10.240.0.20  XXX.XXX.XXX.XXX  RUNNING
icc-worker-2 10.240.0.21  XXX.XXX.XXX.XXX  RUNNING
icc-worker-3 10.240.0.22  XXX.XXX.XXX.XXX  RUNNING
````

> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

To make our Kubernetes control plane remotely accessible, a public IP address will be provisioned and assigned to a Load Balancer that will sit in front of the 3 Kubernetes controllers.

## Prerequisites

Create all machines in the Informatik vsphere, set them up to use puppet.

## Setup Networking

All machines of the ICC, except the members of the etcd-cluster must be set up within the FuL network.

Assure proper permissions are set in the ACL:
 * access from controllers to etcd-cluster
 * access from controllers to puppet master


### Create the Kubernetes Public Address

Create a public IP address that will be used by remote clients to connect to the Kubernetes control plane:

This is set to `141.22.10.222` and set in DNS as `icc-k8s-api.informatik.haw-hamburg.de`

Information the HAProxy load balancer that will enable reaching all nodes on the same IP can be found in the [chapter on the HAProxy setup](/docs/02.5-haproxy.md)

## Provision Virtual Machines

All the VMs in this lab will be provisioned using Ubuntu 16.04 mainly because it runs a newish Linux kernel with good support for Docker.

The controlers should have 4 GByte of RAM.

