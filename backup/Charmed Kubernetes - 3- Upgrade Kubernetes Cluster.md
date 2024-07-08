## Infrastructure updates

### Containerd

```
juju upgrade-charm containerd --path=./containerd

```

### ETCD

```
juju upgrade-charm etcd --path=./etcd
juju config etcd channel=3.4/stable
juju attach etcd etcd=./resources/etcd/etcd.snap

```

### EasyRSA

```
juju upgrade-charm easyrsa --path=./easyrsa --resource esayrsa=./resources/easyrsa/easyrsa.tgz

```

### Calico CNI

```
juju upgrade-charm calico --path=./calico \\
    --resource calico=./resources/calico/calico.gz \\
    --resource calico-node-image=./resources/calico/calico-node-image.gz \\
    --resource calico-upgrade=./resources/calico/calico-upgrade.gz

```

## Upgrading Kubernetes

### KubeAPI Load Balancer

```
juju upgrade-charm kubeapi-load-balancer --path=./kubeapi-load-balancer
juju upgrade-charm keepalived --path=./keepalived

```

### Kubernetes-Master Units

```
juju upgrade-charm kubernetes-master --path=./kubernetes-master \\
    --resource cni-amd64=./resources/kubernetes-master/cni-amd64.tgz \\
    --resource cni-arm64=./resources/kubernetes-master/cni-arm64.tgz \\
    --resource cni-s390x=./resources/kubernetes-master/cni-s390x.tgz
juju attach kubernetes-master core=./resources/core/core.snap
juju attach kubernetes-master cdk-addons=./resources/kubernetes-master/cdk-addons.snap
juju attach kubernetes-master kube-apiserver=./resources/kubernetes-master/kube-apiserver.snap
juju attach kubernetes-master kube-controller-manager=./resources/kubernetes-master/kube-controller-manager.snap
juju attach kubernetes-master kube-scheduler=./resources/kubernetes-master/kube-scheduler.snap
juju attach kubernetes-master kube-proxy=./resources/kubernetes-master/kube-proxy.snap
juju attach kubernetes-master kubectl=./resources/kubernetes-master/kubectl.snap

juju config kubernetes-master channel=1.22/stable
juju run-action kubernetes-master/0 upgrade
juju run-action kubernetes-master/1 upgrade

```

### Update Relation

```
juju add-relation kubernetes-master:loadbalancer-external kubeapi-load-balancer:lb-consumers
juju add-relation kubernetes-master:loadbalancer-internal kubeapi-load-balancer:lb-consumers

```

### Kubernetes-Worker Units

```
juju upgrade-charm kubernetes-worker --path=./kubernetes-worker \\
    --resource cni-amd64=./resources/kubernetes-worker/cni-amd64.tgz
juju attach kubernetes-worker kube-proxy=./resources/kubernetes-worker/kube-proxy.snap
juju attach kubernetes-worker kubectl=./resources/kubernetes-worker/kubectl.snap
juju attach kubernetes-worker kubelet=./resources/kubernetes-worker/kubelet.snap

juju config kubernetes-worker channel=1.22/stable
juju run-action kubernetes-worker/0 upgrade
juju run-action kubernetes-worker/1 upgrade

```

<!-- ##{"timestamp":1667750400}## -->