# Cilium clustermsh demo

## Kind setup
Create two kind cluster on the VM using kind. Cluster 1 will have nginx as ingress on the control plane node.

The cluster can limit `podSubnet` and `serviceSubnet` using the configuration file. To connect both clusters using cilium, the pods and services ips should not conflict. 
Note: Cilium can also manage pod IPs from the helm charts using `clusterPoolIPv4PodCIDRList` but we prefer to set it up from the beginning.

Cluster 1:
```yaml
  podSubnet: 10.1.0.0/16
  serviceSubnet: 172.20.1.0/24
```

Cluster 2:
```yaml
  podSubnet: 10.2.0.0/16
  serviceSubnet: 172.20.2.0/24
```

Start both clusters:
```bash
kind create cluster -n cilium --config kind/config-cluster1.yaml
kind create cluster -n cilium-secondary --config kind/config-cluster2.yaml
```

Since both cluster has been created without CNI and kube-proxy, they are not ready. Since kube-proxy being disabled, cilium operator will not be able to talk to the control plane because **the in-cluster Kubernetes service cannot be reached**.
Kind runs cluster nodes in docker containers in a dedicated network. Containers running in the kind network can **resolve IP addresses of other containers by their name**. 

So we can put the context name and port on the cilium configuration `k8sServiceHost`, `k8sServicePort`. Using `kubectl config get current-context`, the host should be `kind-cilium` and `6443` for cilium cluster and `kind-cilium-secondary` and `6443` for cilium-secondary cluster.


```bash
ubuntu@vm:~/cilium-clustermesh-demo$ k get nodes
NAME                   STATUS     ROLES           AGE   VERSION
cilium-control-plane   NotReady   control-plane   1m   v1.31.0
cilium-worker          NotReady   <none>          1m   v1.31.0
```


## Cilium Setup

```bash
helm upgrade cilium ./cilium --namespace kube-system -f ./cilium/values.yaml

```
