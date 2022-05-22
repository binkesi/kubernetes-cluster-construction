## Build High available Kubernetes cluster

Record process of building high avilable k8s cluster.  
Also the problems I met and how to resolve the problems.

### Overview

For me the best practise to learn kubernetes is to build a HA kubernetes cluster step by step. And try any experiment on it.

### Prerequisite

We need a virtualization platform to set up seven VMs. Three master nodes, three worker nodes and one Load Balance node with
HAproxy and Keepalived installed.

- VMs plan:

| IP address | Hostname | VM resource | VM role | Software installed | OS installed |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 33.193.255.121 | master-lb | 2 core, 8G | Load Balance | HAproxy, Keepalived | Centos8 |
| 33.193.255.122 | master-01 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.123 | master-02 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.124 | master-03 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.125 | worker-01 | 2 core, 8G | Worker | kubelet, kube-proxy, core-dns | Centos8 |
| 33.193.255.126 | worker-02 | 2 core, 8G | Worker | kubelet, kube-proxy, core-dns | Centos8 |
| 33.193.255.127 | worker-03 | 2 core, 8G | Worker | kubelet, kube-proxy, core-dns | Centos8 |

- Software plan:

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

- Network plan:

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

    At boot time a window **Shim UEFI key management** gets displayed **for 10 seconds only**   
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
    ```

### Step 2. Deploy master nodes

1. Configure HAproxy and Keepalived 

    - On master-lb and master-01
    ```shell
    yum install keepalived haproxy -y
    vi /etc/haproxy/haproxy.cfg  
    global
    maxconn 2000
    ulimit-n 16384
    log 127.0.0.1 local0 err
    stats timeout 30s  
    defaults
    log global
    mode http
    option httplog
    timeout connect 5000
    timeout client 50000
    timeout server 50000
    timeout http-request 15s
    timeout http-keep-alive 15s  
    frontend monitor-in
    bind *:33305
    mode http
    option httplog
    monitor-uri /monitor  
    frontend k8s-master
    bind 0.0.0.0:16443
    bind 127.0.0.1:16443
    mode tcp
    option tcplog
    tcp-request inspect-delay 5s
    default_backend k8s-master  
    backend k8s-master
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server  master-01  33.193.255.122:6443 check
    server  master-02  33.193.255.123:6443 check
    server  master-03  33.193.255.124:6443 check
    ```

    - On master-01, master-02, master-03
    ```shell
    yum install keepalived haproxy -y
    vi /etc/keepalived/keepalived.conf (notice should change "mcast_src_ip" on each node)  
    ! Configuration File for keepalived
    global_defs {
        router_id LVS_DEVEL
        script_user root
        enable_script_security
        }
        vrrp_script chk_apiserver {
        script "/etc/keepalived/check_apiserver.sh"
        interval 5
        weight -5
        fall 2
        rise 1 # test once success, consider it's alive
        }
        vrrp_instance VI_1 {
        state BACKUP
        nopreempt
        interface ens192
        mcast_src_ip 33.193.255.122
        virtual_router_id 51
        priority 100
        advert_int 2
        authentication {
            auth_type PASS
            auth_pass K8SHA_KA_AUTH
        }
        virtual_ipaddress {
            33.193.255.121
        }
        track_script {
            chk_apiserver
        }
    } 
    ```

    - Health check script on master-01, master-02, master-03
    ```shell
    vi /etc/keepalived/check_apiserver.sh  
    #!/bin/bash
    err=0
    for k in $(seq 1 3)
    do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
    done
    ​
    if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
    else
    exit 0
    fi
    EOF     
    chmod u+x /etc/keepalived/check_apiserver.sh
    ```

2. Configure ssh on master nodes

    - Configure ssh without password (on three master nodes)
    ```shell
    cd /root
    ssh-keygen -t rsa
    for i in master-02 master-03 worker-01 worker-02 worker-03;do ssh-copy-id $i;done
    ```

    - Start HAproxy and Keepalived (on master-lb, master-01, master-02, master-03)
    ```shell
    systemctl daemon-reload
    systemctl enable --now haproxy
    systemctl enable --now keepalived
    ```

3. Prepare Etcd certificates

    - Download ssl certificate tool (on master-01)
    ```shell
    mkdir -p /data/k8s-work  # Create work dir
    cd /data/k8s-work
    wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
    wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
    wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
    chmod +x cfssl*
    mv cfssl_linux-amd64 /usr/local/bin/cfssl
    mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
    mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
    ```

    - Configure ca request file
    ```shell
    vi ca-csr.json  
    {
        "CN": "kubernetes",
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
            "C": "CN",
            "ST": "Shanghai",
            "L": "pudong",
            "O": "k8s",
            "OU": "system"
            }
        ],
        "ca": {
                "expiry": "87600h"
        }
    }  
    ```
    
    - Create ca certificate
    ```shell
    cfssl gencert -initca ca-csr.json | cfssljson -bare ca
    ```

    - Create ca configuration
    ```shell
    vi ca-config.json  
    {
        "signing": {
            "default": {
                "expiry": "87600h"
                },
            "profiles": {
                "kubernetes": {
                    "usages": [
                        "signing",
                        "key encipherment",
                        "server auth",
                        "client auth"
                    ],
                    "expiry": "87600h"
                }
            }
        }
    }  
    ``` 

    - Create Etcd csr file
    ```shell
    vi etcd-csr.json
    {
        "CN": "etcd",
        "hosts": [
            "127.0.0.1",
            "33.193.255.122",
            "33.193.255.123",
            "33.193.255.124"
        ],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [{
            "C": "CN",
            "ST": "Shanghai",
            "L": "pudong",
            "O": "k8s",
            "OU": "system"
        }]
    }   
    ```

    - Generate Etcd certificates
    ```shell
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson  -bare etcd
    ```

4. Deploy Etcd cluster

    - Download and copy Etcd packages to each master node
    ```shell
    wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
    tar -xvf etcd-v3.5.0-linux-amd64.tar.gz
    cp -p etcd-v3.5.0-linux-amd64/etcd* /usr/local/bin/
    scp  etcd-v3.5.0-linux-amd64/etcd* master-02:/usr/local/bin/
    scp  etcd-v3.5.0-linux-amd64/etcd* master-03:/usr/local/bin/
    ```

    - Create Etcd configure file
    ```shell
    vi etcd.conf
    #[Member]
    ETCD_NAME="etcd1"
    ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
    ETCD_LISTEN_PEER_URLS="https://33.193.255.122:2380"
    ETCD_LISTEN_CLIENT_URLS="https://33.193.255.122:2379,http://127.0.0.1:2379"  
    #[Clustering]
    ETCD_INITIAL_ADVERTISE_PEER_URLS="https://33.193.255.122:2380"
    ETCD_ADVERTISE_CLIENT_URLS="https://22.193.255.122:2379"
    ETCD_INITIAL_CLUSTER="etcd1=https://33.193.255.122:2380,etcd2=https://33.193.255.123:2380,etcd3=https://33.193.255.124:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
    ETCD_INITIAL_CLUSTER_STATE="new"  # new -- new cluster, existing -- exist cluster
    ```

    - Create service file
    ```shell
    vi etcd.service  
    [Unit]
    Description=Etcd Server
    After=network.target
    After=network-online.target
    Wants=network-online.target  
    [Service]
    Type=notify
    EnvironmentFile=-/etc/etcd/etcd.conf
    WorkingDirectory=/var/lib/etcd/
    ExecStart=/usr/local/bin/etcd \
    --cert-file=/etc/etcd/ssl/etcd.pem \
    --key-file=/etc/etcd/ssl/etcd-key.pem \
    --trusted-ca-file=/etc/etcd/ssl/ca.pem \
    --peer-cert-file=/etc/etcd/ssl/etcd.pem \
    --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
    --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=65536  ​
    [Install]
    WantedBy=multi-user.target 
    ```

    - Create Etcd directory on each master node, copy cert and configure files to each master node(change ip in each config file)
    ```shell
    mkdir -p /etc/etcd
    mkdir -p /etc/etcd/ssl
    mkdir -p /var/lib/etcd/default.etcd
    cp ca*.pem /etc/etcd/ssl/
    cp etcd*.pem /etc/etcd/ssl/
    cp etcd.conf /etc/etcd/
    cp etcd.service /usr/lib/systemd/system/
    for i in master-02 master-03;do scp  etcd.conf $i:/etc/etcd/;done
    for i in master-02 master-03;do scp  etcd*.pem ca*.pem $i:/etc/etcd/ssl/;done
    for i in master-02 master-03;do scp  etcd.service $i:/usr/lib/systemd/system/;done
    ```

    - Start Etcd cluster on each master nodes
    ```shell
    systemctl daemon-reload
    systemctl enable --now etcd.service
    systemctl status etcd
    ```

    - Check Etcd cluster status
    ```shell
    ETCD_API=3 etcdctl --write-out=table \
    --cacert=/etc/etcd/ssl/ca.pem \
    --cert=/etc/etcd/ssl/etcd.pem \
    --key=/etc/etcd/ssl/etcd-key.pem \
    --endpoints=https://33.193.255.122:2379,https://33.193.255.123:2379,https://33.193.255.124:2379 endpoint health
    ```

5. Deploy k8s master components

    + Deploy api server

        - Download and distribute k8s packages(on master-01)
        ```shell
        wget https://dl.k8s.io/v1.22.1/kubernetes-server-linux-amd64.tar.gz
        tar -xvf kubernetes-server-linux-amd64.tar.gz
        cd kubernetes/server/bin/
        cp kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
        scp   kube-apiserver kube-controller-manager kube-scheduler kubectl master-02:/usr/local/bin/
        scp   kube-apiserver kube-controller-manager kube-scheduler kubectl master-03:/usr/local/bin/
        for i in worker-01 worker-02 ;do scp  kubelet kube-proxy $i:/usr/local/bin/;done
        ```

        - Create work dir on all nodes
        ```shell
        mkdir -p /etc/kubernetes/        
        mkdir -p /etc/kubernetes/ssl     
        mkdir -p /var/log/kubernetes
        ```

        - Create api server csr file(on master-01)
        ```shell
        cat > kube-apiserver-csr.json << "EOF"
        {
            "CN": "kubernetes",
            "hosts": [
                "127.0.0.1",
                "33.193.255.121",
                "33.193.255.122",
                "33.193.255.123",
                "33.193.255.124",
                "33.193.255.125",
                "33.193.255.126",
                "33.193.255.127",
                "33.193.255.128",
                "33.193.255.129",
                "33.193.255.131",
                "33.193.255.132",
                "33.193.255.133",
                "33.193.255.134",
                "33.193.255.135",
                "10.96.0.1",
                "kubernetes",
                "kubernetes.default",
                "kubernetes.default.svc",
                "kubernetes.default.svc.cluster",
                "kubernetes.default.svc.cluster.local"
            ],
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
                {
                "C": "CN",
                "ST": "Shanghai",
                "L": "pudong",
                "O": "k8s",
                "OU": "system"
                }
            ]
        }
        EOF
        ```

        - Generate certificate and token for api server(on master-01)
        ```shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
        cat > token.csv << EOF
        $(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
        EOF
        ```

        - Create api server configure file
        ```shell
        cat > kube-apiserver.conf << "EOF"
        KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
        --anonymous-auth=false \
        --bind-address=33.193.255.122 \
        --secure-port=6443 \
        --advertise-address=33.193.255.122 \
        --insecure-port=0 \
        --authorization-mode=Node,RBAC \
        --runtime-config=api/all=true \
        --enable-bootstrap-token-auth \
        --service-cluster-ip-range=10.96.0.0/16 \
        --token-auth-file=/etc/kubernetes/token.csv \
        --service-node-port-range=30000-50000 \
        --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem  \
        --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
        --client-ca-file=/etc/kubernetes/ssl/ca.pem \
        --kubelet-client-certificate=/etc/kubernetes/ssl/kube-apiserver.pem \
        --kubelet-client-key=/etc/kubernetes/ssl/kube-apiserver-key.pem \
        --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
        --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
        --service-account-issuer=api \
        --etcd-cafile=/etc/etcd/ssl/ca.pem \
        --etcd-certfile=/etc/etcd/ssl/etcd.pem \
        --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
        --etcd-servers=https://33.193.255.122:2379,https://33.193.255.123:2379,https://33.193.255.124:2379 \
        --enable-swagger-ui=true \
        --allow-privileged=true \
        --apiserver-count=3 \
        --audit-log-maxage=30 \
        --audit-log-maxbackup=3 \
        --audit-log-maxsize=100 \
        --audit-log-path=/var/log/kube-apiserver-audit.log \
        --event-ttl=1h \
        --alsologtostderr=true \
        --logtostderr=false \
        --log-dir=/var/log/kubernetes \
        --v=4"
        EOF
        ```

        - Create api server service file
        ```shell
        cat > kube-apiserver.service << "EOF"
        [Unit]
        Description=Kubernetes API Server
        Documentation=https://github.com/kubernetes/kubernetes
        After=etcd.service
        Wants=etcd.service  
        [Service]
        EnvironmentFile=-/etc/kubernetes/kube-apiserver.conf
        ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
        Restart=on-failure
        RestartSec=5
        Type=notify
        LimitNOFILE=65536  
        [Install]
        WantedBy=multi-user.target
        EOF
        ```

        - Sync certificate and api configure files to each master nodes and modify IP in config files
        ```shell
        cp ca*.pem /etc/kubernetes/ssl/
        cp kube-apiserver*.pem /etc/kubernetes/ssl/
        cp token.csv /etc/kubernetes/
        cp kube-apiserver.conf /etc/kubernetes/ 
        cp kube-apiserver.service /usr/lib/systemd/system/
        scp   token.csv master-02:/etc/kubernetes/
        scp   token.csv master-03:/etc/kubernetes/
        scp   kube-apiserver*.pem master-02:/etc/kubernetes/ssl/
        scp   kube-apiserver*.pem master-03:/etc/kubernetes/ssl/
        scp   ca*.pem master-02:/etc/kubernetes/ssl/
        scp   ca*.pem master-03:/etc/kubernetes/ssl/
        scp   kube-apiserver.conf master-02:/etc/kubernetes/
        scp   kube-apiserver.conf master-03:/etc/kubernetes/
        scp   kube-apiserver.service master-02:/usr/lib/systemd/system/
        scp   kube-apiserver.service master-03:/usr/lib/systemd/system/
        vim kube-apiserver.conf
        ```

        - Start api server on each master nodes
        ```shell
        systemctl daemon-reload
        systemctl enable --now kube-apiserver
        systemctl status kube-apiserver
        ```

        - Create admin csr file for kubelet
        ```shell
        cat > admin-csr.json << "EOF"
        {
            "CN": "admin",
            "hosts": [],
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
                {
                "C": "CN",
                "ST": "Shanghai",
                "L": "pudong",
                "O": "system:masters",             
                "OU": "system"
                }
            ]
        }
        EOF 
        ``` 

    + Deploy kubectl
        
        - Generate certificate
        ```shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
        cp admin*.pem /etc/kubernetes/ssl/
        ``` 

        - Configure kube config
        ```shell
        kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://33.193.255.121:16443 --kubeconfig=kube.config
        kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config
        kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config
        kubectl config use-context kubernetes --kubeconfig=kube.config
        mkdir ~/.kube
        cp kube.config ~/.kube/config
        kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes --kubeconfig=~/.kube/config
        ```

        - Check cluster status and scp kubectl config file to other master nodes
        ```shell
        export KUBECONFIG=$HOME/.kube/config
        kubectl cluster-info
        kubectl get componentstatuses
        kubectl get all --all-namespaces
        scp    /root/.kube/config master-02:/root/.kube/
        scp    /root/.kube/config master-03:/root/.kube/
        ```

        - Configure kubectl task completion
        ```shell
        yum install -y bash-completion
        source /usr/share/bash-completion/bash_completion
        source <(kubectl completion bash)
        kubectl completion bash > ~/.kube/completion.bash.inc
        source '/root/.kube/completion.bash.inc'  
        source $HOME/.bash_profile
        ```

    + Deploy kube-controller-manager  

        - Create csr file
        ```shell
        cat > kube-controller-manager-csr.json << "EOF"
        {
            "CN": "system:kube-controller-manager",
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "hosts": [
                "127.0.0.1",
                "33.193.255.122",
                "33.193.255.123",
                "33.193.255.124"
            ],
            "names": [
                {
                    "C": "CN",
                    "ST": "Shanghai",
                    "L": "pudong",
                    "O": "system:kube-controller-manager",
                    "OU": "system"
                }
            ]
        }
        EOF
        ```

        - Generate certificate file
        ```shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
        ls kube-controller-manager*.pem
        ```

        - Generate kube-controller-manager.kubeconfig
        ```shell
        kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://33.193.255.121:16443 --kubeconfig=kube-controller-manager.kubeconfig
        kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
        kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
        kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
        ```

        - Create kube-controller-manager.conf
        ```shell
        cat > kube-controller-manager.conf << "EOF"
        KUBE_CONTROLLER_MANAGER_OPTS="--port=0 \
        --secure-port=10257 \
        --bind-address=127.0.0.1 \
        --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \
        --service-cluster-ip-range=10.96.0.0/16 \
        --cluster-name=kubernetes \
        --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
        --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
        --allocate-node-cidrs=true \
        --cluster-cidr=172.168.0.0/16 \
        --experimental-cluster-signing-duration=87600h \
        --root-ca-file=/etc/kubernetes/ssl/ca.pem \
        --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
        --leader-elect=true \
        --feature-gates=RotateKubeletServerCertificate=true \
        --controllers=*,bootstrapsigner,tokencleaner \
        --horizontal-pod-autoscaler-sync-period=10s \
        --tls-cert-file=/etc/kubernetes/ssl/kube-controller-manager.pem \
        --tls-private-key-file=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
        --use-service-account-credentials=true \
        --alsologtostderr=true \
        --logtostderr=false \
        --log-dir=/var/log/kubernetes \
        --v=2"
        EOF   
        ```

        - Create kube-controller-manager.service
        ```shell
        cat > kube-controller-manager.service << "EOF"
        [Unit]
        Description=Kubernetes Controller Manager
        Documentation=https://github.com/kubernetes/kubernetes​  
        [Service]
        EnvironmentFile=-/etc/kubernetes/kube-controller-manager.conf
        ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
        Restart=on-failure
        RestartSec=5  
        [Install]
        WantedBy=multi-user.target
        EOF 
        ```

        - Copy files to each master nodes
        ```shell
        cp kube-controller-manager*.pem /etc/kubernetes/ssl/
        cp kube-controller-manager.kubeconfig /etc/kubernetes/
        cp kube-controller-manager.conf /etc/kubernetes/
        cp kube-controller-manager.service /usr/lib/systemd/system/
        scp  kube-controller-manager*.pem master-02:/etc/kubernetes/ssl/
        scp  kube-controller-manager*.pem master-03:/etc/kubernetes/ssl/
        scp  kube-controller-manager.kubeconfig kube-controller-manager.conf master-02:/etc/kubernetes/
        scp  kube-controller-manager.kubeconfig kube-controller-manager.conf master-03:/etc/kubernetes/
        scp  kube-controller-manager.service master-02:/usr/lib/systemd/system/
        scp  kube-controller-manager.service master-03:/usr/lib/systemd/system/
        ```

        - Start kube controller manager service
        ```shell
        systemctl daemon-reload 
        systemctl enable --now kube-controller-manager
        systemctl status kube-controller-manager  
        ```

    + Deploy kube scheduler

        - Create csr file
        ```shell
        cat > kube-scheduler-csr.json << "EOF"
        {
            "CN": "system:kube-scheduler",
            "hosts": [
            "127.0.0.1",
            "33.193.255.122",
            "33.193.255.123",
            "33.193.255.124"
            ],
            "key": {
                "algo": "rsa",
                "size": 2048
            },
            "names": [
            {
                "C": "CN",
                "ST": "Shanghai",
                "L": "pudong",
                "O": "system:kube-scheduler",
                "OU": "system"
            }
            ]
        }
        EOF 
        ```  

        - Generate certificate
        ```shell
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-schedule
        ls kube-scheduler*.pem
        ```

        - Generate kubeconfig for kube-scheduler
        ```shell
        kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://33.193.255.121:16443 --kubeconfig=kube-scheduler.kubeconfig
        kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
        kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
        kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
        ```

        - Create kube-scheduler.conf
        ```shell
        cat > kube-scheduler.conf << "EOF"
        KUBE_SCHEDULER_OPTS="--address=127.0.0.1 \
        --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
        --leader-elect=true \
        --alsologtostderr=true \
        --logtostderr=false \
        --log-dir=/var/log/kubernetes \
        --v=2"
        EOF
        ```

        - Create kube-scheduler.service
        ```shell
        cat > kube-scheduler.service << "EOF"
        [Unit]
        Description=Kubernetes Scheduler
        Documentation=https://github.com/kubernetes/kubernetes
        [Service]
        EnvironmentFile=-/etc/kubernetes/kube-scheduler.conf
        ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
        Restart=on-failure
        RestartSec=5
        [Install]
        WantedBy=multi-user.target
        EOF
        ```

        - Copy files to each master node
        ```shell
        cp kube-scheduler*.pem /etc/kubernetes/ssl/
        cp kube-scheduler.kubeconfig /etc/kubernetes/
        cp kube-scheduler.conf /etc/kubernetes/
        cp kube-scheduler.service /usr/lib/systemd/system/
        scp  kube-scheduler*.pem master-02:/etc/kubernetes/ssl/
        scp  kube-scheduler*.pem master-03:/etc/kubernetes/ssl/
        scp  kube-scheduler.kubeconfig kube-scheduler.conf master-02:/etc/kubernetes/
        scp  kube-scheduler.kubeconfig kube-scheduler.conf master-03:/etc/kubernetes/
        scp  kube-scheduler.service master-02:/usr/lib/systemd/system/
        scp  kube-scheduler.service master-03:/usr/lib/systemd/system/
        ```

        - Start kube-scheduler service
        ```shell
        systemctl daemon-reload
        systemctl enable --now kube-scheduler
        systemctl status kube-scheduler
        ```
