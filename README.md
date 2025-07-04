# Setup Kubernetes v1.33 using kubeadm onto Ubuntu 25.04
Notes on setup of Kubernetes cluster v1.33 on Ubuntu using kubeadm.

Following the [article](https://medium.com/@Kubway/installing-the-first-node-of-a-kubernetes-cluster-with-kubeadm-c116ab0cc38b) by [Kubway](https://medium.com/@Kubway), I installed [Kubernetes v1.33[(https://kubernetes.io/blog/2025/04/23/kubernetes-v1-33-release/) on Ubuntu 25.04 using [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/). 

## Preparing the linux controller node.
```
# swap removing
sudo swapoff -a
# Verification
free -h

#Add Kernel parameters
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
# Check
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.bridge.bridge-nf-call-iptables

comment out swap in /etc/fstab
reboot
```

## Containerd
```
CONTAINERD_VERSION=2.1.3 # Choose one there https://github.com/containerd/containerd/tags
wget https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz

# Verification
containerd -v

# Configure containerd as a service
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo mv containerd.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Configure containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
# Modify  SystemdCgroup  value in /etc/containerd/config.toml
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
# Modify pause image for compliance with Kubernetes 1.30
sudo sed -i 's/registry.k8s.io\/pause:3.8/registry.k8s.io\/pause:3.10/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Verification
systemctl status containerd
# ctr is istalled with containerd
sudo ctr c ls;sudo ctr t ls;sudo ctr image ls
```

## runc
```
RUNC_VERSION=1.3.0 # Choose one there https://github.com/opencontainers/runc/tags
wget https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

/usr/local/sbin/runc --v
sudo systemctl restart containerd
```

## crictl
```
# crictl installation. Choose one version there https://github.com/kubernetes-sigs/cri-tools/tags
CRICTL_VERSION="v1.33.0" 
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$CRICTL_VERSION/crictl-$CRICTL_VERSION-linux-amd64.tar.gz
     https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.33.0/crictl-v1.33.0-darwin-amd64.tar.gz
sudo tar zxvf crictl-$CRICTL_VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$CRICTL_VERSION-linux-amd64.tar.gz
# endpoint configuration
cat <<EOF | sudo tee /etc/crictl.yaml 
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF
```

## kubelet, kubeadm and kubectl
```
# Products installation
KUBE_VERSION=v1.33 # Choose one there https://kubernetes.io/releases/
# Update the apt package index and install packages needed to use the Kubernetes apt repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Download public signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBE_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# Add the Kubernetes apt repository
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBE_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
# Installation
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Alias and completion configuration
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

## Cluster and first controller
```
kubeadm init --pod-network-cidr=10.245.0.0/16 --service-cidr 10.97.0.0/23
kubeadm init --pod-network-cidr=10.244.0.0/16 --service-cidr=10.97.0.0/12

root@knode204:~# kubeadm init --pod-network-cidr=10.244.0.0/16 --service-cidr=10.97.0.0/12
[init] Using Kubernetes version: v1.33.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [knode204.kozik.net kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.100.204]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [knode204.kozik.net localhost] and IPs [192.168.100.204 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [knode204.kozik.net localhost] and IPs [192.168.100.204 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 1.502287868s
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://192.168.100.204:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-controller-manager is healthy after 2.847149478s
[control-plane-check] kube-scheduler is healthy after 5.001787477s
[control-plane-check] kube-apiserver is healthy after 7.005412233s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node knode204.kozik.net as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node knode204.kozik.net as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: n3hgwl.t9kegotn2ygbgt3o
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.204:6443 --token n3hgwl.t9kegotn2ygbgt3o \
        --discovery-token-ca-cert-hash sha256:ce0c61d8445d778303d26d1cbee4163d775627233b89dcc39d2cfffe698f2655
root@knode204:~#
```
Setup regular user account.
```
jkozik@knode204:~$   mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
[sudo] password for jkozik:
jkozik@knode204:~$
```

## Installing Calico
```
CALICO_VERSION=3.30.2 # Choose one there https://github.com/projectcalico/calico/tags
# Operator installation
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v$CALICO_VERSION/manifests/tigera-operator.yaml
# Get model for creating custom resources
curl  https://raw.githubusercontent.com/projectcalico/calico/v$CALICO_VERSION/manifests/custom-resources.yaml -O
jkozik@knode204:~$ kubectl -n kube-system get pod -l component=kube-controller-manager -o yaml | grep -i cluster-cidr
      - --cluster-cidr=10.244.0.0/16
sed -i 's+192.168.0.0/16+10.244.0.0/16+' custom-resources.yaml
k apply -f custom-resources.yaml
watch kubectl get pods -n calico-system
kubectl patch installation default --type=merge -p '{"spec": {"calicoNetwork": {"bgp": "Disabled"}}}'
jkozik@knode204:~$ kubectl get ippools
NAME                  AGE
default-ipv4-ippool   5m21s
jkozik@knode204:~$ kubectl describe ippools default-ipv4-ippool
Name:         default-ipv4-ippool
Namespace:
Labels:       app.kubernetes.io/managed-by=tigera-operator
Annotations:  <none>
API Version:  projectcalico.org/v3
Kind:         IPPool
Metadata:
  Creation Timestamp:  2025-06-22T20:47:50Z
  Generation:          1
  Resource Version:    18148
  UID:                 818122ba-b3aa-40a7-8588-427982288acc
Spec:
  Allowed Uses:
    Workload
    Tunnel
  Assignment Mode:  Automatic
  Block Size:       26
  Cidr:             10.244.0.0/16
  Ipip Mode:        Never
  Nat Outgoing:     true
  Node Selector:    all()
  Vxlan Mode:       CrossSubnet
Events:             <none>
jkozik@knode204:~$
```

## Installing calicoctl
```
curl -L https://github.com/projectcalico/calico/releases/download/v3.30.2/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
sudo mv ./calicoctl /usr/local/bin/
jkozik@knode204:~$ calicoctl ipam show --show-configuration
+--------------------+-------+
|      PROPERTY      | VALUE |
+--------------------+-------+
| StrictAffinity     | false |
| AutoAllocateBlocks | true  |
| MaxBlocksPerHost   |     0 |
+--------------------+-------+
jkozik@knode204:~$
```
## /etc/host on each node
```
192.168.100.204 knode204
192.168.100.205 knode205
192.168.100.206 knode206
```
## Setup Kubernetes Metrics Server
```
kubectl apply -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/refs/heads/main/lab-setup/manifests/metrics-server/metrics-server.yaml
jkozik@knode204:~$ kubectl top nodes
NAME                 CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
knode204.kozik.net   385m         19%      1310Mi          8%
knode205.kozik.net   58m          2%       507Mi           3%
knode206.kozik.net   4m           0%       444Mi           6%
```
## On controller, install helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
jkozik@knode206:~$ helm version
version.BuildInfo{Version:"v3.18.3", GitCommit:"6838ebcf265a3842d1433956e8a622e3290cf324", GitTreeState:"clean", GoVersion:"go1.24.4"}
jkozik@knode206:~$
```
## Install NFS
Create an NFS storage class.  Following the instructions found at [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner#with-helm), install helm, apply the yaml, and verify the nfs-client storage class.
```
jkozik@knode206:~$ showmount -e 192.168.101.152
Export list for 192.168.101.152:
/home/nfs/pv-2            192.168.100.0/24
/home/nfs/pv-1            192.168.100.0/24
/home/nfs/zabbix/enc      192.168.100.0/24
/home/weewx/nfs/archive   192.168.100.0/24
/home/weewx/nfs/conf      192.168.100.0/24
/home/nfs/kubedata        192.168.100.0/24
/home/weather/public_html 192.168.100.0/24
/home/wjr/public_html     192.168.100.0/24
/var/nfsshare             192.168.100.0/24

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.101.152 \
    --set nfs.path=/home/nfs/kubedata
```
```
jkozik@knode204:~$  showmount -e 192.168.101.152
Export list for 192.168.101.152:
/home/nfs/pv-2            192.168.100.0/24
/home/nfs/pv-1            192.168.100.0/24
/home/nfs/zabbix/enc      192.168.100.0/24
/home/weewx/nfs/archive   192.168.100.0/24
/home/weewx/nfs/conf      192.168.100.0/24
/home/nfs/kubedata        192.168.100.0/24
/home/weather/public_html 192.168.100.0/24
/home/wjr/public_html     192.168.100.0/24
/var/nfsshare             192.168.100.0/24
jkozik@knode204:~$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
"nfs-subdir-external-provisioner" has been added to your repositories
jkozik@knode204:~$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.101.152 \
    --set nfs.path=/home/nfs/kubedata
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Tue Jun 24 10:28:01 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
jkozik@knode204:~$
jkozik@knode204:~$ kubectl get storageclass
NAME         PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   63s
jkozik@knode204:~$
```
## NGINX Ingress Controller
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
```
jkozik@knode204:~$ helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
Release "ingress-nginx" does not exist. Installing it now.
NAME: ingress-nginx
LAST DEPLOYED: Tue Jun 24 11:28:13 2025
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
jkozik@knode204:~$
```
The NGINX Ingress controller needs to be configured for NodePort.  By default it is LoadBalancer.  Edit using the following
```
kubectl edit svc -n ingress-nginx ingress-nginx-controller
```
Look for LoadBalancer and change to type: NodePort
Verify, none as External-IP
```
jkozik@knode204:~/k8sNw.com$ kubectl get service -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.109.11.165    <none>        80:30457/TCP,443:30766/TCP   25h
ingress-nginx-controller-admission   ClusterIP   10.103.110.222   <none>        443/TCP                      25h
jkozik@knode204:~/k8sNw.com$
```
See details in
- [K8S ingress: Deploy internet facing kubernetes ingress with NodePort](https://shafinhasnat97.medium.com/k8s-ingress-deploy-internet-facing-kubernetes-ingress-with-nodeport-1d3f25effc72) 

# References
- [Installing the first node of a Kubernetes cluster with kubeadm.](https://medium.com/@Kubway/installing-the-first-node-of-a-kubernetes-cluster-with-kubeadm-c116ab0cc38b)
- [https://medium.com/@Kubway/installing-the-first-node-of-a-kubernetes-cluster-with-kubeadm-c116ab0cc38b](https://medium.com/@Kubway/installing-and-understanding-calico-on-a-kubernetes-cluster-337fa7cd8ba2)
- [How To Setup Kubernetes Cluster Using Kubeadm](https://devopscube.com/setup-kubernetes-cluster-kubeadm/)

