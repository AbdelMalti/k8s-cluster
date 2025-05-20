# How to install k8s on ubuntu

## If you need VMs localy
Follow the README in this github repo [local-vms](https://github.com/AbdelMalti/local-vms)

## Ansible part

`ansible-playbook -i hosts site.yml`    
`ansible-playbook site.yml -i hosts --user=vagrant --private-key=../local-vms/id_rsa_vagrant`

## Sources

- [computingforgeeks](https://computingforgeeks.com/deploy-kubernetes-cluster-on-ubuntu-with-kubeadm/)
- [letscloud](https://www.letscloud.io/community/how-to-install-kubernetesk8s-and-docker-on-ubuntu-2004)
- [CNI](https://www.suse.com/c/rancher_blog/comparing-kubernetes-cni-providers-flannel-calico-canal-and-weave/)
- [Calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)
- [Flannel](https://github.com/flannel-io/flannel#flannel)
- [kubernetes setup using ansible and vagrant](https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/)
- [containerd playbook](https://github.com/geerlingguy/ansible-role-containerd/blob/master/tasks/main.yml)

## Steps

1. Update the OS (All nodes)
2. Install kubelet, kubeadm, kubectl (All nodes)
    1. Install kubeadm (On master)
3. Disable Swap (All nodes)
4. Install container runtime (All nodes)
5. Init master node (On master)
6. Install CNI (On master)
    1. Calico
    2. Flannel
    3. Weave
7. Init worker nodes (On workers)

## Commands per steps

### Step 1

```
sudo apt update -y
sudo apt -y upgrade && sudo systemctl reboot
```

### Step 2

```
sudo apt update
sudo apt -y install vim git curl wget kubelet kubectl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

sudo apt-mark hold kubelet kubectl

kubectl version --client
```

#### Step 2.1
```
sudo apt -y install kubeadm
sudo apt-mark hold kubeadm
kubeadm version
```

### Step 3

```
sudo swapoff -a
```

### Step 4 (containerd)

```
# Configure persistent loading of modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Load at runtime
sudo modprobe overlay
sudo modprobe br_netfilter

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload configs
sudo sysctl --system

# Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd and start service
sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

# restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status  containerd
```

### Step 5

```
sudo hostnamectl set-hostname master-node

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket /run/containerd/containerd.sock \
  --upload-certs 

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

### Step 6

#### 1 - Calico

```
# For 50 nodes or less
curl https://docs.projectcalico.org/manifests/calico.yaml -O calico.yaml
kubectl apply -f calico.yaml

# For more than 50 nodes
curl https://docs.projectcalico.org/manifests/calico-typha.yaml -o calico.yaml
kubectl apply -f calico.yaml
```

#### 2 - Flannel

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

#### 3 - Weave

```
# TODO
```

### Step 7

```
sudo hostnamectl set-hostname worker1 # Or worker2, worker3, worker4, etc ...
kubeadm join --discovery-token abcdef.1234567890abcdef --discovery token-ca-cert-hash sha256:1234..cdef 1.2.3.4:6443

```
