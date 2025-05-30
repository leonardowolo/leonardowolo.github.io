---
title: High Available K3S Cluster
date: 2025-05-30 14:00:00 -0000
categories: [kubernetes, k3s]
tags: [homelab, proxmox, kubernetes, k3s, kube-vip, metallb, nginx]
image:
  path: /assets/img/headers/k8s-blocks.webp
  alt: k8s blocks
pin: false
---

k3s is a lightweight Kubernetes distribution that is designed to run in resource-constrained environments such as edge and IoT devices. In this documentation, we will be discussing how to set up a highly available k3s cluster with an embedded database on three nodes using kube-vip to load balance the control plane API and use MetalLB as a load balancer for exposing our applications.

### Prerequisites

Before proceeding with the setup, ensure that the following prerequisites are met:

- Three Linux machines with Ubuntu 22.04 or later installed with 2 vCPU and 2 GB of RAM
- All machines should be accessible through SSH
- A user with sudo privileges on all machines
- The IP addresses of each node

### Step 1: Configure the master node

Let’s use K3Sup to create the K3S cluster. To create the K3s cluster using K3Sup.

<aside>
⚠️ Please make sure to update the following values to match your environment:

- IP Address of the master node:`--ip = 192.168.1.151`
- User with sudo priviledges:`--user leonardo`
- IP Address to be used by kube-vip:`--tls-san = 192.168.1.150`

For more information about all the available arguments visit the GitHub [page](https://github.com/alexellis/k3sup#do-you-love-k3sup).

</aside>

```bash
k3sup install \
    --cluster \
    --ip 192.168.1.151 \
    --user leonardo \
    --tls-san 192.168.1.150 \
    --context k3s-cluster-1 \
    --local-path ~/.kube/config \
    --k3s-extra-args '--disable servicelb,traefik' \
    --k3s-channel stable
```

### Step 2: Deploy Kube-VIP **DaemonSet**

Let’s look at the Kube-VIP configuration for K3s control plane HA. In my example below, I will be using the first control node I spun up using the K3sup utility to install Kube-VIP. After you install Kube-VIP, you can then join up your additional nodes using the virtual IP.

**Apply RBAC**

The first thing we want to do is apply the RBAC script for Kube-VIP. On your K3s cluster, run the following: 

```bash
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
```

****

**Pull the image and create an alias**

SSH into the first master node and do these steps as root.

```bash
ssh 192.168.1.151
```

```bash
sudo -i
```

```bash
export KVVERSION=v0.6.0
```

```bash
# Pull image - Check latest version on github or refer to docs.
ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
```

```bash
# Create alias for the Kube-VIP command
alias kube-vip="ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
```

<aside>
⚠️ Please make sure to update the following values to match your environment:

- IP Address to be used by kube-vup:`-address 192.168.1.150`
- Network interface (check with`ip a or ifconfig`):`-interface eth0`
</aside>

```bash
# Generate and deploy the manifest
kube-vip manifest daemonset \
    --interface eth0 \
    --address 192.168.1.150 \
    --inCluster \
    --taint \
    --controlplane \
    --arp \
    --leaderElection | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml
```

This should auto deploy because the manifest is copied to the k3s/server folder. Before we continue let's check if everything is up and running.

```bash
# Start pinging the virtual ip
ping 192.168.1.150 -c 100
```

```bash
# Check if the daemonset is running
kubectl get daemonset -A
```

```bash
# Check the logs of kube-vip
kubectl -n kube-system logs <podname>
```

**Modify the .kube/config.yml**

Modify the config and change the server IP address to that used by kube-vip `192.168.1.150`.

### Step 3: Deploy additional master nodes

Let’s use K3Sup to create the K3S cluster. To create the K3s cluster using K3Sup.

<aside>
⚠️ Please make sure to update the following values to match your environment:

- IP Address of new master nodes:
    - `-ip = 192.168.1.152`
    - `-ip = 192.168.1.153`
- IP address used by kube-vip:`-server-ip 192.168.1.150`
- User with sudo priviledges:`-user leonardo`
</aside>

**Deploy k3s to additional master nodes & join cluster**

```bash
# Join the second master node
k3sup join \
    --ip 192.168.1.152 \
    --server-ip 192.168.1.150 \
    --server \
    --sudo \
    --user leonardo \
    --k3s-extra-args '--disable servicelb,traefik' \
    --k3s-channel stable
```

```bash
# Join the third master node
k3sup join \
    --ip 192.168.1.153 \
    --server-ip 192.168.1.150 \
    --server \
    --sudo \
    --user leonardo \
    --k3s-extra-args '--disable servicelb,traefik' \
    --k3s-channel stable
```

**Check if the master nodes are up**

```bash
kubectl get nodes
```

### Step 4: Taint all master nodes

Tainting master nodes to prevent workloads from being scheduled on them is generally not recommended, as it can interfere with the proper functioning of your cluster. Master nodes are responsible for managing the cluster's control plane components and should not be overloaded with application workloads.

```bash
# Taint all master nodes
kubectl taint nodes <master-node-name> CriticalAddonsOnly=true:NoExecute
```

### Step 5: Add worker nodes

Let’s use K3Sup to create the K3S cluster. To create the K3s cluster using K3Sup.

<aside>
⚠️ Please make sure to update the following values to match your environment:

- IP Address of new worker nodes:
    - `-ip = 192.168.1.161`
    - `-ip = 192.168.1.162`
    - `-ip = 192.168.1.163`
- IP address used by kube-vip:`-server-ip 192.168.1.150`
- User with sudo priviledges:`-user leonardo`
- Agents nodes agent-nodes do automatically inherit the flags `--k3s-extra-args '--disable servicelb,traefik,local-storage'`
</aside>

**Deploy k3s to additional master nodes & join cluster**

```bash
# Join the first worker node
k3sup join \
    --ip 192.168.1.161 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

```bash
# Join the second worker node
k3sup join \
    --ip 192.168.1.162 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

```bash
# Join the third worker node
k3sup join \
    --ip 192.168.1.163 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

```bash
# Join the fourth worker node
k3sup join \
    --ip 192.168.1.164 \
    --server-ip 192.168.1.150 \
    --sudo \
    --user leonardo \
    --k3s-channel stable
```

**Check if the worker nodes are up**

```bash
kubectl get nodes
```

### Step 6: Deploy and Configure MetalLB

Metallb is a load balancer solution for Kubernetes that allows you to create a layer 2 or layer 3 load balancer in your Kubernetes cluster. This step will explain how to deploy Metallb.

**Install the manifests**

```bash
# Apply the metallb configurations
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```

**Defining the IPs to assign to the load balancer services**

In order to assign an IP to the services, MetalLB must be instructed to do so via the`IPAddressPool`CR.

All the IPs allocated via`IPAddressPool`s contribute to the pool of IPs that MetalLB uses to assign IPs to services.

```yaml
# metallb-addrespool.yml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: loadbalancer-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.245-192.168.1.250
```

Multiple instances of`IPAddressPools` can co-exist and addresses can be defined by CIDR, by range, and both IPV4 and IPV6 addresses can be assigned.

**Announce The Service IPs**

Once the IPs are assigned to a service, they must be announced. The specific configuration depends on the protocol(s) you want to use to announce service IPs. We are going to use Layer 2 Configuration.

- Layer 2 mode is the simplest to configure: in many cases, you don’t need any protocol-specific configuration, only IP addresses.
- Layer 2 mode does not require the IPs to be bound to the network interfaces of your worker nodes. It works by responding to ARP requests on your local network directly, to give the machine’s MAC address to clients.

In order to advertise the IP coming from an`IPAddressPool`, an`L2Advertisement`instance must be associated with the`IPAddressPool`.

```yaml
# metallb-l2advertisement.yml
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: loadbalancer-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - loadbalancer-pool
```

Setting no`IPAddressPool`selector in an`L2Advertisement`instance is interpreted as that instance being associated with all the`IPAddressPools` available. So in case, there are specialized`IPAddressPools`, and only some of them must be advertised via L2, the list of`IPAddressPools` we want to advertise the IPs from must be declared (alternative, a label selector can be used).

**Deploy the Metallb custom resources**

```bash
kubectl apply -f metallb-addrespool.yml
```

```bash
kubectl apply -f metallb-l2advertisement.yml
```

**Check if MetalLb is running**

```bash
kubectl -n metallb-system get daemonsets
```

```bash
kubectl -n metallb-system logs <podname>
```

### Step 7: **Test Deployment using Nginx**

When everything is up and running we can test our cluster by running a Nginx application.

**Create the manifests files**

```yaml
# deployment.yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

```yaml
# service.yml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

**Deploy manifests**

```bash
kubectl apply -f deployment.yml
```

```bash
kubectl apply -f service.yml
```

**Check the IP address assigned by Metal-LB**

```bash
kubectl describe service nginx
```

**Check accessible**

```bash
# ip address handed out by MetalLB from described service above
curl 192.168.0.XXX
```

**Teardown**

```bash
kubectl delete deployment,service nginx
```

## Conclusion

Setting up a high-availability Kubernetes cluster with K3s, Kube-VIP, and MetalLB provides a lightweight yet powerful foundation for running production-grade workloads. By combining K3s’s simplicity, Kube-VIP’s virtual IP failover, and MetalLB’s load balancing capabilities, you can achieve reliable access to services and maintain cluster resilience without the overhead of a full Kubernetes distribution.

This setup is ideal for homelabs, edge deployments, or anyone looking to gain hands-on experience with HA Kubernetes architecture. As always, make sure to monitor your cluster’s health and security, and consider adding GitOps practices or backup strategies to further strengthen your infrastructure.

**Happy clustering!**
