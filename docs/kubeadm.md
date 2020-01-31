# Setting up Kubernetes with kubeadm

What I used to build my own Kubernetes Raspberry Pi Cluster.


Creating a Kubernetes Cluster
hostnamectl set-hostname master.kube.local
hostnamectl set-hostname node-01.kube.local
hostnamectl set-hostname node-02.kube.local
hostnamectl set-hostname node-03.kube.local

Edit /etc/hosts

10.0.0.10 master.kube.local
10.0.0.11 node-01.kube.local
10.0.0.12 node-02.kube.local
10.0.0.13 node-03.kube.local

memory control group subsystem is disabled by default at Raspberry Pi build of Ubuntu 18.04.
To fix that — we need to append the following line

```
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

to Linux kernel cmd line on each device. In Ubuntu 18.04, /boot/firmware/cmdline.txt file sets the required parameter for Raspberry Pi.

sudo reboot

sudo apt-get upgrade

Install Docker
# Install Docker CE
## Set up the repository:
### Install packages to allow apt to use a repository over HTTPS
apt-get update && apt-get install apt-transport-https ca-certificates curl software-properties-common

### Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

### Add Docker apt repository.
add-apt-repository \
  "deb [arch=arm64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

## Install Docker CE.
apt-get update && apt-get install docker-ce=18.06.2~ce~3-0~ubuntu

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker

Install kubeadm

apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl docker-ce

Setup Kubernetes Cluster

sudo kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm join 10.0.0.10:6443 --token ojtwn9.mua2515qenc12wy0 \
    --discovery-token-ca-cert-hash sha256:9203353a5aa26017513f63ee578ec00d0b5b673f6a538e1625d1b53918c62af6

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml

Add a Route
sudo route -n add -net  10.0.0.0/24 192.168.1.1
