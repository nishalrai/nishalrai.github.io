---
title: Setup your own Kubernetes Cluster
description: >-
  Get started with Chirpy basics in this comprehensive overview.
  You will learn how to install, configure, and use your first Chirpy-based website, as well as deploy it to a web server.
author: nishalrai
date: 2024-02-27 20:55:00 +0800
categories: [Cloud Native, Microservice]
tags: [getting started, kubernetes]
image: /assets/img/images/k8s/k8sOverview.png
---


The best way to deepen your understanding is to build, break, and fix it—at least, this is how my learning works. After delving into videos, sifting through documentation, and scouring online posts, I’ve managed to glean a high-level understanding of container orchestration within Kubernetes clusters. Yet, the burning question persists: How does Kubernetes wield its management magic?

Well, you can guess—time for action. Most of the commands and descriptions have been sourced from the official Kubernetes documentation.

Before we jump straight into the configuration sections, please allow yourself to read a few lines of documentation about the essential key components of the K8s cluster in the official documentation of Kubernetes. This will be very helpful in the latter section of the configuration.

## Prior to Installation

- A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
- 2 GB or more of RAM per machine (any less will leave little room for your apps). 
- 2 CPUs or more.
- Full network connectivity between all machines in the cluster (public or private network is fine).
- Unique hostname, MAC address, and product_uuid for every node. See here for more details.
- Certain ports are open on your machines. See here for more details.
- Swap configuration. The default behavior of a kubelet was to fail to start if swap memory was detected on a node. Swap has been  supported since v1.22. And since v1.28, Swap is supported for cgroup v2 only; the NodeSwap feature gate of the kubelet is beta but disabled by default.


For this setup, three virtual instances with Rocky Linux version 8.9 as nodes will be used to create a K8s cluster.

![kubernetes](/assets/img/images/k8s/a0 os version.png){: width="519" height="408" }
_Full screen width and center alignment_

The configuration of the overall K8s cluster is divided into three major segments: configuring the master node, configuring two worker nodes, and possible issues. Since almost two-thirds of the configuration on the master nodes needs to be performed on the **worker node** too, with minimal changes. Thus, all the necessary configurations are grouped in a single section at the second section.
The below section lists the common configuration required on all three nodes and illustrates the master configuration along the steps.

- **Disable SElinux, and swap memory on all nodes**
- **Set hostname, define instance in the hostfile and reboot**
- **Configure firewall rules**
- **Load kernel modules and configure Kernel Parameters**
- **Installing Container Runtime – containerd**
- **Configuring Kubernetes Repository**
- **Installing Kubernetes Tools**
- **Enabling kubelet service**
- **Creating user and Kubernetes Configuration**
- **Installing Calico**
- **Initiating kubedm in master nodes**
- **Adding worker nodes to master (control plane)**
- **Labeling Worker Nodes**

<br>

The next section documents the commands that need to be executed, with a short description of their usage and a screenshot of the necessary steps for ease-of understanding.
<br>


## Disable SELinux and swap memory on all nodes
The default behavior of a kubelet was to fail to start if swap memory was detected on a node. Swap has been supported since v1.22. And since v1.28, swap is supported for cgroup v2 only.

```
$ vi /etc/selinux/config
$ vi /etc/fstab

$ hostnamectl set-hostname <host-name>
$ reboot
```

Thus, the current installation version is 1.28 and you can verify your cgroup by executing a command in the CLI.

```
$ stat -fc %T /sys/fs/cgroup/
```

- For cgroup v2, the output is cgroup2fs.
- For cgroup v1, the output is tmpfs.
  
[View the version of the cgroups](https://kubernetes.io/docs/concepts/architecture/cgroups/)

![](/assets/img/images/k8s/k8sMaster.png){: width="600" height="400"}

![](/assets/img/images/k8s/k8sSwap.png){: width="600" height="400"}


**Disabling SELinux on the system**

![](/assets/img/images/k8s/k8sSelinux.png){: width="600" height="400"}

![](/assets/img/images/k8s/k8sSelinux1.png){: width="600" height="400"}

You need to define the machines in the host file on all the nodes.

```linux
vi /etc/hosts

192.168.190.6 master
192.168.190.4 worker01
192.168.190.9 worker02
```
<br>

## Configuring firewall-rules
Configuring firewall rules is essential for securing your Kubernetes cluster. On the master node, you’ll use firewall-cmd to define rules that permit traffic on specific ports, ensuring smooth communication between cluster components. Similarly, on worker nodes, similar rules need to be established to facilitate proper functioning.

![](/assets/img/images/k8s/k8sFirewall.png){: width="600" height="400"}

```linux
$ firewall-cmd --permanent --add-port {6443,2379,2380,10250,10251,10252,10257,10259,179}/tcp 
$ firewall-cmd --permanent --add-port=4789/udp 
$ firewall-cmd --reload
```

![](/assets/img/images/k8s/k8sFirewall1.png){: width="600" height="400"}

![](/assets/img/images/k8s/k8sFirewall2.png){: width="600" height="400"}

Similarly, on the worker nodes, the following ports are added:.

```
$ firewall-cmd --permanent --add-port={179,10250,30000-32767}/tcp 
$ firewall-cmd --permanent --add-port=4789/udp 
$ firewall-cmd --reload
```
<br>

## Load Kernel modules and configure kernel parameters
or Kubernetes plugin developers and users, it’s important to configure support for kube-proxy. Depending on the plugin used, it may require specific settings to ensure compatibility with the iptables proxy. For instance, if the plugin connects containers to a Linux bridge, it must enable the net/bridge/bridge-nf-call-iptables sysctl setting to 1. This ensures the proper functioning of the iptables proxy. If the plugin uses alternatives like Open vSwitch, it needs to ensure appropriate routing for container traffic to work with the proxy. By default, if no kubelet network plugin is specified, the noop plugin is used. This sets net/bridge/bridge-nf-call-iptables=1, ensuring smooth operation, especially for simple configurations like Docker with a bridge.

More detailed information can be found at the following link:.

In the case of configuring kernel modules and parameters in your system, you can follow either the Kubernetes documentation process or the other.

<br>

### Option I: K8s documentation
Forwarding IPv4 and letting iptables see bridged traffic based on the network plugin requirements.

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
<br>

### Option II: vi editor
You need to first add two kernel modules by creating a file on `/etc/modules-load.d/k8s.f` then insert few parameters into a new file `“/etc/sysctl.d/k8s.conf”`.

<br>

```linux
vi /etc/modules-load.d/k8s.conf

overlay 
br_netfilter


vi /etc/sysctl.d/k8s.conf 

net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1 
net.bridge.bridge-nf-call-ip6tables = 1

sysctl --system
```

![](/assets/img/images/k8s/k8sEditor.png){: width="600" height="400"}

<br>

## Installing Container Runtime & yum utility
One needs to install a container runtime into each node in the cluster to be able to run the Pods and some of the common container runtimes with Kubernetes are:

- containerd
- CRI-O
- Docker Engine
- Mirantis Container Runtime

> Note:
> Kubernetes releases before v1.24 included a direct integration with Docker Engine, using a component named dockershim. That special direct integration is no longer part of Kubernetes (this removal was announced as part of the v1.20 release).

<br>

More information can be found at the following link.
```
$ yum install -y yum-utils
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ dnf install containerd.io -y
```

![](/assets/img/images/k8s/k8sYum.png){: width="600" height="400"}

![](/assets/img/images/k8s/k8sYumrepo.png){: width="600" height="400"}

<br>

### Modifying the system_cgroup
On Linux, control groups (cgroups) manage allocated resources for processes. The kubelet and container runtime interact with cgroups to enforce resource management for pods, setting CPU/memory requests and limits. They must use the same cgroup driver and configuration.
Two cgroup drivers exist:

- cgroupfs
- systemd.

The cgroupfs driver is the default in the kubelet, interfacing directly with the cgroup filesystem. However, it’s not recommended with systemd as the init system due to potential conflicts. Instead, when using systemd, the systemd cgroup driver is preferred. Systemd acts as a cgroup manager, allocating a cgroup per systemd unit. This integration avoids the instability that can occur when using cgroupfs alongside systemd.

Additional information can be found at the following link – systemd cgroup driver.


```
mv /etc/containerd/config.toml /etc/containerd/config.toml.backup
containerd config default ＞ /etc/containerd/config.toml 2> /dev/null

vim /etc/containerd/config.toml
SystemdCgroup = true
```

![](/assets/img/images/k8s/k8sCgroup.png){: width="600" height="400"}

```
sudo systemctl restart containerd
sudo systemctl enable containerd
```

![](/assets/img/images/k8s/k8sContainerd.png){: width="600" height="400"}


## Configuring Kubernetes Repo
This step involves configuring the Kubernetes repository on all nodes. By creating a file named “kubernetes.repo” in the “/etc/yum.repos.d/” directory and adding the specified configuration, you enable access to the Kubernetes packages. This configuration includes details such as the repository name, base URL for package retrieval, whether it’s enabled, GPG key verification, and exclusions for specific Kubernetes tools. This setup ensures that necessary Kubernetes packages can be easily installed and managed on all nodes in the cluster.

> Note:
> You can change the repo links to the latest version of K8s v 1.29

<br>

```
vi  /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

![](/assets/img/images/k8s/k8sRepo.png){: width="600" height="400"}

![](/assets/img/images/k8s/k8sRepo1.png){: width="600" height="400"}

<br>

## Installing Kubernetes Tools
This command installs essential Kubernetes tools (kubelet, kubeadm, and kubectl packages) on the nodes simultaneously using the yum install command. Here, the –disableexcludes=kubernetes flag ensures that Kubernetes-related packages are not excluded during installation. This step is crucial for setting up and managing Kubernetes clusters across all nodes efficiently.

`yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes`

![](/assets/img/images/k8s/k8sComponent.png){: width="600" height="400"}

<br>

### Enabling kubelet service
This command enables the kubelet service on the node. By using the systemctl enable command, the kubelet service is configured to start automatically during system boot, ensuring that the Kubernetes node agent is always running and ready to manage containers and pods as part of the Kubernetes cluster

```
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```
![](/assets/img/images/k8s/k8sKubelet.png){: width="600" height="400"}

<br>

### Initiating kubedm in master node
The command kubeadm init –control-plane-endpoint=master initializes the Kubernetes control plane on a master node. By specifying the –control-plane-endpoint flag with the value master, it designates the control plane endpoint. Typically used during the setup of a Kubernetes cluster to establish the primary control plane components on the master node, it facilitates cluster management and coordination.
```
kubeadm init --control-plane-endpoint=master
kubeadm init --control-plane-endpoint=<master-hostname>
```
![](/assets/img/images/k8s/k8sKubeinit.png){: width="600" height="400"}

![](/assets/img/images/k8s/k8sKubeinit1.png){: width="600" height="400"}

<br>

## Option to run the cluster
To begin utilizing your cluster, you have two options to configure, depending on your user privileges.

As a regular user, execute the following commands:
These commands create a directory for Kubernetes configuration, copy the admin configuration file to it, and adjust ownership to the current user.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Root User Option
Alternatively, if you’re logged in as the root user, you can set the configuration directly:
This command sets the KUBECONFIG environment variable to point to the admin configuration file.
`export KUBECONFIG=/etc/kubernetes/admin.conf`

> Note:
> Regular user has been used in this configuration process.

## Configuring networking in master node
The below command installs Calico, a networking and network policy provider for Kubernetes clusters. By applying the YAML configuration file hosted at the specified URL, Calico is deployed onto the cluster, enabling network connectivity and policy enforcement functionalities. Calico needs to be installed on all the nodes—the master and worker nodes.

`kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/calico.yaml`

![](/assets/img/images/k8s/k8sNetwork.png){: width="600" height="400"}

**Verifying the master node running**
`kubectl get nodes`

![](/assets/img/images/k8s/k8sNode.png){: width="600" height="400"}

<br>

### Configuration on worker nodes
After the completion of configuring the master node, we need to repeat the same process of configuration until we enable the Kubelet – section of the post, or you can simply follow the overall steps mentioned below:

```
Disable SELinux and swap memory on all nodes
$ vi /etc/selinux/config
[Disable SELinux]

$ vi /etc/fstab
[Comment swap section]

$ hostnamectl set-hostname <host-name>

$ vi /etc/hosts

<ip-address> <hostname>
192.168.190.6 master
192.168.190.4 worker01
192.168.190.9 worker02

$ reboot


Configuring firewall-rules
$ firewall-cmd --permanent --add-port={179,10250,30000-32767}/tcp 
$ firewall-cmd --permanent --add-port=4789/udp 
$ firewall-cmd --reload


Load kernel modules and configure kernel parameters
$ vi /etc/modules-load.d/k8s.conf

overlay 
br_netfilter


$ vi /etc/sysctl.d/k8s.conf 

net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1 
net.bridge.bridge-nf-call-ip6tables = 1

$ sysctl --system


Installing Container Runtime & yum utility
$ yum install -y yum-utils

$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ dnf install containerd.io -y


Modifying the system_cgroup variable value
$ mv /etc/containerd/config.toml /etc/containerd/config.toml.backup
$ containerd config default ＞ /etc/containerd/config.toml 2> /dev/null

$ vim /etc/containerd/config.toml
SystemdCgroup = true


$ sudo systemctl restart containerd
$ sudo systemctl enable containerd


Configure kubernetes repo
$ vi  /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni


Installing kubernetes tools
$ yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes


Enabling kubelet service
$ systemctl enable kubelet
$ systemctl start kubelet
$ systemctl status kubelet

Configuring networking
$ kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/calico.yaml


Copy the kubeadm join command:
$ kubeadm join master:6443 --token rbs3i8.5kelv7xvewgb9h9o \
        --discovery-token-ca-cert-hash sha256:8f80e86f1f981684f75b9154d232e2d9f9b5ad43a23ec5ba86453ca6cf70029b
```

## Possible issue prompt in the configuration
It would be quite boring if no error pops up throughout the configuration of the Kubernetes cluster, and some of the possible issues are mentioned below:

### Unable to initiate kubeadm in the nodes
Sometimes the default configuration file of containerd residing on /etc/containerd/config.toml causes the issues to initiate kubeadm service in the nodes.

![](/assets/img/images/k8s/k8sIssue1.png){: width="600" height="400"}

```
Error prompt while trying to initiate kubeadm

[root@master ~]# kubeadm init --control-plane-endpoint=master
I0225 21:33:16.227738   23648 version.go:256] remote version is much newer: v1.29.2; falling back to: stable-1.28
[init] Using Kubernetes version: v1.28.7
[preflight] Running pre-flight checks
        [WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
        [WARNING FileExisting-tc]: tc not found in system path
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR CRI]: container runtime is not running: output: time="2024-02-25T21:33:17+05:45" level=fatal msg="validate service connection: validate CRI v1 runtime                  API for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
, error: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

If you simply remove the default config file and restart the containerd service, the issues seem to be resolved.

```
$ rm /etc/containerd/config.toml
$ systemctl restart containerd
$ kubeadm init --control-plane-endpoint=master
```
![](/assets/img/images/k8s/k8sIssue.png){: width="600" height="400"}

### Without the configuration of network configuration in the nodes
If you don’t configure networking in all the master node, and jump straight into adding the worker nodes in the cluster. You will be receiving the following error prompts:

![](/assets/img/images/k8s/k8sToken.png){: width="600" height="400"}

```
Error prompts in the terminal

[root@worker02 ~]# kubeadm join master:6443 --token rbs3i8.5kelv7xvewgb9h9o \
>         --discovery-token-ca-cert-hash sha256:8f80e86f1f981684f75b9154d232e2d9f9
[preflight] Running pre-flight checks
        [WARNING FileExisting-tc]: tc not found in system path
error execution phase preflight: couldn't validate the identity of the API Server: invali                                                                                              d discovery token CA certificate hash: invalid hash "sha256:8f80e86f1f981684f75b9154d232e                                                                                              2d9f9", expected a 32 byte SHA-256 hash, found 17 bytes
To see the stack trace of this error execute with --v=5 or higher
```

You simply need to run in terminal.
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/calico.yaml

![](/assets/img/images/k8s/k8sToken1.webp){: width="600" height="400"}

### Trying kubectl command outside the configured user
This issue prompts when you try to execute the kubectl command outside the configured user for the cluster management in the master node.

![](/assets/img/images/k8s/k8sGetpods.webp){: width="600" height="400"}

```
[root@master ~]# kubectl get pods
E0225 22:23:26.880146   25223 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect:                  connection refused
E0225 22:23:26.880593   25223 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect:                  connection refused
E0225 22:23:26.882285   25223 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect:                  connection refused
E0225 22:23:26.883697   25223 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect:                  connection refused
E0225 22:23:26.884950   25223 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect:                  connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

You need to follow the configuration steps mentioned in the early section of configuration.

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
```