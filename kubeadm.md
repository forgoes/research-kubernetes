# install kubernetes on ubuntu 20.04

## install
```shell

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

# config cgroup
# https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/

# kubeadm init
sudo kubeadm init --config kubeadm-config.yaml
sudo kubeadm init --config kubeadm-config.yaml --pod-network-cidr=172.16.0.0/12
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

```