# Kubernetes Cluster
Guide with step by step to deploy a new Kubernetes Cluster and test it.
Its important to know that this guide is based on official references: 
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm

## Architecture

### Steps

#### Hosts file
Because I didnt setup DNS services I configured /etc/hosts identically on each node.

´´´bash
192.168.0.231   kub01
192.168.0.232   kub02
192.168.0.233   kub03
´´´

#### SSH
In each node its recommended to have passwordless communication between the nodes.
Run this commands to grant ssh access between the nodes:

ssh-keygen
ssh-copy-id $node




sudo swapoff -a
nc 127.0.0.1 6443

