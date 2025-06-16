# MicroK8s Guide

MicroK8s is an open-source system for automating deployment, scaling, and management of containerised applications. 
It provides the functionality of core Kubernetes components, in a small footprint, scalable from a single node to a 
high-availability production cluster.

This is a collection of my notes for setting up MicroK8s in my Ubuntu-based home lab.

-------------------
## Prerequisites

- An Ubuntu 22.04 LTS, 20.04 LTS, 18.04 LTS or 16.04 LTS environment to run the commands (or another operating system which supports snapd - see the snapd documentation).
- A system with at least 20G of disk space and 4G of memory.
- An internet connection

-------------------
## Getting Started

MicroK8s is easy to install and use on Ubuntu or any Linux which supports snaps - see the 
[microk8s-installation-guide.md](docs%2Fmicrok8s-installation-guide.md) for detailed on the steps I used to install it 
on Ubuntu 22.04, including OS level changes, and using ETCD as the metadata store for H/A.

-------------------
## MicroK8s Clustering

Although MicroK8s is designed as an ultra-lightweight implementation of Kubernetes, it is still possible, and useful to 
create a MicroK8s cluster. My home lab consists of three Dell Precision servers, each running Ubuntu 22.04. 

- k8s-node00 (Dell Precision T5600 - 40 cores & 128 GB of RAM)
- k8s-node01 (Dell Precision T7200 - 56 cores & 512 GB of RAM)
- k8s-node02 (Dell Precision T7200 - 72 cores & 512 GB of RAM)

I then installed MicroK8s on each of them using the [microk8s-installation-guide.md](docs%2Fmicrok8s-installation-guide.md). 

In order to get them to act as a single K8s cluster, that would allow the K8s schedule to allocate resources from all 
three of my servers, I had to perform the following steps:

### Adding a node

To create a cluster out of two or more already-running MicroK8s instances, use the microk8s add-node command. The MicroK8s
instance on which this command is run will be the master of the cluster and will host the Kubernetes control plane:

On k8s-node00:  

```bash
microk8s add-node
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.0.80:25000/b83780a7497a862c2869cd69923c90a6/c0648301f8a2
```

On k8s-node01 & k8s-node02

```bash
microk8s join 192.168.0.80:25000/b83780a7497a862c2869cd69923c90a6/c0648301f8a2
```

Joining a node to the cluster should only take a few seconds. Afterward you should be able to see the nodes have 
joined the cluster by running the following command and getting a list with all 3 nodes in it as shown here:

```bash
kubectl get no
NAME                        STATUS   ROLES    AGE     VERSION
k8s-node00.kubernetes.net   Ready    <none>   4d18h   v1.29.10
k8s-node01.kubernetes.net   Ready    <none>   4d18h   v1.29.10
k8s-node02.kubernetes.net   Ready    <none>   4d18h   v1.29.10
```


-------------------
## MicroK8s Addons

To be as lightweight as possible, MicroK8s only installs the basics of a usable
Kubernetes install:

- api-server
- controller-manager
- scheduler 
- kubelet 
- cni 
- kube-proxy

While this does deliver a pure Kubernetes experience with the smallest resource footprint possible, there are situations
where you may require additional services. MicroK8s caters for this with the concept of “Addons” - extra services which 
can easily be added to MicroK8s. These addons can be enabled and disabled at any time, and most are pre-configured to 
‘just work’ without any further set up. Below are links to the steps I took to install the addons required for my cluster:


- [minio-installation.md](docs%2Fminio-installation.md)

-------------------
References
-------------------
1. https://microk8s.io/docs
2. https://microk8s.io/docs/addon-dashboard
