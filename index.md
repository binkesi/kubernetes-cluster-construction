## Build High available Kubernetes cluster

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Overview

For me the best practise to learn kubernetes is to build a HA kubernetes cluster step by step. And try any experiment on it.

### Prerequisite

We need a virtualization platform to set up seven VMs. Three master nodes, three worker nodes and one Load Balance node with
HAproxy and Keepalived installed.
