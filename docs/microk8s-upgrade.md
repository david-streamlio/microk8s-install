# Upgrading MicroK8s on Ubuntu 22.04

If you followed the installation steps in the [README](../README.md) file, then you installed version 1.25 of the MicroK8s software on
your machine(s). This guide covers the process of upgrading MicroK8s to a newer version and covers some "gotchas" and gaps
in the official [documentation](https://microk8s.io/docs/upgrading).

## Single Node Cluster

Refreshing the MicroK8s snap to the desired channel effectively upgrades the cluster. For instance, to upgrade a 
v1.25 cluster to v1.27 simply run `sudo snap refresh microk8s --channel=1.27/stable`. 

## Multi-Node Cluster

For a multi-node cluster the process is a bit more complicated, as you must repeat the following steps for **_EACH_** of the nodes
in the cluster 1-by-1:

1. Drain all running K8s pods on the node

```
kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data
```

The official documentation doesn't mention the `-delete-emptydir-data` switch, but it is needed if you have a Pulsar cluster
deployment running on MicroK8s. During the eviction process, you might encounter some pods that cannot be drained due 
to "disruption budget", as shown here:

```
evicting pod pulsar/zookeepers-zk-0
error when evicting pods/"zookeepers-zk-0" -n "pulsar" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
```

2. Delete any pods that won't drain. (Optional)

In order to evict these problematic pods, you will have to delete each of them using the `kubectl delete pod` command, e.g.:

`kubectl -n pulsar delete pod zookeeeprs-zk-0`

If the kubectl delete command "hangs", you can remedy this with the following command:

`kubectl -n pulsar patch pod zookeepers-zk-0 -p '{"metadata":{"finalizers":null}}'`

3. Validate that all pods have been evicted using `kubectl get po -A -o wide`

4. Update the MicroK8s version with the following command: `udo snap refresh microk8s --channel=1.27/stable`

5. After the new version has been fetched and the snap is updated, the node should register with the new version

```
kubectl get nodes

NAME         STATUS   ROLES    AGE   VERSION
k8s-node00   Ready    <none>   16d   v1.27.7
k8s-node01   Ready    <none>   17d   v1.25.15
```

6. To gain access to addons that weren't in the previous version of MicroK8s, you will need to update the addon repository
with the following command: `sudo microk8s addons repo update core`

These steps should be repeated on all the nodes in the cluster.
