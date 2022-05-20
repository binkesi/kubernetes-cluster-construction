## Build High available Kubernetes cluster

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Overview

For me the best practise to learn kubernetes is to build a HA kubernetes cluster step by step. And try any experiment on it.

### Prerequisite

We need a virtualization platform to set up seven VMs. Three master nodes, three worker nodes and one Load Balance node with
HAproxy and Keepalived installed.

VMs plan:

| IP address | hostname | VM resource | VM role | software installed | OS installed |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 33.193.255.121 | master-lb | 2 core, 8G | Load Balance | HAproxy, Keepalived | Centos8 |
| 33.193.255.122 | master-01 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, \
kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.123 | master-02 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, \
kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.124 | master-03 | 2 core, 8G | Master | kube-apiserver, kube-controller-manager, \
kube-scheduler, etcd, haproxy, keepalived | Centos8 |
| 33.193.255.125 | worker-01 | 2 core, 8G | Worker | HAproxy, Keepalived | Centos8 |
| 33.193.255.126 | worker-02 | 2 core, 8G | Worker | HAproxy, Keepalived | Centos8 |
| 33.193.255.127 | worker-03 | 2 core, 8G | Worker | HAproxy, Keepalived | Centos8 |


### Step 1. Prepare base VM

1. Set up a base Centos8 VM for clone which can access Internet.
2. 
