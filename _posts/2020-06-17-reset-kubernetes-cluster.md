---
layout: post
title: Reset Kubernetes Cluster using kubeadm
date: 2020-06-07 19:00:00 +0530
tags: [kubernetes, reset, cluster]
---

In this tutorial, we will learn about how to reset single node/control-plane Kubernetes Cluster. It will wipe out Kubernetes cluster data that was configured using `kubeadm init` as described in [Create single node Kubernetes cluster on Ubuntu using kubeadm on Google Cloud Platform (GCP)](2020-06-17-single-node-k8s-ubuntu-gcp-kubeadm.md). Same steps which are mentioned below could be also performed on any Kubernetes cluster created on Ubuntu whether they are hosted on Google Cloud Platform (GCP) or not.  

In case you have configured your single node/control-plane Kubernetes cluster using alternate methods, steps mentioned below may not be sufficient to completely wipe out Kubernetes configuration. There might be some additional steps involved to completely wipe out everything.

## Reset Kubernetes cluster

To reset a Kubernetes cluster, use `kubeadm reset` command like below:
```
sudo kubeadm reset
```
Sample output:
```
$ sudo kubeadm reset
[reset] Reading configuration from the cluster...
[reset] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
[reset] Removing info for node "k8s-master" from the ConfigMap "kubeadm-config" in the "kube-system" Namespace
[reset] failed to remove etcd member: etcdserver: re-configuration failed due to not enough started members.Please manually remove this etcd member using etcdctl
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
[reset] Deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.

rm -f $HOME/.kube/config
```

As mentioned in the output, there are additional commands that needs to be run to complete the clean-up.

Remove CNI configuration entries:
```
sudo rm -f /etc/cni/net.d/10-flannel.conflist
``` 
Reset iptables:
```
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```
Remove kubeconfig file:
```
sudo rm -f $HOME/.kube/config
```
