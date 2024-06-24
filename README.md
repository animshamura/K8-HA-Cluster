## Highly Available Kubernetes Cluster Architecture: ##
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/ha-kube-cluster.drawio.png?raw=true">

## Requirements: ##
- Minimum 3 Master Nodes
- Minimum 2 Loadbalancers
- HAProxy
- Keepalived
## Section: ##
- VM Creation
- HAProxy and Keepalived Installation in Loadbalancer VMs
- Setup Multi Cluster 

## VM Creation: ##

#### Step 1: Create 6 VMs in Proxmox. ####

<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr1.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr2.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr3.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr4.png?raw=true">

***Create one VM and clone 5 VMs to get total of 6 VMs in Proxmox. Again, hostname and IP address of each VM has to be edited.***

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
## HAProxy and Keepalives Installation in Loadbalancer VMs: ##

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
