## Highly Available Kubernetes Cluster Architecture: ##
The below architecture refers a highly available Kubernetes cluster which doesn't fall in one point failure for loadbalancing.
<br/>
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/ha-kube-cluster.drawio.png?raw=true">

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
***Edit "00-installer-config.yaml" file according to the desired IP.***

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
***Create keepalived config /etc/keepalived/keepalived.conf.***

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
**Step 2: Join master and worker nodes.** <br/>
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
kubeadm join 172.17.17.110:6443 --token 5g5jo2.agl26wfzkujgjt3s --discovery-token-ca-cert-hash ha256:57795a664200425258ed0619af960fe476d1ae93f99182a3d710ce1185468d3f --apiserver-advertise-address=172.17.17.113
```
**Step 3: Deploy Calico network.**
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml
```

