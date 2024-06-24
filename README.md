# Highly-Available-Kubernetes-Cluster
## Requirements: ##
- Minimum 3 master nodes
- Minimum 2 loadbalancers
- HAProxy
- Keepalived

## VM Creation: ##

#### Step 1: Create 6 VMs in Proxmox. ####

<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr1.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr2.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr3.png?raw=true">
<img src="https://github.com/animshamura/Highly-Available-Kubernetes-Cluster/blob/main/screenshots/pr4.png?raw=true">
<br/>

***Create one VM and clone 5 VMs to get total 0f 6 VMs in Proxmox. Furthermore, hostname and IP address of each VM has to be edited.***

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

