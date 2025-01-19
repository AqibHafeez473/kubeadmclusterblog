# Setting Up a Kubernetes Cluster Using Kubeadm (Version 1.29)

## Pre-requisites
- Ensure all nodes (both master and worker) have a compatible OS.
- Have root or sudo access on each node.

## Steps for Both Master and Worker Nodes

### 1. Become Root User
```bash
sudo su
```

### 2. Update System Packages
```bash
sudo apt-get update
```

### 3. Install Docker
```bash
sudo apt install docker.io -y
sudo chmod 777 /var/run/docker.sock
```

### 4. Create and Run Setup Script
Create a script named `1.sh` with the following content and execute it:
```bash
#!/bin/bash
# Disable swap
sudo swapoff -a

# Load necessary kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters
sudo sysctl --system

# Install CRIO runtime
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service

# Install Kubernetes components
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-get install -y jq

sudo systemctl enable --now kubelet
sudo systemctl start kubelet
```

## Steps for Master Node

### 1. Initialize the Master Node
Create and run a script named `2.sh` with the following commands:
```bash
#!/bin/bash
sudo kubeadm config images pull
sudo kubeadm init

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u):$(id -g)" "$HOME"/.kube/config

# Install Calico network plugin
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Print the join command for worker nodes
kubeadm token create --print-join-command
```

### 2. Save the Join Command
Note the join command output, which will be used for worker nodes.

## Steps for Worker Nodes

### 1. Join the Cluster
Execute the join command obtained from the master node and append `--v=5` at the end for verbose logging:
```bash
<join-command> --v=5

