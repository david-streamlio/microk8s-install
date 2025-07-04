# Kubernetes Installation on Ubuntu 22.04 

## 1. Increase max virtual memory

```
echo "vm.max_map_count=1048576" >> /etc/sysctl.conf
sudo sysctl -p
```

-----------------
## 2. Enable cgroups
```
vi /etc/default/grub

GRUB_CMDLINE_LINUX="cgroup_enable=memory cgroup_memory=1 systemd.unified_cgroup_hierarchy=1"
sudo update-grub
```
     **Kernel reboot is required**

-----------------
## 3. Install the microk8s snap

Install the microk8s snap onto the Ubuntu machine using the following command.

`sudo snap install microk8s --channel 1.29/stable --classic`

-----------------
## 4. Install etcd to use as the backend storage mechanism for microk8s (CRITICAL)

Follow the [etcd installation guide](./docs/etcd-install.md) to create your etcd cluster.

-----------------
## 5. Configure microk8s to use etcd

`sudo vi /var/snap/microk8s/current/args/kube-apiserver`

Add the following line:

`--etcd-servers=http://<ETCD-NODE-0>:2379,http://<ETCD-NODE-1>:2379,....`

-----------------
## 6. Restart microk8s

```
sudo microk8s stop
sudo microk8s start
```

-----------------
## 7. Confirm that microk8s is using etcd now

  ```
  sudo microk8s status
     microk8s is running
       datastore endpoints:
         <ETCD-NODE-0>:2379
         <ETCD-NODE-1>:2379
         ...

  ```

-----------------
## 8. Configure Networking

Configure the Container Network Interface (CNI) used by MicroK8s. Applying network configuration settings, such as:

- The type of CNI plugin to use (e.g., flannel, calico, cilium).
- Network CIDR (IP range) for the pods.
- IP allocation settings.
- Any specific settings related to the CNI plugin being used.

```sudo microk8s kubectl apply -f /var/snap/microk8s/current/args/cni-network/cni.yaml```

If you are having network issues in MicroK8s (e.g., pods cannot communicate), inspect the logs of the CNI pods

```microk8s kubectl logs -n kube-system <cni-pod-name>```

-----------------
## 9. Stop the k8s-dqlite service (optional)

The K8s dqlite service is used to store information about the K8s cluster. However, since we are
using etcd for that purpose, it is safe to disable this service as it serves no purpose at this point.

`sudo systemctl stop snap.microk8s.daemon-k8s-dqlite.service`

-----------------
## 10. Enable microk8s services
For microk8s to work as needed, we need to enable a few services using the following command:

```
microk8s enable hostpath-storage dns ingress dashboard
```

- [HostPath](https://microk8s.io/docs/addon-hostpath-storage) is a storage provisioner in MicroK8s. `hostPath` is a type
  of persistent storage in Kubernetes that maps a directory on the host machine (where the Kubernetes node is running) 
  to a directory inside a pod. This type of storage is useful for local development or testing when you don't need a 
  distributed storage solution like NFS, Ceph, or cloud-based storage (e.g., AWS EBS, GCP PD).


- [CoreDNS](https://microk8s.io/docs/addon-dns) is a flexible, extensible DNS server that can serve as the DNS service 
  for Kubernetes clusters. It helps resolve DNS names within the cluster.


- [Ingress](https://microk8s.io/docs/addon-ingress): This addon adds an NGINX Ingress Controller for MicroK8s, which can
  be used for load-balancing in front of service endpoints.
  

- [Dashboard](https://microk8s.io/docs/addon-dashboard): This addon enables the standard K8s dashboard, which is a 
  convenient way to keep track of the activity and resource use of MicroK8s.

-----------------
## 11. Add and set new storage classes for SSD and NVME

For my cluster, I have created two RAID pools. One comprised of 4 SSD drives per machine, and another comprised of 2 NVMe drives per machine. In order to specify the disk types I want to use inside my K8s deployments, I must first create `storage class` definitions for each of these.

Once those definition are applied, you can also override the default storage class so that all K8s mananged PVC arebacked by SSD storage by default.

```
kubectl apply -f ./configs/ssd-raid-sc.yaml
kubectl apply -f ./configs/nvme-raid-sc.yaml

kubectl patch storageclass  ssd-raid -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass  microk8s-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

-----------------
## 12. Enable microk8s Container Registry

This must be done after the default storage class is changed to ensure that the registry uses a PV created from the ssd-raid
pool vs. the root partition. It is also important to perform this step **BEFORE** you install any software on microk8s that
requires an image download, e.g. `microk8s enable cert-manager`, otherwise there won't be a place to store the images that
are downloaded from the internet and the installation will hang.

```
microk8s enable registry:size=300Gi    # Specify whatever size you like.
```

-----------------
## 13. Enable the microk8s load balancer

This allows microk8s to assign static IPs on your internal router network so that they are publicly accessible inside
your network

`microk8s enable metallb`
(Enter 192.168.0.200-192.168.0.240 for the IP range) this will give you a pool of 40 IP addresses that can be used to 
expose services running inside microk8s.

After you created the address pool, you need to configure the L2Advertisement this will allow that the ip addresses of 
the pool answer to ARP requests, because the ip addresses are not associated to any interface, to do this using the 
following command

```
microk8s kubectl apply -f configs/advertisement.yaml
```

-----------------
## 14. Create an alias for microk8s.kubectl
```
echo "alias kubectl='microk8s.kubectl'" > ~/.bash_aliases
```

## 15. Expose the K8s Dashboard service 

Next, we are going to enable the Kubernetes dashboard and expose it using a friendly name using an ingress controller. 
This allows us to access the dashboard using an easy to remember DNS name rather than using port forwarding from the pod.
To do this, we need to run the following command to create the ingress controller for the dashboard pod and associate it
with the IP address of `192.168.0.201`

```
kubectl apply -f configs/k8s-dashboard-service.yaml
```

-----------------
## 16. Get the secret token used to log into the dashboard
```
token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s kubectl -n kube-system describe secret $token
```

You can then access the Dashboard at https://127.0.0.1:10443

![K8s-Dashboard.png](..%2Fimages%2FK8s-Dashboard.png)


-------------------
References
-------------------
1. https://microk8s.io/docs/getting-started
2. https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/
3. https://microk8s.io/docs/addon-dashboard
4. https://github.com/canonical/microk8s/issues/463
5. https://askubuntu.com/questions/1237813/enabling-memory-cgroup-in-ubuntu-20-04
