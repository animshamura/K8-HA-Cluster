## Why high availability is mandatory for Kubernetes clusters? ##
<div align="justify">A highly available Kubernetes cluster is essential for modern cloud-native applications due to its ability to ensure reliability, scalability, and resilience. By distributing workload across multiple nodes, these clusters eliminate single points of failure, minimizing downtime and disruptions. Redundancy in cluster components enables continuous operation even if individual nodes or components fail. This fault tolerance not only improves uptime but also enhances the overall reliability of applications hosted within the cluster. <br/> <br/>

Scalability is another critical benefit of highly available Kubernetes clusters. They can dynamically scale resources up or down based on workload demands, ensuring optimal performance without manual intervention. Load balancing mechanisms distribute incoming traffic across multiple nodes and pods, enhancing performance and resource utilization. This elasticity allows organizations to efficiently handle varying levels of traffic and adapt to changing business needs. <br/>

Operational efficiency is also significantly improved in highly available Kubernetes environments. Automated failover mechanisms and rolling updates allow for seamless maintenance and updates without affecting service availability. Security and compliance requirements are met through isolation and redundancy, ensuring data integrity and protecting against potential breaches. Ultimately, highly available Kubernetes clusters not only support business continuity and disaster recovery but also enable organizations to meet stringent service level agreements (SLAs) by providing robust and reliable infrastructure for their applications.</div>
## Highly Available Kubernetes Cluster Architecture: ##
<div align="justify">A Highly Available Kubernetes cluster architecture is designed with redundancy and fault tolerance in mind, ensuring continuous operation and scalability for containerized applications. The architecture centers around a distributed control plane comprising redundant instances of the API server, etcd, controller manager, and scheduler. These components manage cluster state, resource allocation, and workload distribution across multiple nodes. <br/> <br/>

Etcd serves as a distributed key-value store for storing critical cluster data, configured in a fault-tolerant manner with an odd-numbered quorum of nodes. Controller managers oversee various aspects of cluster management, ensuring resilience by running multiple instances across different control plane nodes. The scheduler optimizes resource allocation by placing pods on worker nodes based on resource requirements and policies, enhancing efficiency and performance.<br/>

Worker nodes execute containers and communicate with the control plane components via APIs. Load balancers are used to distribute incoming traffic across worker nodes and services, further improving availability and scalability. The architecture supports elasticity, allowing nodes and components to scale dynamically based on workload demands, while automated failover mechanisms ensure resilience against node failures or disruptions. <br/>
<br/>
Overall, a Highly Available Kubernetes cluster architecture provides a robust foundation for deploying and managing cloud-native applications, meeting high availability requirements and enabling organizations to maintain operational efficiency and meet stringent SLAs.</div>

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
|Master Node-1 |master1| 172.17.17.110|
|Master Node-2 |master2| 172.17.17.111|
|Master Node-3 |master3| 172.17.17.112|
|Worker Node-1 |worker1| 172.17.17.113|
|Loadbalancer-1 |loadbalancer1| 172.17.17.114|
|Loadbalancer-2 |loadbalancer2| 172.17.17.115|

| Name | IP Address|
|------|-----------|
|Virtual IP | 172.17.17.116|

## HAProxy and Keepalived Installation in Loadbalancer VMs: ##

**Step 1: Install HAProxy and Keepalived.**

```
sudo apt update && sudo apt install -y keepalived haproxy
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
sudo systemctl enable --now keepalived
sudo systemctl start --now keepalived
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
sudo systemctl enable haproxy && sudo systemctl restart haproxy
```
## Cluster Pre-requisites: ##
**Step 1: Disable swap.**
```
sudo apt-get update
sudo swapoff -a
sudo vim /etc/fstab
sudo init 6
```
**Step 2: Disable firewall.**
```
sudo systemctl disable --now ufw
```
**Step 3: Enable and load kernel modules.**
```
{
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
}
```
**Step 4: Add kernel settings.**
```
{
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
}
```
**Step 5: Install containerd runtime, modify configuaration and restart service.**

```
{
sudo apt-get update
sudo apt-get install -y containerd
}
```
Modify containerd configuration.
```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
cat /etc/containerd/config.toml
```
Restart containerd service.
```
sudo systemctl restart containerd.service
sudo systemctl status containerd
```

**Step 6: Install Kubernetes components.**
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo apt-get install -y apt-transport-https ca-certificates curl


curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
{
  sudo apt update
  sudo apt install -y kubeadm kubelet kubectl
}
```
## Bootstrapping Clusters: ##
**Step 1: Initialize Kubernetes cluster.**
```
kubeadm init --control-plane-endpoint="172.17.17.116:6443" --upload-certs --apiserver-advertise-address=172.17.17.110 --pod-network-cidr=192.168.1.0/16
```
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
**Step 3: Deploy Calico network.**
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/custom-resources.yaml -O
kubectl create -f custom-resources.yaml
```
**Step 3: Join master and worker nodes.** <br/> <br/>
Executing print-join-command for getting the joining token.
```
kubeadm token create --print-join-command
```
For joining master-node-2 in the multicluster.
```
kubeadm join 172.17.17.116:6443 --token 6u1d8n.pxipslewcckpvql8 --discovery-token-ca-cert-hash sha256:9b1b55035c2671b63635f4dbcc218c499d09d6e3319992d954754832ed988fe2 --control-plane --certificate-key 8be7f614b62e0a0b9999e14966c572c68894fa995c9124f12e946777973227b9 --apiserver-advertise-address=172.17.17.111
```
For joining master-node-3 in the multicluster.
```
kubeadm join 172.17.17.116:6443 --token 6u1d8n.pxipslewcckpvql8 --discovery-token-ca-cert-hash sha256:9b1b55035c2671b63635f4dbcc218c499d09d6e3319992d954754832ed988fe2 --control-plane --certificate-key 8be7f614b62e0a0b9999e14966c572c68894fa995c9124f12e946777973227b9 --apiserver-advertise-address=172.17.17.112
```
For joining worker-node-1 in the multicluster.
```
sudo kubeadm join 172.17.17.116:6443 --token m5mp8x.hiy0a3j086jzjxud --discovery-token-ca-cert-hash sha256:9b1b55035c2671b63635f4dbcc218c499d09d6e3319992d954754832ed988fe2 
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
**Step 3: Execute below command in Loadbalancer-1.**
```
ip a s
```
This will show the virtual IP 172.17.17.116 is attached to Loadbalancer-1.

**Step 3: Turn off the Loadbalancer-1 and execute below command in Loadbalancer-2.**
```
ip a s
```
Now, this will show the virtual IP 172.17.17.116 is attached to Loadbalancer-2. <br/>

