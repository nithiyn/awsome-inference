# Steps to set up cluster

In this guide we will set up the Kubernetes cluster for the deployment of LLMs using Triton Server and TRT-LLM. 
* 
## 1. Add node label and taint

As first step we will add node labels and taints

* A node label of `nvidia.com/gpu=present` to more easily identify nodes with NVIDIA GPUs.
* A node taint of `nvidia.com/gpu=present:NoSchedule` to prevent non-GPU pods from being deployed to GPU nodes.

Run the following command to get nodes with instance type:

```
kubectl get nodes -L node.kubernetes.io/instance-type
```

You should see output something similar to below:

```
NAME                          STATUS   ROLES    AGE     VERSION    INSTANCE-TYPE
ip-192-168-117-30.ec2.internal   Ready    <none>   3h10m   v1.30.2-eks-1552ad0    p5.48xlarge 
ip-192-168-127-31.ec2.internal   Ready    <none>   155m    v1.30.2-eks-1552ad0    c5.2xlarge
ip-192-168-26-106.ec2.internal   Ready    <none>   3h23m   v1.30.2-eks-1552ad0    p5.48xlarge
```

> [!Note]
> Here we have 3 nodes: 1 CPU node and 2 GPU nodes. You only need to apply labels and taints to GPU nodes. Note that because EFA is enabled on GPU nodes, they have to be in the same subnet in your VPC. Thus, their IP addresses are closer. In this case, the top 2 nodes are likely to be the GPU nodes. You can also run `kubectl describe node <node_name>` to verify.

Run the following command to add label and taints:

```
# Add label
kubectl label nodes $(kubectl get nodes -L node.kubernetes.io/instance-type | grep p5 | cut -f 1 -d ‘ ‘) nvidia.com/gpu=present

# Add taint
kubectl taint nodes $(kubectl get nodes -L node.kubernetes.io/instance-type | grep p5 | cut -f 1 -d ‘ ‘) nvidia.com/gpu=present:NoSchedule
```

Alternatively, you can add labels and taints in node groups under [EKS console](https://console.aws.amazon.com/eks/home).

## 2. Install Kubernetes Node Feature Discovery service

This allows for the deployment of a pod onto a node with a matching taint that we set above.

```
kubectl create namespace monitoring
helm repo add kube-nfd https://kubernetes-sigs.github.io/node-feature-discovery/charts && helm repo update
helm install -n kube-system node-feature-discovery kube-nfd/node-feature-discovery \
  --set nameOverride=node-feature-discovery \
  --set worker.tolerations[0].key=nvidia.com/gpu \
  --set worker.tolerations[0].operator=Exists \
  --set worker.tolerations[0].effect=NoSchedule
```

## 3. Install NVIDIA Device Plugin

We are using NVIDIA Device Plugin here because the default EKS optimzied AMI (Amazon Linux 2) already has NVIDIA drivers pre-installed. If you would like to use EKS Ubuntu AMI which does not have the drivers pre-installed, you need to install [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/amazon-eks.html#nvidia-gpu-operator-with-amazon-eks) instead.

```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml
```

## 4. Install NVIDIA GPU Feature Discovery service

```
cd multinode_helm_chart/
kubectl apply -f nvidia_gpu-feature-discovery_daemonset.yaml
```

## 5. Install Prometheus Kubernetes Stack

The Prometheus Kubernetes Stack installs the necessary components including Prometheus, Kube-State-Metrics, Grafana, etc. for metrics collection.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
helm install -n monitoring prometheus prometheus-community/kube-prometheus-stack \
  --set tolerations[0].key=nvidia.com/gpu \
  --set tolerations[0].operator=Exists \
  --set tolerations[0].effect=NoSchedule
```

## 6. Install NVIDIA DCGM Exporter

This exporter allows us to collect GPU metrics through DCGM which is the recommended way to monitor GPU status in our cluster.

```
helm repo add nvidia-dcgm https://nvidia.github.io/dcgm-exporter/helm-charts && helm repo update
helm install -n monitoring dcgm-exporter nvidia-dcgm/dcgm-exporter --values nvidia_dcgm-exporter_values.yaml
```

You can verify by showing the metrics collected by DCGM:

```
kubectl -n monitoring port-forward svc/dcgm-exporter 8080:9400
```

In you local browser, you should be able to see metrics in `localhost:8080`.

## 7. Install Prometheus Adapter

This allows the Triton metrics collected by Prometheus server to be available to Kuberntes' Horizontal Pod Autoscaler service.

```
helm install -n monitoring prometheus-adapter prometheus-community/prometheus-adapter \
  --set metricsRelistInterval=6s \
  --set customLabels.monitoring=prometheus-adapter \
  --set customLabels.release=prometheus \
  --set prometheus.url=http://prometheus-kube-prometheus-prometheus \
  --set additionalLabels.release=prometheus
```

To verify that Prometheus Adapter is working properly, run the following command:

```
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```

If the command fails, wait longer and retry. If the command fails for more than a few minutes then the adapter is misconfigured and will require intervention.

## 8. Install Prometheus rule for Triton metrics

This generates custom metrics from a formula that uses the Triton metrics collected by Prometheus. One of the custom metrics is used in Horizontal Pod Autoscaler (HPA). Users can modify this manifest to create their own custom metrics and set them in the HPA manifest.

```
kubectl apply -f triton-metrics_prometheus-rule.yaml
```

At this point, all metrics components should have been installed. All metrics including Triton metrics, DCGM metrics, and custom metrics should be availble to Prometheus server now. You can verify by showing all metrics in Prometheus server:

```
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 8080:9090
```

In you local browser, you should be able to see all the metrics mentioned above in `localhost:8080`.

## 9. Install EFA Kubernetes Device Plugin

Please check if EFA Kubernetes Device Plugin is already installed. If yes, please uninstall it first.

Pull the EFA Kubernetes Device Plugin helm chart:

```
helm repo add eks https://aws.github.io/eks-charts
helm pull eks/aws-efa-k8s-device-plugin --untar
```

Add tolerations in `aws-efa-k8s-device-plugin/values.yaml` at line 134 like below:

```
tolerations:
- key: nvidia.com/gpu
  operator: Exists
  effect: NoSchedule
```

Install the EFA Kubernetes Device Plugin helm chart:

```
helm install aws-efa-k8s-device-plugin --namespace kube-system ./aws-efa-k8s-device-plugin/
```

## 10. Install Cluster Autoscaler

> [!Note]
> - Autoscaler IAM add-on policy needs to be attached (done already if using the example config to create an EKS cluster).
> - The Cluster Autoscaler won't exceed the maximum number of nodes that you set in your node group. So if you want to allow more nodes to be added to your node group by the Cluste Autoscaler, make sure you set maximum nodes accordingly.
> - The Cluster Autoscaler only scales up number of nodes when there are `unschedulable` pods. It also scales down when the additional nodes become "free".

### a. Deploy the Cluster Autoscaler deployment

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

### b. Set image version

Here we set the image version to be `v1.30.2`. Make sure it matches your EKS cluster version.

```
kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=registry.k8s.io/autoscaling/cluster-autoscaler:v1.30.2
```

### c. Add the required safe-to-evict annotation to the deployment

```
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

### d. Edit the manifest file

```
kubectl -n kube-system edit deployment.apps/cluster-autoscaler

# Change 1: Add your cluster name:
- --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<your_cluster_name>

# Change 2: Add the following two lines below the line above:
- --balance-similar-node-groups
- --skip-nodes-with-system-pods=false
```

## 11. Install the LeaderWorkerSet

This allows us to use the LeaderWorkerSet API for our multi-node Triton deployment.

```
VERSION=v0.3.0
kubectl apply --server-side -f https://github.com/kubernetes-sigs/lws/releases/download/$VERSION/manifests.yaml
```

## 12. Verify installation

List all the Pods:

```
kubectl get pods -A
```

```
NAMESPACE     NAME                                                     READY   STATUS    RESTARTS   AGE
kube-system   aws-node-55lp4                                           2/2     Running   0          3h11m
kube-system   aws-node-lz7sm                                           2/2     Running   0          144m
kube-system   aws-node-qr69w                                           2/2     Running   0          179m
kube-system   cluster-autoscaler-7bc88498df-cjjrg                      1/1     Running   0          3h35m
kube-system   coredns-d9b6d6c7d-hlx8k                                  1/1     Running   0          3h35m
kube-system   coredns-d9b6d6c7d-pgzl8                                  1/1     Running   0          3h20m
kube-system   efa-aws-efa-k8s-device-plugin-6m7wl                      1/1     Running   0          144m
kube-system   efa-aws-efa-k8s-device-plugin-tz2j8                      1/1     Running   0          179m
kube-system   efs-csi-controller-7675bbb88-98rcz                       3/3     Running   0          3h35m
kube-system   efs-csi-controller-7675bbb88-vhwvq                       3/3     Running   0          3h35m
kube-system   efs-csi-node-b6ltd                                       3/3     Running   0          179m
kube-system   efs-csi-node-cp229                                       3/3     Running   0          144m
kube-system   efs-csi-node-z2r8v                                       3/3     Running   0          3h11m
kube-system   gpu-feature-discovery-shmzv                              1/1     Running   0          102m
kube-system   gpu-feature-discovery-tpg4m                              1/1     Running   0          102m
kube-system   kube-proxy-8mf5m                                         1/1     Running   0          179m
kube-system   kube-proxy-mp5x4                                         1/1     Running   0          3h11m
kube-system   kube-proxy-wx8rq                                         1/1     Running   0          144m
kube-system   node-feature-discovery-gc-7fd4d8b94f-668tz               1/1     Running   0          3h35m
kube-system   node-feature-discovery-master-5d589d89b6-fm4dv           1/1     Running   0          3h35m
kube-system   node-feature-discovery-worker-28njz                      1/1     Running   0          144m
kube-system   node-feature-discovery-worker-74vrx                      1/1     Running   0          179m
kube-system   node-feature-discovery-worker-cp2k4                      1/1     Running   0          3h11m
kube-system   nvidia-device-plugin-daemonset-5wmfq                     1/1     Running   0          179m
kube-system   nvidia-device-plugin-daemonset-btm97                     1/1     Running   0          144m
kube-system   nvidia-device-plugin-daemonset-wdv5t                     1/1     Running   0          3h11m
lws-system    lws-controller-manager-799c9c77bc-wk897                  2/2     Running   0          3h35m
monitoring    alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          3h35m
monitoring    dcgm-exporter-jmf5l                                      1/1     Running   0          102m
monitoring    dcgm-exporter-r7f8n                                      1/1     Running   0          102m
monitoring    prometheus-adapter-5447c4cc95-8db8g                      1/1     Running   0          3h35m
monitoring    prometheus-grafana-5f846bc55f-7dnsm                      3/3     Running   0          3h35m
monitoring    prometheus-kube-prometheus-operator-5464cbd4d5-svrn6     1/1     Running   0          3h35m
monitoring    prometheus-kube-state-metrics-5749f84cb-m56c7            1/1     Running   0          3h35m
monitoring    prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          3h35m
monitoring    prometheus-prometheus-node-exporter-dbm6m                1/1     Running   0          179m
monitoring    prometheus-prometheus-node-exporter-jglc6                1/1     Running   0          3h11m
monitoring    prometheus-prometheus-node-exporter-zghvb                1/1     Running   0          144m
```

List all the Deployments:

```
kubectl get deployments -A
```

```
NAMESPACE     NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-autoscaler                    1/1     1            1           4h42m
kube-system   coredns                               2/2     2            2           42d
kube-system   efs-csi-controller                    2/2     2            2           42d
kube-system   node-feature-discovery-gc             1/1     1            1           42d
kube-system   node-feature-discovery-master         1/1     1            1           42d
lws-system    lws-controller-manager                1/1     1            1           11d
monitoring    prometheus-adapter                    1/1     1            1           42d
monitoring    prometheus-grafana                    1/1     1            1           42d
monitoring    prometheus-kube-prometheus-operator   1/1     1            1           42d
monitoring    prometheus-kube-state-metrics         1/1     1            1           42d
```

List all the DaemonSets:

```
kubectl get ds -A
```

```
NAMESPACE     NAME                                  DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   aws-node                              3         3         3       3            3           <none>                   42d
kube-system   efa-aws-efa-k8s-device-plugin         2         2         2       2            2           <none>                   14d
kube-system   efs-csi-node                          3         3         3       3            3           kubernetes.io/os=linux   42d
kube-system   gpu-feature-discovery                 2         2         2       2            2           <none>                   18d
kube-system   kube-proxy                            3         3         3       3            3           <none>                   42d
kube-system   node-feature-discovery-worker         3         3         3       3            3           <none>                   42d
kube-system   nvidia-device-plugin-daemonset        3         3         3       3            3           <none>                   18d
monitoring    dcgm-exporter                         2         2         2       2            2           nvidia.com/gpu=present   14d
monitoring    prometheus-prometheus-node-exporter   3         3         3       3            3           kubernetes.io/os=linux   42d
```

List all the Services:

```
kubectl get services -A
```

```
NAMESPACE     NAME                                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
default       kubernetes                                           ClusterIP   10.100.0.1       <none>        443/TCP                        42d
kube-system   kube-dns                                             ClusterIP   10.100.0.10      <none>        53/UDP,53/TCP                  42d
kube-system   prometheus-kube-prometheus-coredns                   ClusterIP   None             <none>        9153/TCP                       42d
kube-system   prometheus-kube-prometheus-kube-controller-manager   ClusterIP   None             <none>        10257/TCP                      42d
kube-system   prometheus-kube-prometheus-kube-etcd                 ClusterIP   None             <none>        2381/TCP                       42d
kube-system   prometheus-kube-prometheus-kube-proxy                ClusterIP   None             <none>        10249/TCP                      42d
kube-system   prometheus-kube-prometheus-kube-scheduler            ClusterIP   None             <none>        10259/TCP                      42d
kube-system   prometheus-kube-prometheus-kubelet                   ClusterIP   None             <none>        10250/TCP,10255/TCP,4194/TCP   42d
lws-system    lws-controller-manager-metrics-service               ClusterIP   10.100.213.37    <none>        8443/TCP                       11d
lws-system    lws-webhook-service                                  ClusterIP   10.100.103.158   <none>        443/TCP                        11d
monitoring    alertmanager-operated                                ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP     42d
monitoring    dcgm-exporter                                        ClusterIP   10.100.62.240    <none>        9400/TCP                       14d
monitoring    prometheus-adapter                                   ClusterIP   10.100.13.192    <none>        443/TCP                        42d
monitoring    prometheus-grafana                                   ClusterIP   10.100.56.89     <none>        80/TCP                         42d
monitoring    prometheus-kube-prometheus-alertmanager              ClusterIP   10.100.40.55     <none>        9093/TCP,8080/TCP              42d
monitoring    prometheus-kube-prometheus-operator                  ClusterIP   10.100.232.224   <none>        443/TCP                        42d
monitoring    prometheus-kube-prometheus-prometheus                ClusterIP   10.100.144.122   <none>        9090/TCP,8080/TCP              42d
monitoring    prometheus-kube-state-metrics                        ClusterIP   10.100.194.231   <none>        8080/TCP                       42d
monitoring    prometheus-operated                                  ClusterIP   None             <none>        9090/TCP                       42d
monitoring    prometheus-prometheus-node-exporter                  ClusterIP   10.100.44.228    <none>        9100/TCP                       42d
```
