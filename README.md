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
