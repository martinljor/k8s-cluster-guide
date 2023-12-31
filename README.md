# Kubernetes Cluster
Guide with step by step to deploy a new Kubernetes Cluster and test it creating a pod.
Its important to know that this guide is based on official references: 
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm

Principal reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## Architecture
![ARCK8S](images/image-10.png)

### Hosts file
Because I didnt setup DNS services I configured /etc/hosts identically on each node.

```bash
192.168.0.231   kub01
192.168.0.232   kub02
192.168.0.233   kub03
```

### SSH
In each node its recommended to have passwordless communication between the nodes.
Run this commands to grant ssh access between the nodes:

```bash
ssh-keygen
ssh-copy-id $node
```

### SWAP OFF
Disable SWAP on each node.
```bash
sudo swapoff -a
sed -i -e 's/.*swap/#&/' /etc/fstab
mount -a
```

### Disable selinux
Disable SELINUX on each node.

```bash
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config && reboot
```
### YUM repos
Create the repo
```bash
 wget https://download.docker.com/linux/centos/docker-ce.repo
 cp docker-ce.repo /etc/yum.repos.d/
yum repolist
```
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

### Install components
Install the containerd service

```bash
yum install containerd -y
systemctl enable containerd
systemctl start containerd


yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable kubelet
```

### Firewall

Add rules to the firewall on each node
```bash

firewall-cmd --permanent --add-port=8001/tcp
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10248/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp

firewall-cmd --reload
firewall-cmd --list-ports
```

### Enable modules in all nodes

```bash
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

modprobe overlay
modprobe br_netfilter
```

Change /etc/containerd/config.toml from disabled_plugin to enabled_plugin and save the file:
```bash
sed -i 's/disabled_plugins/enabled_plugins/g' /etc/containerd/config.toml
```

Restart service containerd: 
```bash
systemctl restart containerd
```

### Create cluster
Init Cluster
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```
*Expected results are: Your Kubernetes control-plane has initialized successfully!

Env vars to access to the API server:
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Create pod network with flannel
Flannel is a simple and easy way to configure a layer 3 network fabric designed for Kubernetes:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```


Join nodes to the cluster
```bash
kubeadm join kub01:6443 --token $$TOKEN --discovery-token-ca-cert-hash $$HASH
```
*Change $$TOKEN and $$HASH to the correct onces. Its appear when you inited the cluster.

More information about joining nodes to the cluster: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes

Change label for worker nodes:
```bash
kubectl label node kub02 node-role.kubernetes.io/worker=worker
kubectl label node kub03 node-role.kubernetes.io/worker=worker
```

![ClusterReady](images/image-6.png)

Cluster Ready!

## Graphic interface 
The Dashboard UI is not deployed by default. To deploy it, run the following command:

```bash
mkdir /root/dashboard
cd /root/dashboard
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Modify "spec" -> "type" to "NodePort"
Modify "spec" -> "ports" -> "nodePort: 30001"

Like this...

![ModifyDashAccess](images/image-8.png)

Apply file configuration:
```bash
kubectl create -f  ~/dashboard/recommended.yaml
kubectl get pods -A  -o wide
```

Now create this 3 files:
dashboard-adminuser.yaml
dashboard-role.yaml
dashboard-token.yaml

```bash
cd /root/dashboard
cat >> dashboard-adminuser.yaml<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat >>dashboard-role.yaml.yaml<<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat >>dashboard-token.yaml<<EOF
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token

```
![Files-DashboardYAML](images/image-9.png)

Apply 3 files configuration:

```bash
kubectl create -f  /root/dashboard-adminuser.yaml
kubectl create -f  /root/dashboard-role.yaml
kubectl create -f  /root/dashboard-token.yaml
```

Now you can now access: https: // NodeIP: 30001
To obtain the token use this command:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Copy and paste it into the login screen.

#### First pod
Once you’re in the Kubernetes sandbox environment, make sure you’re connected to the Kubernetes cluster by executing kubectl get nodes in the command line to see the cluster's nodes in the terminal. If that worked, you’re ready to create and run a pod.

```bash
kubectl run nginx --image=nginx --restart=Never
```

Once you hit enter, the pod will be created. You should see pod/nginx created appear in the terminal.
You can now run the command kubectl get pods to see the status of your pod. To view the entire configuration of the pod, just run kubectl describe pod nginxin your terminal.

```bash
kubectl get pods
kubectl describe pod nginx
```

![nginxrunning](images/image-7.png)

Its running on worker kub02.


## "Thanks to reach here :)"








