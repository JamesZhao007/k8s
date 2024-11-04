# Setup K8s cluster on RHEL 8

# **Install containerd, kubeadm, kubelet, kubectl**

## Create a [install.sh](http://install.sh) file with the following cotent

```bash
#!/bin/bash

# swap already support https://kubernetes.io/blog/2023/08/24/swap-linux-beta/ if you want to enalbe swap, please check the link
echo "[TASK 1] Disable and turn off SWAP"
sed -i '/swap/d' /etc/fstab
swapoff -a

echo "[TASK 2] Install some tools"
yum install -q -y vim jq iputils net-tools >/dev/null 2>&1

echo "[TASK 3] Stop and Disable firewall"
systemctl disable --now firewalld >/dev/null 2>&1

echo "[TASK 4] Enable and Load Kernel modules"
cat >>/etc/modules-load.d/containerd.conf<<EOF
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

echo "[TASK 5] Add Kernel settings"
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system >/dev/null 2>&1

echo "[TASK 6] Install containerd runtime"
yum install -q -y yum-utils >/dev/null 2>&1
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo >/dev/null 2>&1
yum install -q -y containerd.io >/dev/null 2>&1
containerd config default >/etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd >/dev/null 2>&1

echo "[TASK 7] Add yum repo for Kubernetes"
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl
EOF

yum install -q -y kubelet kubeadm kubectl --disableexcludes=kubernetes >/dev/null 2>&1
yum update -q -y >/dev/null 2>&1

```

## Run the shell on EVERY node:

```bash
sudo sh install.sh
```

## Run the following command on Master node:

```bash
yum list --showduplicates kubeadm

Installed Packages
kubeadm.x86_64  1.30.4-150500.1.1       @kubernetes

yum list --showduplicates kubelet

yum list --showduplicates kubectl
```

## Run the following command on ALL nodes to install kubeadm/kubelet/kubectl:

```bash
VERSION=1.30.4
sudo yum install -y kubeadm-$VERSION kubelet-$VERSION 
VERSION=488.0.0-1
sudo yum install -y kubectl-$VERSION

```

## Check if kubeadm，kubelet，kubect are installed properly:

```bash
kubeadm version
kubelet --version
kubectl version --client=true
```

# Initiate Master node:

All the followings are executed on the MASTER node

## Obtain the necessary images required by the cluster:

```bash
sudo kubeadm config images pull
```

If succeed, the following can be output:

```bash
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.30.4
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.30.4
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.30.4
[config/images] Pulled registry.k8s.io/kube-proxy:v1.30.4
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.11.1
[config/images] Pulled registry.k8s.io/pause:3.9
[config/images] Pulled registry.k8s.io/etcd:3.5.12-0
```

## Initiate kubeadmin:

- --apiserver-advertise-address 
The `--apiserver-advertise-address` is the IP address that the Kubernetes API server will advertise to members of the cluster. This address is typically the IP address of the master node that other nodes will use to connect to the API server.

```bash
ip addr show
```

- --pod-network-cidr
Since you'll be using Flannel as the network plugin for your Kubernetes cluster, the `--pod-network-cidr` value is typically set to `10.244.0.0/16`. This is the default CIDR block that Flannel uses to assign IP addresses to pods within the cluster.

```bash
sudo kubeadm init --apiserver-advertise-address=10.241.32.80  --pod-network-cidr=10.244.0.0/16
```

### Save the output

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.241.32.80:6443 --token 7rl96t.hdd0cjjrj6w4om9b \
	--discovery-token-ca-cert-hash sha256:9874232451ffc0c4931c546186719da823e9fe2061530bc4e26152070d855af0
```

# Prepare .kube, deploy pod network, add worker nodes

### config .kube

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### check output

```bash
kubectl get nodes
kubectl get pods -A

#Sample output 
parallels@k8s-master:~$ kubectl get node -o wide
NAME         STATUS     ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
k8s-master   NotReady   control-plane   63s   v1.30.3   10.211.55.11   <none>        Ubuntu 22.04.2 LTS   5.15.0-117-generic   containerd://1.7.19
parallels@k8s-master:~$ kubectl get pod -A
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-7db6d8ff4d-c5xr9             0/1     Pending   0          3m11s
kube-system   coredns-7db6d8ff4d-g256n             0/1     Pending   0          3m11s
kube-system   etcd-k8s-master                      1/1     Running   0          3m26s
kube-system   kube-apiserver-k8s-master            1/1     Running   0          3m26s
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          3m26s
kube-system   kube-proxy-g2gqs                     1/1     Running   0          3m12s
kube-system   kube-scheduler-k8s-master            1/1     Running   0          3m26s
```

### enable auto completion:

```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### deploy pod network:

```bash
# download kube-flannel.yml
curl -LO https://github.com/flannel-io/flannel/releases/download/v0.25.5/kube-flannel.yml
```

- check `Network` configuration make sure it is same as –pod-network-cidr 10.244.0.0/16

```bash
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "vxlan"
    }
  }
```

- check args of kube-flannel, make sure the ifac=<network interface name of –apiserver-advertise-address configured earlier> (example enp0s8)

```bash
129       containers:
130       - args:
131         - --ip-masq
132         - --kube-subnet-mgr
133         - --iface=eth0
```

example:

```bash
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP group default qlen 1000
    link/ether 42:01:0a:f1:20:50 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    altname ens4
    inet 10.241.32.80/32 scope global dynamic noprefixroute eth0
       valid_lft 3134sec preferred_lft 3134sec
    inet6 fe80::4001:aff:fef1:2050/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

Breakdown of enp0s4:
en - Indicates an Ethernet interface.
p0 - Refers to the PCI bus number (p0 means PCI bus 0).
s4 - Refers to the slot number (s4 means slot 4 on that bus).

Summary:
eth0: Traditional naming, less predictable, used in older Linux distributions.
enp0s4: Predictable network interface naming, more consistent, used in modern Linux distributions, and based on physical hardware location.
```

- save as flannel.yaml then apply the change

```bash
kubectl apply -f kube-flannel.yml

# output 
vagrant@k8s-master:~$ kubectl apply -f kube-flannel.yml
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

- Check if all pods are running:

```bash
kubectl get pods -A

NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-6d4b75cb6d-m5vms             1/1     Running   0          3h19m
kube-system   coredns-6d4b75cb6d-mmdrx             1/1     Running   0          3h19m
kube-system   etcd-k8s-master                      1/1     Running   0          3h19m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          3h19m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          3h19m
kube-system   kube-flannel-ds-blhqr                1/1     Running   0          3h18m
kube-system   kube-proxy-jh4w5                     1/1     Running   0          3h17m
kube-system   kube-scheduler-k8s-master            1/1     Running   0          3h19m
```

- Check if nodes are ready:

```bash
NAME              STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                                KERNEL-VERSION                 CONTAINER-RUNTIME
guedspek8smst01   Ready    control-plane   40m   v1.30.4   10.241.32.80   <none>        Red Hat Enterprise Linux 8.10 (Ootpa)   4.18.0-553.8.1.el8_10.x86_64   containerd://1.6.32
```

## Add worker nodes

- Run the following command on the WORKER NODES only

```bash
sudo kubeadm join 10.241.32.80:6443 --token 7rl96t.hdd0cjjrj6w4om9b \
--discovery-token-ca-cert-hash sha256:9874232451ffc0c4931c546186719da823e9fe2061530bc4e26152070d855af0
```

- How to retrieve  token and `discovery-token-ca-cert-hash`
    
    ```bash
    # Retrive token
    $ kubeadm token list
    TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
    0pdoeh.wrqchegv3xm3k1ow   23h         2022-07-19T20:13:00Z   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
    
    # Retrieve discovery-token-ca-cert-hash
    openssl x509 -in /etc/kubernetes/pki/ca.crt -pubkey -noout |
    openssl pkey -pubin -outform DER |
    openssl dgst -sha256
    ```
    

### Confirm the worker nodes on Master node (execute on the Master node):

```bash
# nodes
vagrant@k8s-master:~$ kubectl get nodes -o wide
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
k8s-master    Ready    control-plane   9h    v1.30.3   10.211.55.11   <none>        Ubuntu 22.04.2 LTS   5.15.0-117-generic   containerd://1.7.19
k8s-worker1   Ready    <none>          53s   v1.30.3   10.211.55.12   <none>        Ubuntu 22.04.2 LTS   5.15.0-117-generic   containerd://1.7.19
k8s-worker2   Ready    <none>          17s   v1.30.3   10.211.55.13   <none>        Ubuntu 22.04.2 LTS   5.15.0-117-generic   containerd://1.7.19

# pods
vagrant@k8s-master:~$ kubectl get pods -A
NAMESPACE      NAME                                 READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-bp4zf                1/1     Running   0          40s
kube-flannel   kube-flannel-ds-m2qwp                1/1     Running   0          75s
kube-flannel   kube-flannel-ds-qhhnl                1/1     Running   0          8h
kube-system    coredns-7db6d8ff4d-c5xr9             1/1     Running   0          9h
kube-system    coredns-7db6d8ff4d-g256n             1/1     Running   0          9h
kube-system    etcd-k8s-master                      1/1     Running   0          9h
kube-system    kube-apiserver-k8s-master            1/1     Running   0          9h
kube-system    kube-controller-manager-k8s-master   1/1     Running   0          9h
kube-system    kube-proxy-g2gqs                     1/1     Running   0          9h
kube-system    kube-proxy-lvx2n                     1/1     Running   0          40s
kube-system    kube-proxy-qsxwd                     1/1     Running   0          75s
kube-system    kube-scheduler-k8s-master            1/1     Running   0          9h
```

## 

```bash

```

```bash

```

```bash

```

```bash

```

## After the cluster is created:

- On the workers nodes:

```bash
mkdir -p $HOME/.kube
```

- On the master node:

```bash
sudo scp /etc/kubernetes/kubelet.conf zhepzhao@guedspek8swk101.dev.tibco.com:/home/zhepzhao/.kube/config
sudo scp /etc/kubernetes/kubelet.conf zhepzhao@guedspek8swk201.dev.tibco.com:/home/zhepzhao/.kube/config
```

- On the workers nodes:

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
cat ~/.kube/config
sudo chmod 644 /var/lib/kubelet/pki/kubelet-client-2024-08-15-23-09-18.pem
kubectl get nodes
sudo systemctl restart kubelet
```

## Keep cluster available after restarting VMs

### Ensure Control Plane Components Restart Automatically

```bash
sudo systemctl enable kube-apiserver
sudo systemctl enable kube-controller-manager
sudo systemctl enable kube-scheduler
sudo systemctl enable kubelet

sudo systemctl status kube-apiserver
sudo systemctl status kube-controller-manager
sudo systemctl status kube-scheduler
sudo systemctl status kubelet

kubectl get pods -n kube-system

```

### Ensure Worker Nodes Restart Automatically
