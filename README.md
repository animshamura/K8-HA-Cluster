## Why high availability is mandatory for Kubernetes clusters? ##
A highly available Kubernetes cluster is essential for modern cloud-native applications due to its ability to ensure reliability, scalability, and resilience. By distributing workload across multiple nodes, these clusters eliminate single points of failure, minimizing downtime and disruptions. Redundancy in cluster components enables continuous operation even if individual nodes or components fail. This fault tolerance not only improves uptime but also enhances the overall reliability of applications hosted within the cluster. <br/>

Scalability is another critical benefit of highly available Kubernetes clusters. They can dynamically scale resources up or down based on workload demands, ensuring optimal performance without manual intervention. Load balancing mechanisms distribute incoming traffic across multiple nodes and pods, enhancing performance and resource utilization. This elasticity allows organizations to efficiently handle varying levels of traffic and adapt to changing business needs. <br/>

Operational efficiency is also significantly improved in highly available Kubernetes environments. Automated failover mechanisms and rolling updates allow for seamless maintenance and updates without affecting service availability. Security and compliance requirements are met through isolation and redundancy, ensuring data integrity and protecting against potential breaches. Ultimately, highly available Kubernetes clusters not only support business continuity and disaster recovery but also enable organizations to meet stringent service level agreements (SLAs) by providing robust and reliable infrastructure for their applications.
## Highly Available Kubernetes Cluster Architecture: ##
A Highly Available Kubernetes cluster architecture is designed with redundancy and fault tolerance in mind, ensuring continuous operation and scalability for containerized applications. The architecture centers around a distributed control plane comprising redundant instances of the API server, etcd, controller manager, and scheduler. These components manage cluster state, resource allocation, and workload distribution across multiple nodes.

Etcd serves as a distributed key-value store for storing critical cluster data, configured in a fault-tolerant manner with an odd-numbered quorum of nodes. Controller managers oversee various aspects of cluster management, ensuring resilience by running multiple instances across different control plane nodes. The scheduler optimizes resource allocation by placing pods on worker nodes based on resource requirements and policies, enhancing efficiency and performance.

Worker nodes execute containers and communicate with the control plane components via APIs. Load balancers are used to distribute incoming traffic across worker nodes and services, further improving availability and scalability. The architecture supports elasticity, allowing nodes and components to scale dynamically based on workload demands, while automated failover mechanisms ensure resilience against node failures or disruptions.

Overall, a Highly Available Kubernetes cluster architecture provides a robust foundation for deploying and managing cloud-native applications, meeting high availability requirements and enabling organizations to maintain operational efficiency and meet stringent SLAs.

<br/>
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/ha-kube-cluster.drawio.png?raw=true">

The above architecture refers a highly available Kubernetes cluster which doesn't fall in one point failure for loadbalancing.
## Requirements: ##
- Minimum 3 Master Nodes
- Minimum 2 Loadbalancers
- HAProxy
- Keepalived
## Sections: ##
- VM Creation
- HAProxy and Keepalived Installation in Loadbalancer VMs
- Cluster Pre-requisites
- Bootstrapping Clusters
- Check High Availability

## VM Creation: ##

#### Step 1: Create 6 VMs in Proxmox. ####

<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr1.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr2.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr3.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr4.png?raw=true">

<font color="blue">Create one VM and clone 5 VMs to get total of 6 VMs in Proxmox. Again, hostname and IP address of each VM has to be edited.<font>

#### Step 2: Change hostname of the VMs in Proxmox. ####

```
sudo hostnamectl set-hostname master1
```
#### Step 3: Change IP addres of the VMs in Proxmox. ####

```
sudo vim /etc/netplan/00-installer-config.yaml
```
Edit "00-installer-config.yaml" file according to the desired IP.

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens18:
      addresses:
      - 172.17.17.112/24
      nameservers:
        addresses:
        - 8.8.8.8
        search: []
      routes:
      - to: default
        via: 172.17.17.1
  version: 2
```
| VM | Host Name| IP Address | 
|------|--------|------------|
|Master Node 1 |master1| 172.17.17.110|
|Master Node 2 |master2| 172.17.17.111|
|Master Node 3 |master3| 172.17.17.112|
|Worker Node 1 |worker1| 172.17.17.113|
|Loadbalancer 1 |loadbalancer1| 172.17.17.114|
|Loadbalancer 2 |loadbalancer2| 172.17.17.115|
|Virtual IP | - | 172.17.17.116|

## HAProxy and Keepalived Installation in Loadbalancer VMs: ##

**Step 1: Install HAProxy and Keepalived.**

```
apt update && apt install -y keepalived haproxy
```
**Step 2: Configure Keepalived.**

```
cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.17.17.116; then
  curl --silent --max-time 2 --insecure https://172.17.17.116:6443/ -o /dev/null || errorExit "Error GET https://172.17.17.116:6443/"
fi
EOF

chmod +x /etc/keepalived/check_apiserver.sh
```
Create keepalived config /etc/keepalived/keepalived.conf.

```
cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens18
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        172.17.17.116
    }
    track_script {
        check_apiserver
    }
}
EOF
```
**Step 3: Enable and start Keepalived service.**
```
systemctl enable --now keepalived
systemctl start --now keepalived
```
**Step 4: Configure HAProxy.**
```
cat >> /etc/haproxy/haproxy.cfg <<EOF

frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server master1 172.17.17.110:6443 check fall 3 rise 2
    server master2 172.17.17.111:6443 check fall 3 rise 2
    server master3 172.17.17.112:6443 check fall 3 rise 2

EOF
```
**Step 5: Enable and restart HAProxy service.**
```
systemctl enable haproxy && systemctl restart haproxy
```
## Cluster Pre-requisites: ##
**Step 1: Disable swap.**
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
**Step 2: Disable firewall.**
```
systemctl disable --now ufw
```
**Step 3: Enable and Load Kernel modules.**
```
{
cat >> /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
}
```
**Step 4: Add Kernel settings.**
```
{
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
}
```
**Step 5: Install containerd runtime.**
```
{
  apt update
  apt install -y containerd apt-transport-https
  mkdir /etc/containerd
  containerd config default > /etc/containerd/config.toml
  systemctl restart containerd
  systemctl enable containerd
}
```
**Step 6: Add apt repository for Kubernetes.**
```
{
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
}
```
**Step 7: Install Kubernetes components.**
```
{
  apt update
  apt install -y kubeadm kubelet kubectl
}
```
## Bootstrapping Clusters: ##
**Step 1: Initialize Kubernetes cluster.**
```
kubeadm init --control-plane-endpoint="172.17.17.116:6443" --upload-certs --apiserver-advertise-address=172.17.17.110 --pod-network-cidr=192.168.1.0/16
```
**Step 2: Join master and worker nodes.** <br/> <br/>
Executing print-join-command for getting the joining token.
```
kubeadm token create --print-join-command
```
For joining master node 2 in the multicluster.
```
kubeadm join 172.17.17.110:6443 --token 5g5jo2.agl26wfzkujgjt3s --discovery-token-ca-cert-hash ha256:57795a664200425258ed0619af960fe476d1ae93f99182a3d710ce1185468d3f --apiserver-advertise-address=172.17.17.111
```
For joining master node 3 in the multicluster.
```
kubeadm join 172.17.17.110:6443 --token 5g5jo2.agl26wfzkujgjt3s --discovery-token-ca-cert-hash ha256:57795a664200425258ed0619af960fe476d1ae93f99182a3d710ce1185468d3f --apiserver-advertise-address=172.17.17.112
```
For joining worker node 1 in the multicluster.
```
kubeadm join 172.17.17.110:6443 --token 5g5jo2.agl26wfzkujgjt3s --discovery-token-ca-cert-hash ha256:57795a664200425258ed0619af960fe476d1ae93f99182a3d710ce1185468d3f 
```
**Step 3: Deploy Calico network.**
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml
```
## Check High Availability: ##
**Step 1: Copy kubeconfig file to the host machine.**
```
scp anim@172.17.18.200:/home/anim/.kube/config .
```
**Step 2: Watch nodes.**
```
watch kubectl get nodes
```
**Step 3: Execute below command in Loadbalancer 1.**
```
ip a s
```
This will show the virtual IP 172.17.17.116 is attached to Loadbalancer 1.

**Step 3: Turn off the Loadbalancer 1 and execute below command in Loadbalancer 2.**
```
ip a s
```
Now, this will show the virtual IP 172.17.17.116 is attached to Loadbalancer 1. <br/>

