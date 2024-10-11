In this section, I will show how to deploy Kubernetes 1.21 with Calico CNI plugin.

Please ensure that you have a sufficient number of nodes prepared and have already installed Ubuntu Server 20.04.

The same deployment method is applicable at least until version 1.23 and has been verified to be upgradable to the current latest version 1.27 in subsequent articles.

## Add a Model

The model holds a specific deployment. It is a good idea to create a new one specifically for each deployment.

```bash
juju add-model k8s-lab
```

Remember that you can have multiple models on each controller, so you can deploy multiple Kubernetes clusters, or other applications.

## Machine Preparation

### Add some machines

For a Production-Ready Charmed Kubernetes, you should have at least three etcd nodes, two master nodes, and several worker nodes. All these nodes should have password-less login set up in advance.

> [!NOTE]
> the master and etcd are not deployed on the same node. Although they can be deployed on the same node, I recommend deploying etcd separately.

```bash
juju switch k8s-lab
# ETCD
juju add-machine ssh:ares@100.64.1.121
juju add-machine ssh:ares@100.64.1.122
juju add-machine ssh:ares@100.64.1.123
# Master
juju add-machine ssh:ares@100.64.1.124
juju add-machine ssh:ares@100.64.1.125
# Worker
juju add-machine ssh:ares@100.64.1.126
juju add-machine ssh:ares@100.64.1.127
# kubeapi-load-balancer
# If you are using an external load balancer, these two machines are not necessary.
juju add-machine ssh:ares@100.64.1.128
juju add-machine ssh:ares@100.64.1.129
```

### View existing machines

Run the following command to see if the machine has been set up. Also, make sure to remember the machine's ID because we'll need it for our deployment later on.

```bash
juju status
Model    Controller  Cloud/Region            Version  SLA          Timestamp
k8s-lab  infra-demo  ares-homelab/default  2.9.25   unsupported  02:09:31Z

Machine  State    DNS           Inst id              Series  AZ  Message
0        started  100.64.1.121  manual:100.64.1.121  focal       Manually provisioned machine
1        started  100.64.1.122  manual:100.64.1.122  focal       Manually provisioned machine
...
6        started  100.64.1.127  manual:100.64.1.127  focal       Manually provisioned machine
7        started  100.64.1.128  manual:100.64.1.128  focal       Manually provisioned machine
8        started  100.64.1.129  manual:100.64.1.129  focal       Manually provisioned machine
```

## Deploy Kubernetes

Now, we begin deploying Kubernetes according to the roles assigned to the machine in the Machine Preparation step.

Download https://git.motofans.club/Motofans/charmed-kubernetes/archive/Bugfix-Release_657.zip and unzip it.

Please proceed with the following deployment command：

> [!NOTE]
> At the beginning, we will not deploy it in a high-availability state. We will temporarily deploy only a single replica for each component.
> 
> Please make sure to replace the "service-cidr" and "calico-cidr" with the appropriate values.
> 
> We use Calico CNI and enable IgnoreLooseRPF, You need to install ethtool on the machine in advance.

```bash
# Core
for i in {0..7}; do
  juju scp resources/core/core.* $i:
  juju run --machine $i "sudo snap ack /home/ubuntu/core.assert"
  juju run --machine $i "sudo snap install --dangerous /home/ubuntu/core.snap"
done

# EasyRSA
juju deploy --to 0 ./easyrsa
juju attach easyrsa easyrsa=./resources/easyrsa/easyrsa.tgz

# ETCD
juju deploy --to 0 ./etcd \\
    --config bind_to_all_interfaces=false \\
    --config channel=3.4/stable
juju attach etcd snapshot=./resources/etcd/snapshot.gz
juju attach etcd etcd=./resources/etcd/etcd.snap

# Containerd
juju deploy ./containerd

# Kubernetes-master
# Disable the built-in dashboard
# Use the IPVS mode
juju deploy --to 3 ./kubernetes-master \
    --config channel=1.21/stable \
    --config service-cidr=172.31.192.0/21 \
    --config enable-dashboard-addons=false \
    --config image-registry=jcr.motofans.club/mirror/rocks.canonical.com/cdk \
    --config proxy-extra-args='bind-address=0.0.0.0 proxy-mode=ipvs'

juju attach kubernetes-master core=./resources/core/core.snap
juju attach kubernetes-master cdk-addons=./resources/kubernetes-master/cdk-addons.snap
juju attach kubernetes-master kube-apiserver=./resources/kubernetes-master/kube-apiserver.snap
juju attach kubernetes-master kube-controller-manager=./resources/kubernetes-master/kube-controller-manager.snap
juju attach kubernetes-master kube-scheduler=./resources/kubernetes-master/kube-scheduler.snap
juju attach kubernetes-master kube-proxy=./resources/kubernetes-master/kube-proxy.snap
juju attach kubernetes-master kubectl=./resources/kubernetes-master/kubectl.snap

# Kubernetes-worker
# Use the IPVS mode
juju deploy --to 5 ./kubernetes-worker \
    --config channel=1.21/stable \
    --config ingress=false \
    --config proxy-extra-args='bind-address=0.0.0.0 proxy-mode=ipvs'

juju attach kubernetes-worker cni-amd64=./resources/kubernetes-worker/cni-amd64.tgz
juju attach kubernetes-worker kube-proxy=./resources/kubernetes-worker/kube-proxy.snap
juju attach kubernetes-worker kubectl=./resources/kubernetes-worker/kubectl.snap
juju attach kubernetes-worker kubelet=./resources/kubernetes-worker/kubelet.snap

# Calico
juju deploy ./calico \
    --config cidr=172.31.128.0/18 \
    --config vxlan=Always \
    --config ignore-loose-rpf=true

juju attach calico calico=./resources/calico/calico.gz
juju attach calico calico-node-image=./resources/calico/calico-node-image.gz
juju attach calico calico-upgrade=./resources/calico/calico-upgrade.gz

# Add relate
juju relate etcd:certificates easyrsa:client
juju relate kubernetes-master:kube-control kubernetes-worker:kube-control
juju relate kubernetes-master:certificates easyrsa:client
juju relate kubernetes-worker:certificates easyrsa:client
juju relate kubernetes-master:etcd etcd:db
juju relate containerd:containerd kubernetes-worker:container-runtime
juju relate containerd:containerd kubernetes-master:container-runtime
juju relate kubernetes-master:kube-api-endpoint kubernetes-worker:kube-api-endpoint
juju relate calico:etcd etcd:db
juju relate calico:cni kubernetes-master:cni
juju relate calico:cni kubernetes-worker:cni
```

Juju is now busy creating instances, installing software and connecting the different parts of the cluster together, which can take several minutes. You can monitor what’s going on by running:

```bash
watch -c juju status --color
```

Once all the workloads are displayed as active, we can start increasing the number of replicas for the components by running:

```bash
juju add-unit etcd --to 1
juju add-unit etcd --to 2
juju add-unit kubernetes-master --to 4
juju add-unit kubernetes-worker --to 6
```

## KubeAPI Load Balancer

Once the scaling is complete, We've got two Master nodes, and to spread the load across both, we need to bring in a load balancer (LB). This LB can be set up using a controller internally, or externally using F5 or other load balancing devices.

### Software LB

*If you are using an external load balancer, you can skip this part.*

Here's how to set up the software LB using a controller.

```bash
export VIP=100.64.1.128
export VIP_HOSTNAME=kube-api.motofans.club
juju deploy --to 7 ./kubeapi-load-balancer

juju config kubernetes-master extra_sans="$VIP $VIP_HOSTNAME"
juju config kubeapi-load-balancer extra_sans="$VIP $VIP_HOSTNAME"

juju remove-relation kubernetes-master:kube-api-endpoint kubernetes-worker:kube-api-endpoint

juju relate kubernetes-master:kube-api-endpoint kubeapi-load-balancer:apiserver
juju relate kubernetes-worker:kube-api-endpoint kubeapi-load-balancer:website
juju relate kubeapi-load-balancer:certificates easyrsa:client
juju relate kubernetes-master:loadbalancer kubeapi-load-balancer:loadbalancer
```

Now, we have a load balancer distributing requests across two master nodes.

You may have noticed that the load balancer itself isn't highly available as it's deployed on a specific node. If this node experiences issues, worker nodes will be unable to access the master nodes. We can solve this issue by deploying Keepalived:

```bash
export VIP=100.64.1.110
export VIP_HOSTNAME=kube-api.motofans.club
juju deploy ./keepalived

juju config keepalived virtual_ip=$VIP
juju config keepalived vip_hostname=$VIP_HOSTNAME
juju config kubeapi-load-balancer extra_sans="$VIP $VIP_HOSTNAME"
juju config kubernetes-master extra_sans="$VIP $VIP_HOSTNAME"

juju add-relation keepalived:juju-info kubeapi-load-balancer:juju-info
juju add-relation keepalived:lb-sink kubeapi-load-balancer:website
juju add-relation keepalived:loadbalancer kubernetes-master:loadbalancer
juju add-relation keepalived:website kubernetes-worker:kube-api-endpoint

juju remove-relation kubernetes-worker:kube-api-endpoint kubeapi-load-balancer:website
juju remove-relation kubernetes-master:loadbalancer kubeapi-load-balancer:loadbalancer

juju add-unit kubeapi-load-balancer --to 8

juju config kubernetes-master proxy-extra-args='bind-address=0.0.0.0 proxy-mode=ipvs master=https://kube-api.motofans.club:443'
juju config kubernetes-worker proxy-extra-args='bind-address=0.0.0.0 proxy-mode=ipvs master=https://kube-api.motofans.club:443'
```

### External Load Balancer

**Recommendation**

*If you are not using an external load balancer, you can skip this part.*

When using an external load balancer, you need to distribute the traffic to two Master nodes, and it is recommended to ensure high availability of the external load balancer in a production environment.

```
juju config kubernetes-master extra_sans="$VIP $VIP_HOSTNAME"
juju config kubernetes-master loadbalancer-ips="$VIP $VIP_HOSTNAME"

```

## Retrieve Config File

Finally, You can use the following command to retrieve the cluster config file:

```bash
mkdir -p .kube
juju scp kubernetes-master/0:config ~/.kube/config
```

## Use CoreDNS charm

CoreDNS has been the default DNS provider for Charmed Kubernetes clusters since 1.14. It will be installed and configured as part of the install process of Charmed Kubernetes.

For additional control over CoreDNS, you can also deploy it into the cluster using the [[CoreDNS Kubernetes operator charm](https://charmhub.io/coredns)](https://charmhub.io/coredns).

```bash
juju config -m k8s-lab kubernetes-master dns-provider=none
juju add-k8s k8s-cloud --controller infra-demo
juju add-model k8s-model k8s-cloud
cat ./resources/coredns-image.tgz
juju deploy ./coredns --resource coredns-image=./resources/coredns-image.tgz
juju offer coredns:dns-provider
juju consume -m k8s-lab k8s-model.coredns
juju relate -m k8s-lab coredns kubernetes-master
```

Once everything settles out, new or restarted pods will use the CoreDNS charm as their DNS provider. The CoreDNS charm config allows you to change the cluster domain, the IP address or config file to forward unhandled queries to, add additional DNS servers, or even override the Corefile entirely.

<!-- ##{"timestamp":1667750400}## -->