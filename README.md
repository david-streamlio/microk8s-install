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
`sudo snap install microk8s --channel 1.25/stable --classic`

## 4. Add and set new storage classes for SSD and NVME
```
kubectl apply -f ./configs/ssd-raid-sc.yaml
kubectl apply -f ./configs/nvme-raid-sc.yaml

kubectl patch storageclass  ssd-raid -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass  microk8s-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

## 5. Enable microk8s services
```
microk8s enable cert-manager dashboard dns helm hostpath-storage metrics-server
microk8s enable registry:size=200Gi    # Specify whatever size you like.
```

## 6. Enable the microk8s load balancer
`microk8s enable metallb`
(Enter 192.168.1.120-192.168.1.130 for the IP range)


## 7. Create an alias for microk8s.kubectl
```
echo "alias kubectl='microk8s.kubectl'" > ~/.bash_aliases
```

## 8. Get the kube config file, set it to default, and verify
```
cp /var/snap/microk8s/current/credentials/client.config ~/.kube/microk8s.config
export KUBECONFIG=~/.kube/microk8s.config
kubectl get nodes
```


## 9. Get the secret token used to log into the dashboard
```
token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s kubectl -n kube-system describe secret $token

microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443 &
```

You can then access the the Dashboard at https://127.0.0.1:10443

![K8s-Dashbaord.png](images%2FK8s-Dashbaord.png)


-------------------
References
-------------------
1. https://microk8s.io/docs/getting-started
2. https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/
3. https://microk8s.io/docs/addon-dashboard
4. https://github.com/canonical/microk8s/issues/463
5. https://askubuntu.com/questions/1237813/enabling-memory-cgroup-in-ubuntu-20-04
