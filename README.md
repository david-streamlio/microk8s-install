# Kubernetes Installation on Ubuntu 22.04 

## 1. Increase max virtual memory

```
echo "vm.max_map_count=1048576" >> /etc/sysctl.conf
sudo sysctl -p
```

## 2. Enable cgroups
```
vi /etc/default/grub

GRUB_CMDLINE_LINUX="cgroup_enable=memory cgroup_memory=1 systemd.unified_cgroup_hierarchy=1"
sudo update-grub
```
     **Kernel reboot is required**

## 3. Install the microk8s snap
`sudo snap install microk8s --channel 1.27/stable --classic`

## 4. Install etcd to use as the backend storage mechanism for microk8s

Follow the [etcd installation guide](./docs/etcd-install.md) to create your etcd cluster

## 5. Configure microk8s to use etcd

`sudo vi /var/snap/microk8s/current/args/kube-apiserver`

Add the following line:

`--etcd-servers=http://<ETCD-NODE-0>:2379,http://<ETCD-NODE-1>:2379,....`

## 6. Restart microk8s

```
sudo microk8s stop
sudo microk8s start
```

## 7. Confirm that microk8s is using etcd now

  ```
  sudo microk8s status
     microk8s is running
       datastore endpoints:
         <ETCD-NODE-0>:2379
         <ETCD-NODE-1>:2379

  ```

## 8. Stop the k8s-dqlite service (optional)

`sudo systemctl stop snap.microk8s.daemon-k8s-dqlite.service`

## 9. Enable microk8s services
```
microk8s enable dashboard dns hostpath-storage
```

## 10. Add and set new storage classes for SSD and NVME
```
kubectl apply -f ./configs/ssd-raid-sc.yaml
kubectl apply -f ./configs/nvme-raid-sc.yaml

kubectl patch storageclass  ssd-raid -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass  microk8s-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

## 11. Enable microk8s Container Registry

This must be done after the default storage class is changed to ensure that the registry uses a PV created from the ssd-raid
pool vs. the root partition. It is also important to perform this step **BEFORE** you install any software on microk8s that
requires an image download, e.g. `microk8s install cert-manager`, otherwise there won't be a place to store the images that
are downloaded from the internet and the installation will hang.

```
microk8s enable registry:size=200Gi    # Specify whatever size you like.
```

## 12. Enable the microk8s load balancer

This allows microk8s to assign static IPs on your internal router network so that they are publicly accessible inside
your network

`microk8s enable metallb`
(Enter 192.168.1.120-192.168.1.140 for the IP range) this will give you a pool of 20 IP addresses that can be used to 
expose services running inside microk8s.


## 13. Create an alias for microk8s.kubectl
```
echo "alias kubectl='microk8s.kubectl'" > ~/.bash_aliases
```

## 14. Get the kube config file, set it to default, and verify
```
cp /var/snap/microk8s/current/credentials/client.config ~/.kube/microk8s.config
export KUBECONFIG=~/.kube/microk8s.config
kubectl get nodes
```

## 15. Get the secret token used to log into the dashboard
```
token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s kubectl -n kube-system describe secret $token

microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443 &
```

You can then access the Dashboard at https://127.0.0.1:10443

![K8s-Dashbaord.png](images%2FK8s-Dashbaord.png)


-------------------
References
-------------------
1. https://microk8s.io/docs/getting-started
2. https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/
3. https://microk8s.io/docs/addon-dashboard
4. https://github.com/canonical/microk8s/issues/463
5. https://askubuntu.com/questions/1237813/enabling-memory-cgroup-in-ubuntu-20-04
