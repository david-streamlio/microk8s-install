# Observability Stack Installation Guide

--------
## Add helm repos

1. Instead of the using the builtin microk8s `monitoring` addon, you can use the community Helm charts for both Prometheus and Grafana.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts

```

## Install Prometheus

1. Create a K8s namespace called `monitor` 
2. Install Prometheus using Helm

```bash
kubectl create ns monitor
helm install prometheus prometheus-community/prometheus -n monitor --set alertmanager.enabled=false --set kube-state-metrics.enabled=false --set prometheus-pushgateway.enabled=false


NAME: prometheus
LAST DEPLOYED: Mon Nov 25 07:45:23 2024
NAMESPACE: monitor
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.monitor.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitor -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitor port-forward $POD_NAME 9090


#################################################################################
######   WARNING: Pod Security Policy has been disabled by default since    #####
######            it deprecated after k8s 1.25+. use                        #####
######            (index .Values "prometheus-node-exporter" "rbac"          #####
###### .          "pspEnabled") with (index .Values                         #####
######            "prometheus-node-exporter" "rbac" "pspAnnotations")       #####
######            in case you still need it.                                #####
#################################################################################


For more information on running Prometheus, visit:
https://prometheus.io/
```

## Install Grafana

1. Install the Grafana Chart into the `monitor` namespace

```bash
helm install grafana grafana/grafana -n monitor --set image.repository=streamnative/private-cloud-grafana --set image.tag=0.1.1

NAME: grafana
LAST DEPLOYED: Mon Nov 25 07:50:36 2024
NAMESPACE: monitor
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace monitor grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo


2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.monitor.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
     export POD_NAME=$(kubectl get pods --namespace monitor -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace monitor port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################
```

## Access the Grafana dashboards

1. Expose the Grafana service

```bash
kubectl expose svc grafana --name grafana-external -n monitor --port 3000 --target-port 3000 --type LoadBalancer
```

2. Get the Grafana password

```bash
kubectl get secret --namespace monitor grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

3. Access the Grafana Dashboard

```bash
kubectl get svc -n monitor --field-selector metadata.name=grafana-external
```

---------
## References

- https://docs.streamnative.io/private/private-cloud-monitor#install-monitoring-stacks
- https://medium.com/@gayatripawar401/deploy-prometheus-and-grafana-on-kubernetes-using-helm-5aa9d4fbae66