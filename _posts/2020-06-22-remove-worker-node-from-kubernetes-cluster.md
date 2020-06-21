---
layout: post
title: Remove worker node from Kubernetes cluster
date: 2020-06-21 15:00:00 +0530
tags: [kubernetes, remove, worker-node]
---

In this tutorial, we will learn about how to remove worker node from Kubernetes Cluster. To learn more about how to create a single node/control-plane Kubernetes cluster, please refer [Create single node Kubernetes cluster on Ubuntu using kubeadm on Google Cloud Platform (GCP)](2020-06-17-single-node-k8s-ubuntu-gcp-kubeadm.md) and to add worker node refer [Add worker node to Kubernetes Cluster](2020-06-17-add-worker-node.md).

We need to first drain the node, so that it does not have any running pods. Then remove the node from Kubernetes cluster. Lastly we will run `kubeadm reset`.

To drain the node, run below command with as an user with appropriate permissions to run `kubectl drain`:

```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
```

If above command is successful, we can now remove the node from Kubernetes cluster:

```
kubectl delete node <node name>
```

Lastly, to revert of changes made by `kubeadm init` or `kubeadm join`, use `kubeadm reset` command like below:

```
sudo kubeadm reset
```
Sample output:
```
```