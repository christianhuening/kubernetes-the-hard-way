# Cloud Supporting Infrastructure Provisioning - 


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

This is set to `IP_OF_YOUR_LB` and set in DNS as `HOSTNAME`

Information the HAProxy load balancer that will enable reaching all nodes on the same IP can be found in the [chapter on the HAProxy setup](/docs/02.5-haproxy.md)

## Provision Virtual Machines

All the VMs in this lab will be provisioned using Ubuntu 16.04 mainly because it runs a newish Linux kernel with good support for Docker.

The controlers should have 4 GByte of RAM.

