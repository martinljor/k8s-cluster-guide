# Kubernetes Cluster
Guide with step by step to deploy a new Kubernetes Cluster and test it.
Its important to know that this guide is based on official references: 
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm

## Architecture

### Steps

#### Hosts file
Because I didnt setup DNS services I configured /etc/hosts identically on each node.

```bash
192.168.0.231   kub01
192.168.0.232   kub02
192.168.0.233   kub03
```

#### SSH
In each node its recommended to have passwordless communication between the nodes.
Run this commands to grant ssh access between the nodes:

```bash
ssh-keygen
ssh-copy-id $node
```

#### SWAP OFF
```bash
sudo swapoff -a
```

#### Ports Available
nc 127.0.0.1 6443

 wget https://download.docker.com/linux/centos/docker-ce.repo
 cp docker-ce.repo /etc/yum.repos.d/
yum repolist

yum install containerd -y
systemctl enable containerd
systemctl start containerd


sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
reboot

- Add repo for k8s
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable kubelet

firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp

firewall-cmd --reload
firewall-cmd --list-ports

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl -p
```


- Change /etc/containerd/config.toml from disabled_plugin to enabled_plugin and save the file:
```bash
sed -i 's/disabled_plugins/enabled_plugins/g' /etc/containerd/config.toml
```

- Restart service containerd: 
```bash
systemctl restart containerd
```



https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/





