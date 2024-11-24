# Cilium clustermsh demo

## Kind setup
Create two kind cluster using kind. Cluster 1 will have a ingress on the control plane node.

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

```bash
ubuntu@vm:~/cilium-clustermesh-demo$ k get nodes
NAME                   STATUS     ROLES           AGE   VERSION
cilium-control-plane   NotReady   control-plane   1m   v1.31.0
cilium-worker          NotReady   <none>          1m   v1.31.0
```

Kind runs cluster nodes in docker containers in a dedicated network. Containers running in the kind network can **resolve IP addresses of other containers by their name**. 

So we can put the context name and port on the cilium configuration `k8sServiceHost`, `k8sServicePort`. Using `kubectl config get current-context`, the host should be `kind-cilium` and `6443` for cilium cluster and `kind-cilium-secondary` and `6443` for cilium-secondary cluster.


## Cilium Setup

All the values are already setup in `values-cluster1.yaml` and `values-cluster2.yaml`. The only differences between both files are:

 - k8sServiceHost
 - ingressController.enabled
 - cluster.name
 - cluster.id
 - ipam.operator.clusterPoolIPv4PodCIDRList
 - ipv4NativeRoutingCIDR
 - clustermesh.config.clusters.name      # name of the second cluster on the values-cluster1.yaml
 - clustermesh.config.clusters.addresse  # DNS addresse of the second cluster on the values-cluster1.yaml

```bash
# Cluster1
ubuntu@vm:~/cilium-clustermesh-demo$ helm upgrade --install cilium ./charts --namespace kube-system -f ./charts/values-cluster1.yaml --force
ubuntu@vm:~/cilium-clustermesh-demo$ kubectl -n kube-system get po
NAME                                           READY   STATUS    RESTARTS        AGE
cilium-envoy-7628z                             1/1     Running   0               13m
cilium-envoy-tlqqx                             1/1     Running   0               13m
cilium-nlf8d                                   1/1     Running   0               13m
cilium-node-init-5bvqd                         1/1     Running   0               13m
cilium-node-init-sdmh4                         1/1     Running   0               13m
cilium-operator-6787966855-7v7tw               1/1     Running   0               13m
cilium-tb6vs                                   1/1     Running   0               13m
clustermesh-apiserver-7d8d7ff784-wfd5m         2/2     Running   0               13m
coredns-6f6b679f8f-jk6g7                       1/1     Running   0               102m
coredns-6f6b679f8f-v5v8x                       1/1     Running   0               102m
etcd-cilium-control-plane                      1/1     Running   0               102m
hubble-relay-6bf99866c-s4qw6                   1/1     Running   0               13m
hubble-ui-77555d5dcf-w4phz                     2/2     Running   0               13m
kube-apiserver-cilium-control-plane            1/1     Running   0               102m
kube-controller-manager-cilium-control-plane   1/1     Running   0               102m
kube-scheduler-cilium-control-plane            1/1     Running   0               102m

# Cluster2
ubuntu@vm:~/cilium-clustermesh-demo$ helm upgrade --install cilium ./charts --namespace kube-system -f ./charts/values-cluster2.yaml --force
ubuntu@vm:~/cilium-clustermesh-demo$ kubectl -n kube-system get po
NAME                                                     READY   STATUS    RESTARTS   AGE
cilium-6z4nm                                             1/1     Running   0          10m
cilium-envoy-qjnc5                                       1/1     Running   0          10m
cilium-envoy-z4z4g                                       1/1     Running   0          10m
cilium-lh2zf                                             1/1     Running   0          10m
cilium-node-init-sxztv                                   1/1     Running   0          10m
cilium-node-init-tqzhq                                   1/1     Running   0          10m
cilium-operator-59899b9894-gsx4v                         1/1     Running   0          10m
clustermesh-apiserver-7d8d7ff784-fnb6t                   2/2     Running   0          10m
coredns-6f6b679f8f-fst46                                 1/1     Running   0          105m
coredns-6f6b679f8f-qz2m8                                 1/1     Running   0          105m
etcd-cilium-secondary-control-plane                      1/1     Running   0          105m
hubble-relay-6bf99866c-r9jts                             1/1     Running   0          10m
hubble-ui-77555d5dcf-xffdx                               2/2     Running   0          10m
kube-apiserver-cilium-secondary-control-plane            1/1     Running   0          105m
kube-controller-manager-cilium-secondary-control-plane   1/1     Running   0          105m
kube-scheduler-cilium-secondary-control-plane            1/1     Running   0          105m
```

### To connect both clusters

```bash

ubuntu@vm:~/cilium-clustermesh-demo$ CLUSTER1=kind-cilium
ubuntu@vm:~/cilium-clustermesh-demo$ CLUSTER2=kind-cilium-secondary
ubuntu@vm:~/cilium-clustermesh-demo$ cilium clustermesh connect --context $CLUSTER1 --destination-context $CLUSTER2
✅ Detected Helm release with Cilium version 1.16.4
✨ Extracting access information of cluster cilium2...
🔑 Extracting secrets from cluster cilium2...
⚠️   Service type NodePort detected! Service may fail when nodes are removed from the cluster!
ℹ️   Found ClusterMesh service IPs: [172.20.0.5]
✨ Extracting access information of cluster cilium...
🔑 Extracting secrets from cluster cilium...
⚠️   Service type NodePort detected! Service may fail when nodes are removed from the cluster!
ℹ️   Found ClusterMesh service IPs: [172.20.0.2]
⚠️  Cilium CA certificates do not match between clusters. Multicluster features will be limited!
ℹ️  Configuring Cilium in cluster 'kind-cilium' to connect to cluster 'kind-cilium-secondary'
ℹ️  Configuring Cilium in cluster 'kind-cilium-secondary' to connect to cluster 'kind-cilium'
✅ Connected cluster kind-cilium and kind-cilium-secondary!

# Double check
# Cluster 1
ubuntu@vm:~/cilium-clustermesh-demo$ cilium clustermesh status --context $CLUSTER1
⚠️   Service type NodePort detected! Service may fail when nodes are removed from the cluster!
✅ Service "clustermesh-apiserver" of type "NodePort" found
✅ Cluster access information is available:
  - 172.20.0.2:32382
✅ Deployment clustermesh-apiserver is ready
✅ All 2 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]
🔌 Cluster Connections:
  - cilium2: 2/2 configured, 2/2 connected
🔀 Global services: [ min:0 / avg:0.0 / max:0 ]

# Cluster 2
ubuntu@vm:~/cilium-clustermesh-demo$ cilium clustermesh status --context $CLUSTER2
⚠️   Service type NodePort detected! Service may fail when nodes are removed from the cluster!
✅ Service "clustermesh-apiserver" of type "NodePort" found
✅ Cluster access information is available:
  - 172.20.0.5:32382
✅ Deployment clustermesh-apiserver is ready
✅ All 2 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]
🔌 Cluster Connections:
  - cilium: 2/2 configured, 2/2 connected
🔀 Global services: [ min:0 / avg:0.0 / max:0 ]
```

## Test clustermesh
