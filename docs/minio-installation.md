# Minio Installation Guide for Microk8s

MinIO is a well-known and established project in the CNCF ecosystem that provides cloud-agnostic S3-compatible object 
storage. It is free, open-source and well-trusted by multiple organizations.

The minio addon can be used to deploy MinIO on a MicroK8s cluster using minio-operator, as well as the kubectl-minio 
CLI tool for managing the deployment. 

## Enablement

1. For single-node development clusters, enable the MinIO add-on with a single command

```
sudo microk8s enable minio -c 300Gi -s ssd-raid

Initialize minio operator
namespace/minio-operator created
serviceaccount/minio-operator created
clusterrole.rbac.authorization.k8s.io/minio-operator-role created
clusterrolebinding.rbac.authorization.k8s.io/minio-operator-binding created
customresourcedefinition.apiextensions.k8s.io/tenants.minio.min.io created
service/operator created
deployment.apps/minio-operator created
serviceaccount/console-sa created
secret/console-sa-secret created
clusterrole.rbac.authorization.k8s.io/console-sa-role created
clusterrolebinding.rbac.authorization.k8s.io/console-sa-binding created
configmap/console-env created
service/console created
deployment.apps/console created
-----------------

To open Operator UI, start a port forward using this command:

kubectl minio proxy -n minio-operator 

-----------------
Create default tenant with:

  Name: microk8s
  Capacity: 300Gi
  Servers: 1
  Volumes: 1
  Storage class: ssd-raid
  TLS: no
  Prometheus: no
  
...
Tenant 'microk8s' created in 'minio-operator' Namespace

  Username: 9HIGOKULGVRWAX027UI4 
  Password: m3n0GibzfCryZ5RiSeLJb2LGNtRmynfvzGhH6FiU 
  Note: Copy the credentials to a secure location. MinIO will not display these again.

```

After deployment, the output prints the generated username and password. Be sure to copy the credentials to a secure 
location, as you will need them to administer the default Minio tenant. 

2. Validate the operators are installed properly

```
kubectl -n minio-operator get all

NAME                                  READY   STATUS    RESTARTS   AGE
pod/minio-operator-58f7f898df-f7kw7   1/1     Running   0          4m25s
pod/console-5cbc7f5744-lhrx7          1/1     Running   0          4m25s
pod/minio-operator-58f7f898df-4q7rr   1/1     Running   0          4m25s
pod/microk8s-ss-0-0                   1/1     Running   0          3m42s

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/operator           ClusterIP   10.152.183.232   <none>        4222/TCP,4221/TCP   4m25s
service/console            ClusterIP   10.152.183.213   <none>        9090/TCP,9443/TCP   4m25s
service/minio              ClusterIP   10.152.183.70    <none>        80/TCP              3m44s
service/microk8s-console   ClusterIP   10.152.183.75    <none>        9090/TCP            3m43s
service/microk8s-hl        ClusterIP   None             <none>        9000/TCP            3m43s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/console          1/1     1            1           4m25s
deployment.apps/minio-operator   2/2     2            2           4m25s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/console-5cbc7f5744          1         1         1       4m25s
replicaset.apps/minio-operator-58f7f898df   2         2         2       4m25s

NAME                             READY   AGE
statefulset.apps/microk8s-ss-0   1/1     3m42s
```

3. Access the Minio console

You can use the supplied username and password to access and manage the tenant using the MinIO console. Create a 
port-forward for the MinIO console with the following command, and follow the instructions:

`sudo microk8s kubectl-minio proxy`

This command will output a JWT token that can be used to authenticate to the console on http://localhost:9090 where you 
should see a screen like the following.


![Minio-Console.png](..%2Fimages%2FMinio-Console.png)