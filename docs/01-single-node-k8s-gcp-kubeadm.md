# Create single node Kubernetes cluster using kubeadm on Google Cloud Platform (GCP)

This tutorial assumes you have access to the [Google Cloud Platform](https://cloud.google.com).

## Google Cloud Platform SDK

### Install the Google Cloud SDK

Follow the Google Cloud SDK [documentation](https://cloud.google.com/sdk/) to install and configure the `gcloud` command line utility.

Verify the Google Cloud SDK version is 262.0.0 or higher:

```
gcloud version
```

### Set a Default Compute Region and Zone

This tutorial assumes a default compute region and zone have been configured.

If you are using the `gcloud` command-line tool for the first time `init` is the easiest way to do this:

```
gcloud init
```

Then be sure to authorize gcloud to access the Cloud Platform with your Google user credentials:

```
gcloud auth login
```

Next set a default compute region and compute zone:

```
gcloud config set compute/region us-west1
```

Set a default compute zone:

```
gcloud config set compute/zone us-west1-c
```

### Virtual Private Cloud Network
In this section a dedicated Virtual Private Cloud (VPC) network will be setup to host the Kubernetes cluster. 

Create the k8s-demo custom VPC network: 
```shell script
gcloud compute networks create k8s-network --subnet-mode custom
```
A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the kubernetes subnet in the k8s-demo VPC network:
```shell script
gcloud compute networks subnets create kubernetes \
  --network k8s-network \
  --range 10.240.0.0/24
```

### Firewall Rules
Create a firewall rule that allows internal communication across all protocols:
```shell script
gcloud compute firewall-rules create k8s-allow-internal \
  --allow tcp,udp,icmp \
  --network k8s-network \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```
Create a firewall rule that allows external SSH, ICMP, and HTTPS:
```shell script
gcloud compute firewall-rules create k8s-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network k8s-network \
  --source-ranges 0.0.0.0/0
```

### Compute Instance
Create compute instance which will host the Kubernetes control plane:
````shell script
gcloud compute instances create k8s-master \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-2 \
    --private-network-ip 10.240.0.10 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubeadm-test,controller
````
After waiting for some time, continue to check if the instance is available with below command:
```shell script
gcloud compute instances list
```

### SSH Access
SSH access to the k8s-master compute instances can be gained by running below command:
```shell script
gcloud compute ssh k8s-master
```

### Deploy Docker
Install latest Docker container runtime using following command:
```shell script
curl -sSL get.docker.com | sh
```

After the installation is finished, add current user to the "docker" group to use Docker as a non-root user with command:
```shell script
sudo usermod -aG docker $USER
```

### Enable support for cgroup swap limit capabilities
Edit the file /etc/default/grub.d/50-cloudimg-settings.cfg:
```text
sudo nano /etc/default/grub.d/50-cloudimg-settings.cfg
```
Modify the entry for GRUB_CMDLINE_LINUX_DEFAULT and add cgroup_enable=memory swapaccount=1 like below: 
```text
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0 cgroup_enable=memory swapaccount=1"
```
Save the changes and now update grub with command:
```shell script
sudo update-grub
```
Reboot Kubenetes master server now:
```shell script
sudo reboot
```

### Set up the Docker daemon options
Use cgroup driver as systemd and other Docker deamon options by creating a new file /etc/docker/daemon.json with below command: 
```shell script
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### Enable IP forwarding 
Enable IP forwarding by editing /etc/sysctl.conf:
```text
sudo nano /etc/sysctl.conf
```
and uncomment following line:
```text
#net.ipv4.ip_forward=1
```

Reboot Kubenetes master server now:
```shell script
sudo reboot
```

### Deploy Kubernetes packages
Add Kubernetes APT entry into source with:
```shell script
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

Install required Kubernetes packages:
```shell script
sudo apt update
sudo apt install kubeadm kubectl kubelet
```

### Initialize Kubernetes
Run following command to start creation of Kubernetes cluster:
```shell script
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Sample output:
```text
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
W0529 07:01:50.387903    2395 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.240.0.10]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [10.240.0.10 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [10.240.0.10 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0529 07:02:20.596935    2395 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0529 07:02:20.598850    2395 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 19.002634 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: qime8q.8mpf97fdxxxxxxxx
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.240.0.10:6443 --token qime8q.8mpf97fdxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:8f61ee1955f194f6cc7a6888baf37447b29a86a93b214205154a8abdxxxxxxxx
```
As mentioned in the output, we need to run the commands shown below to allow us to use Kubernetes:
```shell script
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
To use Flannel as a pod network, run following command:
```shell script
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Sample output:
```text
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```
Verify that all pods are healthy by running below command:
```shell script
kubectl get pods --all-namespaces -o wide
```
Sample output:
```text
$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE     IP            NODE         NOMINATED NODE   READINESS GATES
kube-system   coredns-66bff467f8-5qh45               1/1     Running   0          4m33s   10.244.0.2    k8s-master   <none>           <none>
kube-system   coredns-66bff467f8-qs4mb               1/1     Running   0          4m33s   10.244.0.3    k8s-master   <none>           <none>
kube-system   etcd-k8s-master                        1/1     Running   0          4m50s   10.240.0.10   k8s-master   <none>           <none>
kube-system   kube-apiserver-k8s-master              1/1     Running   0          4m49s   10.240.0.10   k8s-master   <none>           <none>
kube-system   kube-controller-manager-k8s-master     1/1     Running   0          4m50s   10.240.0.10   k8s-master   <none>           <none>
kube-system   kube-flannel-ds-amd64-9t287            1/1     Running   0          3m19s   10.240.0.10   k8s-master   <none>           <none>
kube-system   kube-proxy-4rzz2                       1/1     Running   0          4m33s   10.240.0.10   k8s-master   <none>           <none>
kube-system   kube-scheduler-k8s-master              1/1     Running   0          4m50s   10.240.0.10   k8s-master   <none>           <none>
``` 
Allow master/controller to schedule pods by running brlow command:
```shell script
kubectl taint nodes --all node-role.kubernetes.io/master-
```
Sample output:
```text
$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/k8s-master untainted
```
Unless this is done, any deployment will not run on this master. Example output from describe pod:
```text
Events:
  Type     Reason            Age                  From                   Message
  ----     ------            ----                 ----                   -------
  Warning  FailedScheduling  87s (x3 over 2m27s)  default-scheduler      0/1 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
  Normal   Pulling           81s                  kubelet, k8s-master    Pulling image "k8s.gcr.io/echoserver:1.4"
  Normal   Pulled            76s                  kubelet, k8s-master    Successfully pulled image "k8s.gcr.io/echoserver:1.4"
  Normal   Created           74s                  kubelet, k8s-master    Created container echoserver
  Normal   Started           74s                  kubelet, k8s-master    Started container echoserver
```
### Add worker node
In case you want to add worker node to the cluster, you can use command like below:
```shell script
kubeadm join 10.240.0.10:6443 --token qime8q.8mpf97fdxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:8f61ee1955f194f6cc7a6888baf37447b29a86a93b214205154a8abdxxxxxxxx
```
Sample output:
```text
$ sudo kubeadm join 10.240.0.10:6443 --token qime8q.8mpf97fdxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:8f61ee1955f194f6cc7a6888baf37447b29a86a93b214205154a8abdxxxxxxxx
W0529 07:12:00.016796    4465 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
### Reset Kubernetes Cluster
Use kubeadm reset command like below:
```shell script
sudo kubeadm reset
```
Sample output:
```shell script
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
```shell script
sudo rm -f /etc/cni/net.d/10-flannel.conflist
``` 
Reset iptables:
```shell script
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```
Remove kubeconfig file:
```shell script
sudo rm -f $HOME/.kube/config
```

### References

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
https://github.com/kelseyhightower/kubernetes-the-hard-way
https://wiki.learnlinux.tv/index.php/How_to_build_your_own_Raspberry_Pi_Kubernetes_Cluster#Install_flannel_network_driver
https://phoenixnap.com/kb/install-kubernetes-on-ubuntu
