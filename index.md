## Build High available Kubernetes cluster

Record process of building high avilable k8s cluster.
Also the problems I met and how to resolve the problems.

### Overview

For me the best practise to learn kubernetes is to build a HA kubernetes cluster step by step. And try any experiment on it.

### Prerequisite

We need a virtualization platform to set up seven VMs. Three master nodes, three worker nodes and one Load Balance node with
HAproxy and Keepalived installed.

VMs plan:

| IP address | Hostname | VM resource | VM role | Software installed | OS installed |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 33.193.255.121 | master-lb | 2 core, 8G | Load Balance | HAproxy, Keepalived | Centos8 |
| 33.193.255.122 | master-01 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.123 | master-02 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.124 | master-03 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.125 | worker-01 | 2 core, 8G | Worker | kubelet, kube-proxy, core-dns | Centos8 |
| 33.193.255.126 | worker-02 | 2 core, 8G | Worker | kubelet, kube-proxy, core-dns | Centos8 |
| 33.193.255.127 | worker-03 | 2 core, 8G | Worker | kubelet, kube-proxy, core-dns | Centos8 |

Software plan:

| Software | Version |
| :---- | :---- |
| centos | v8.3.2011 |
| kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy | v1.22.1 |
| etcd | v3.5.0 |
| calico | v3.19.1 |
| coredns | v1.8.4 |
| docker | v20.10.8 |
| haproxy | v1.5.18 |
| keepalived | v1.3.5 |

Network plan:

| Network | Allocation |
| :---- | :---- |
| Pod network | 172.168.0.0/12 |
| Service network | 10.96.0.0/16 |

### Step 1. Prepare base VM

1. Set up a base Centos8 VM for clone which can access Internet. (deploy ova template on Vcenter)
2. Do some system service configuration
- Disable firewall and selinux
```
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0 

- Close swap off
```
swapoff -a
vi /etc/fstab
#/dev/mapper/centos-swap swap                    swap    defaults        0 0

- Add hosts information
```
vi /etc/hosts
33.193.255.121 master-lb
33.193.255.122 master-01
33.193.255.123 master-02
33.193.255.124 master-03
33.193.255.125 worker-01
33.193.255.126 worker-02
33.193.255.127 worker-03

- Make Centos8 yum avaliable
```
wget 'http://mirror.centos.org/centos/8-stream/BaseOS/x86_64/os/Packages/centos-gpg-keys-8-3.el8.noarch.rpm'
sudo rpm -i 'centos-gpg-keys-8-3.el8.noarch.rpm'
dnf --disablerepo '*' --enablerepo=extras swap centos-linux-repos centos-stream-repos
sudo dnf distro-sync

- Sync system time
```
yum install chrony -y
systemctl enable chronyd
systemctl start chronyd
