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
```shell
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0 
```

- Close swap off
```shell
swapoff -a
vi /etc/fstab
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

- Add hosts information
```shell
vi /etc/hosts
33.193.255.121 master-lb
33.193.255.122 master-01
33.193.255.123 master-02
33.193.255.124 master-03
33.193.255.125 worker-01
33.193.255.126 worker-02
33.193.255.127 worker-03
```

- Make Centos8 yum avaliable
```shell
wget 'http://mirror.centos.org/centos/8-stream/BaseOS/x86_64/os/Packages/centos-gpg-keys-8-3.el8.noarch.rpm'
sudo rpm -i 'centos-gpg-keys-8-3.el8.noarch.rpm'
dnf --disablerepo '*' --enablerepo=extras swap centos-linux-repos centos-stream-repos
sudo dnf distro-sync
```

- Sync system time
```shell
yum install chrony -y
systemctl enable chronyd
systemctl start chronyd
```

3. Upgrade linux kernel

- Import elrepo yum resource
```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm -y
```

- Install new version kernel
```shell
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
yum --enablerepo=elrepo-kernel install kernel-ml -y
```

- Configure shim allow and grub
```shell
yum install pesign -y
pesign -P -h -i /boot/vmlinuz-<version>
mokutil --import-hash <hash value returned from pesign>
input password: somepass
input password again: somepass
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```

At boot time a window **Shim UEFI key management** gets displayed **for 10 seconds only** (unless step 2. has been followed)
Hit a key to enter the **Perform MOK management** menu
Choose **Enroll MOK**
Choose **Continue**
Choose **Yes** in **Enroll the key** menu
Type the password used while executing the `mokutil --import-hash` command
Choose **Reboot** to reboot

- Remove old kernel
```shell
rpm -qa | grep kernel
yum remove -y kernel-core-4.18.0 kernel-devel-4.18.0 kernel-tools-libs-4.18.0 kernel-headers-4.18.0
```

- Optimize kernel parameter
```shell
echo "* soft nofile 655360" >> /etc/security/limits.conf
echo "* hard nofile 655360" >> /etc/security/limits.conf
echo "* soft nproc 655360" >> /etc/security/limits.conf
echo "* hard nproc 655360" >> /etc/security/limits.conf
echo "* soft memlock unlimited" >> /etc/security/limits.conf
echo "* hard memlock unlimited" >> /etc/security/limits.conf
echo "DefaultLimitNOFILE=1024000" >> /etc/systemd/system.conf
echo "DefaultLimitNPROC=1024000" >> /etc/systemd/system.conf
```

- Optimize kernel for k8s
```shell
vi /etc/sysctl.d/k8s.conf

net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 131072
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF

sysctl --system
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git lrzsz -y
```

4. Load ipvs

- Install required tools
```shell
yum install ipvsadm ipset sysstat conntrack libseccomp -y
```

- Configure ipvs module
```shell
modprobe -- ip_vs 
modprobe -- ip_vs_rr 
modprobe -- ip_vs_wrr 
modprobe -- ip_vs_sh 
modprobe -- nf_conntrack 
```

- Create ipvs configure file
```shell
vi /etc/modules-load.d/ipvs.conf

ip_vs 
ip_vs_lc 
ip_vs_wlc 
ip_vs_rr 
ip_vs_wrr 
ip_vs_lblc 
ip_vs_lblcr 
ip_vs_dh 
ip_vs_sh 
ip_vs_fo 
ip_vs_nq 
ip_vs_sed 
ip_vs_ftp 
ip_vs_sh 
nf_conntrack 
ip_tables 
ip_set 
xt_set 
ipt_set 
ipt_rpfilter 
ipt_REJECT 
ipip
EOF
```

### Step 2. Deploy master nodes