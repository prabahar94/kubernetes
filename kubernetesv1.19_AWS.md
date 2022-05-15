## [**Set up a Highly Available Kubernetes Cluster using kubeadm in AWS EC2**](https://gitlab.com/prabahar941/kubernetes/-/edit/main/kubernetes-HA/README.md)
---

This documentation will allow to set up a highly available Kubernetes cluster using Ubuntu 20.04 LTS in AWS EC2

Setting up cluster with three master nodes , two worker nodes , loadbalancer (HAproxy)

**CAUTION: THIS IS ONLY FOR TEST ENVIRONMENT SETUP**

**Environment**
---

| Role | Hostname | IP | OS version | CPU | RAM | Instance type |
| ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| Master1 | kubernetes-master1 | 172.31.21.100 | Ubuntu 20.04 | 2 | 2 | t3.small |
| Master2 | kubernetes-master2 | 172.31.21.101 | Ubuntu 20.04 | 2 | 2 | t3.small |
| Master3 | kubernetes-master3 | 172.31.21.102 | Ubuntu 20.04 | 2 | 2 | t3.small |
| Worker1 | kubernetes-worker1 | 172.31.7.64 | Ubuntu 20.04 | 1 | 1 | t2.micro |
| Worker2 | kubernetes-worker2 | 172.31.7.65 | Ubuntu 20.04 | 1 | 1 | t2.micro |
| Loadbalancer | Loadbalancer | 172.31.6.214 | Ubuntu 20.04 | 1 | 1 | t2.micro |

- Perform all action as root user .

Note: IP address mentioned here are private ip once the intance were created . 

## **Pre-Requisites**
---
- AWS account 
- ssh client installed in local machine (Putty / mobixterm)
- Basic knowledge of working with AWS Console (To Launch EC2 Instance)

**IMPORTANT: while launching EC2 Instance make sure to  open the required ports in security groups**

For Reference please refer the below table 

| Type | Protocol | Port Range | source | Description |
| ------ | ------ | ------ | ------ | ------ |
| Custom TCP | TCP | 30000 - 32767 | 0.0.0.0 | Node Port Service |
| Custom TCP | TCP | 10240 - 10260 | 0.0.0.0 | Kubecontroller components | 
| Custom TCP | TCP | 6443 | 0.0.0.0 | Kube-API service|        
| Custom TCP | TCP | 2379 - 2380 | 0.0.0.0 | ETCD service|
| SSH | TCP | 22 | 0.0.0.0 | SSH port |                 
| All ICMP - IPv4 | ICMP | ALL | 0.0.0.0 | Check the connectivity using Ping |     

Once the EC2 are provisioned Add the entries to the /etc/hosts file
```
172.31.21.100 kubernetes-master1
172.31.21.101 kubernetes-master2
172.31.21.102 kubernetes-master3
172.31.7.64   kubernetes-worker1
172.31.7.65   kubernetes-worker2
172.31.6.214  Loadbalancer

```
Note: Modify the IPs based on your Resource allocation 

## Set up load balancer 
##### Install Haproxy
```
apt update && apt install -y haproxy
```
## Configuring HA Proxy
Add below lines to **/etc/haproxy/haproxy.cfg**
```
frontend kubernetes-frontend
    bind 172.31.6.214:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server kubernetes-master1 172.31.21.100:6443 check fall 3 rise 2
    server kubernetes-master2 172.31.21.101:6443 check fall 3 rise 2
    server kubernetes-master3 172.31.21.102:6443 check fall 3 rise 2

```

##### Restart haproxy service
```
systemctl restart haproxy
```
##### Check haproxy service

Make sure the status of haproxy is active 
```
systemctl status haproxy
```

## Configuring all kubernetes nodes (kubernetes-master1,kubernetes-master2,kubernetes-master3,kubernetes-worker1,kubernetes-worker2)
##### Disable Firewall
```
ufw disable
```

##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
##### Install docker engine
```
{
  apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  apt update && apt install -y docker-ce=5:19.03.10~3-0~ubuntu-focal containerd.io
}
```
### Kubernetes Setup
##### Add Apt repository
```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
}
```
##### Install Kubernetes components
```
apt update && apt install -y kubeadm=1.19.2-00 kubelet=1.19.2-00 kubectl=1.19.2-00
```
## On any one of the Kubernetes master node (Eg: kubernetes-master1)
##### Initialize Kubernetes Cluster
```
kubeadm init --control-plane-endpoint="172.31.6.214:6443" --upload-certs --apiserver-advertise-address=172.31.21.100 --pod-network-cidr=192.168.0.0/16
```
Copy the commands to join other master nodes and worker nodes.

## Join other nodes to the cluster (kubernetes-master2 , kubernetes-master3 ,kubernetes-worker1 ,kubernetes-worker2)
> Use the respective kubeadm join commands  from the output of kubeadm init command on the first master.

> IMPORTANT: To add master node to the cluster we need to pass argument --apiserver-advertise-address <ip-address> to the join cluster.

Example for addding kubernetes-master2 - 172.31.21.101

```
kubeadm join 172.31.6.214:6443 --token *********** \
    --discovery-token-ca-cert-hash ************************* \
    --control-plane --certificate-key *********************** \
    ----apiserver-advertise-address 172.31.21.101  # Added the ip address of node to be join as master
```

##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```

## Downloading kube config to  local machine
On your host machine

copy content of **/etc/kubernetes/admin.conf** from kubernetes-master1 - 172.31.21.100 and paste into **~/.kube/config** of local machine 

## Verifying the cluster
```
kubectl cluster-info
kubectl get nodes
kubectl get cs
```










