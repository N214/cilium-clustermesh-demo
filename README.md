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
```sh
kind create cluster -n cilium --config kind/config-cluster1.yaml
kind create cluster -n cilium-secondary --config kind/config-cluster2.yaml
```

## Cilium Setup
