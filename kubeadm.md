# install kubernetes on ubuntu 20.04

## install ubuntu
```shell
sudo apt install openssh-server
sudo apt install emacs
sudo apt install git
sudo apt install lrzsz
sudo apt install unzip
```

## install ipvs
```shell
# https://installati.one/ubuntu/20.04/ipvsadm/
# https://devopstales.github.io/kubernetes/k8s-ipvs/
sudo apt-get update
sudo apt-get -y install ipvsadm
```

## install Docker
```shell
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io

sudo groupadd docker

sudo usermod -aG docker ${USER}

sudo curl -L "https://github.com/docker/compose/releases/download/v2.3.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```

## install kubeadm kubelet kubectl
```shell
# https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.23.7-00 kubeadm=1.23.7-00 kubectl=1.23.7-00
sudo apt-mark hold kubelet kubeadm kubectl

sudo reboot
```

## init
```shell

# clean previous initialized k8s (if one)
sudo kubeadm reset cleanup-node
# clean network
sudo ifconfig cni0 down
sudo ip link delete cni0
sudo ifconfig flannel.1 down
sudo ip link delete flannel.1
sudo rm -rf /var/lib/cni/
sudo rm -f /etc/cni/net.d/*
# clean iptables
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
# if with IPVS
sudo ipvsadm --clear
# remove configs
sudo rm -rf $HOME/.kube

# kubeadm init
# config cgroup
# https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/
# change Docker engine cgroup /etc/docker/daemon.json
# {
#  "exec-opts": ["native.cgroupdriver=systemd"]
# }
# sudo echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' >> /etc/docker/daemon.json
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl restart docker

sudo kubeadm init --config kubeadm-config.yaml
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# deploy a pod network to the cluster
# Run "kubectl apply -f [pod-network].yaml" with one of the options listed at:
# https://kubernetes.io/docs/concepts/cluster-administration/addons/
# https://github.com/flannel-io/flannel
# change CIDR to 172.16.0.0/12
kubectl apply -f kube-flannel.yml

# schedule Pods on the control-plane node
kubectl taint nodes --all node-role.kubernetes.io/master-

# watch 
watch kubectl get pods --all-namespaces
kubectl cluster-info
kubectl get nodes -o wide

# ipvs
sudo ipvsadm -Ln

# UI
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
kubectl proxy
# http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

# install metallb for bare metal load balance
# https://metallb.universe.tf/ 
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml

kubectl apply -f metallb.yaml
```