# K8-Task
Task to build a Kubernetes cluster and connect it to a private docker registry.
### Nodes are set up using kubeadm, using Ubuntu 20.0.04 with keepalivd and haproxy

# Vagrant Environemt
Load Balancer 1 - IP: 172.16.16.51
Load Balancer 2 - IP: 172.16.16.52
Master 1 - IP: 172.16.16.101
Master 2 - IP: 172.16.16.102
Master 3 - IP: 172.16.16.103
Worker 1 - IP: 172.16.16.201
Worker 2 - IP: 172.16.16.202
* Password for the roor account on all the vms is "kubeadmin" *
## Virtual IP managed by Keepalived on the load balancer nodes

## Prerequisits to run this on local env
Virtualbox
Vagrant
-------------------
```
vagrant up
```
```
apt update && apt install -y keepalived haproxy
```
##### configure keepalived
On both nodes create the health check script /etc/keepalived/check_apiserver.sh
```
cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.16.16.100; then
  curl --silent --max-time 2 --insecure https://172.16.16.100:6443/ -o /dev/null || errorExit "Error GET https://172.16.16.100:6443/"
fi
EOF

chmod +x /etc/keepalived/check_apiserver.sh
```
Create keepalived config /etc/keepalived/keepalived.conf
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
    interface eth1
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        172.16.16.100
        172.16.16.101
    }
    track_script {
        check_apiserver
    }
}
EOF
```
##### Enable & start keepalived service
```
systemctl enable --now keepalived
```
##### Configure haproxy
Update **/etc/haproxy/haproxy.cfg**
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
    server kmaster1 172.16.16.101:6443 check fall 3 rise 2
    server kmaster2 172.16.16.102:6443 check fall 3 rise 2
    server kmaster3 172.16.16.103:6443 check fall 3 rise 2

EOF
```
##### Enable & restart haproxy service
```
systemctl enable haproxy && systemctl restart haproxy
```
## For worker and master nodes:
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Disable Firewall
```
systemctl disable --now ufw
```
##### Enable and Load Kernel modules
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
##### Add Kernel settings
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
##### Install containerd runtime
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
##### Add apt repo for kubernetes
```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
}
```
##### Install Kubernetes components
```
{
  apt update
  apt install -y kubeadm=1.22.0-00 kubelet=1.22.0-00 kubectl=1.22.0-00
}
```
## Bootstrap the cluster
## On node kmaster1
##### Initialize Kubernetes Cluster
```
kubeadm init --control-plane-endpoint="172.16.16.100:6443" --upload-certs --apiserver-advertise-address=172.16.16.101 --pod-network-cidr=192.168.0.0/16
```
Copy the commands to join other master nodes and worker nodes.
##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml
```

## Join other master nodes to the cluster

## Join worker nodes to the cluster

## Downloading kube config
```
mkdir ~/.kube
scp root@172.16.16.101:/etc/kubernetes/admin.conf ~/.kube/config
```
And that's it! The cluster should be running, we can check with kubectl get nodes of course.
