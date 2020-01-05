# KubernetesClusterCreationviaKubeadmInstallationonUbuntuNCENTOS7INGCP
### References: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

## Prequisites:-
## Control-plane node(s)
  <b>Protocol	Direction	Port Range	Purpose	Used By</b>  
     TCP	    Inbound	  6443*	      Kubernetes API server	All  
     TCP	    Inbound	  2379-2380	  etcd server client API	kube-apiserver, etcd  
     TCP	    Inbound	  10250	      Kubelet API	Self, Control plane  
     TCP	    Inbound	  10251	      kube-scheduler	Self  
     TCP	    Inbound	  10252	      kube-controller-manager	Self

## Worker node(s)
  <b>Protocol	Direction	Port Range	Purpose	Used By</b>  
     TCP	    Inbound	  10250	      Kubelet API	Self, Control plane  
     TCP	    Inbound	  30000-32767	NodePort Services**	All

### My experience with setting up Kubenetes multi node setup with kubeadm on GCP Compute Engine VMs.
### First Setup VMs in GCP for Ubuntu 19.04. [ 1 Master Node and 2 Worker Nodes. ]

### Each GCP Compute Engine VMs must have at least 2 CPUS and 2GB RAM.
### At the time of performing the setup, I chose Ubuntu 19.04 briefly known as "Disco Dingo" as OS for the VMs since this is the latest OS version which was supported by Docker, like for eg, Ubunto 19.10 stable APT resource was not available for Docker.

#### Create 3 VMs using the below comand using Google Cloud SDK.
gcloud beta compute --project [YOUR GOOGLE CLOUD PROJECT ID] ssh --zone [ZONE NAME] "kubadmin"  
gcloud beta compute --project [YOUR GOOGLE CLOUD PROJECT ID] ssh --zone [ZONE NAME] "kubenode1"  
gcloud beta compute --project [YOUR GOOGLE CLOUD PROJECT ID] ssh --zone [ZONE NAME] "kubenode2"

For eg.  In my case:-  
gcloud beta compute --project "ace-daylight-256904" ssh --zone "asia-south1-c" "kubadmin"  
gcloud beta compute --project "ace-daylight-256904" ssh --zone "asia-south1-c" "kubenode1"  
gcloud beta compute --project "ace-daylight-256904" ssh --zone "asia-south1-c" "kubenode2"

#### Switch the iptables tooling to 'legacy' mode, as I am using Ubuntu 19.04, need to switch to legacy mode in all 3 VMs.
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy  
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy  
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy  
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy

#### Installing Docker on Ubuntu on these 3 VMs...
sudo apt install apt-transport-https ca-certificates curl software-properties-common  
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -  
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu <b>disco,</b> stable"  
sudo apt update  
apt-cache policy docker-ce  
sudo apt install docker-ce  
sudo systemctl status docker

#### Installing kubeadm, kubelet and kubectl on these 3 VMs.
sudo apt-get update && sudo apt-get install -y apt-transport-https curl  
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -  
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF  
sudo apt-get update  
sudo apt-get install -y kubelet kubeadm kubectl  
sudo apt-mark hold kubelet kubeadm kubectl

#### Initializing the Master Node in Kubernetes Cluster.
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=[kubeadmin host static ip], copy the join syntax required for worker nodes.

#### Installing a pod network add-on on Master Node.
swapoff -a  
kubeadm reset  
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml   
systemctl status kubelet

###### For NON-ROOT Users, apply the config in the Master node.
mkdir -p $HOME/.kube  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config

###### for ROOT user, apply the config in the Master node.:-
export KUBECONFIG=/etc/kubernetes/admin.conf

##### Adapt Docker config and restart in all 3 VMs.
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

systemctl daemon-reload  
systemctl restart docker

----------------------------------------------------------------------
----------------------------------------------------------------------
### Now Setup VMs in GCP for CentOS 7. [ 1 Master Node and 2 Worker Nodes.]
After the VMs are launched in GCP, I have used Mobaxterm utility to connect 3 VMs.

### Make sure that the dependency at the top has been taken care of.

### Install Docker as root user.
yum install yum-utils device-mapper-persistent-data lvm2  
  
> yum-config-manager --add-repo \  
  https://download.docker.com/linux/centos/docker-ce.repo  
  
> yum update && yum install \  
  containerd.io-1.2.10 \  
  docker-ce-19.03.4 \  
  docker-ce-cli-19.03.4  
  
> mkdir /etc/docker  
  
cat > /etc/docker/daemon.json <<EOF  
{  
  "exec-opts": ["native.cgroupdriver=systemd"],  
  "log-driver": "json-file",  
  "log-opts": {  
    "max-size": "100m"  
  },  
  "storage-driver": "overlay2",  
  "storage-opts": [  
    "overlay2.override_kernel_check=true"  
  ]  
}  
EOF

mkdir -p /etc/systemd/system/docker.service.d

#### Restart Docker
systemctl daemon-reload  
systemctl restart docker

### Installing kubeadm, kubelet and kubectl.
cat > /etc/yum.repos.d/kubernetes.repo <<EOF  
[kubernetes]  
name=Kubernetes  
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64  
enabled=1  
gpgcheck=1  
repo_gpgcheck=1  
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg  
EOF
  
setenforce 0  
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config  
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes  
systemctl enable --now kubelet

lsmod | grep br_netfilter

cat <<EOF >  /etc/sysctl.d/k8s.conf  
net.bridge.bridge-nf-call-ip6tables = 1  
net.bridge.bridge-nf-call-iptables = 1  
EOF  
sysctl --system

### Restart kubelet.  
systemctl daemon-reload  
systemctl restart kubelet

### Get the latest version of kubeadm.
yum update -y

#### Installing a pod network add-on on Master Node.
swapoff -a  
kubeadm reset  
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml   
systemctl status kubelet

### Initializing your control-plane node.
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=[kubeadmin host static ip]

### Join the worker nodes to master node. [you must have a ssh connection to your worker nodes.]
kubeadm join 10.160.0.15:6443 --token ylggs4.dhromymm2fwx1ran \  
     --discovery-token-ca-cert-hash sha256:df9dca8fedd13052e9f20b0d4e0f3a051409924be05608881d8c08fee1b9c925
     
###### For NON-ROOT Users, apply the config in the Master node.
mkdir -p $HOME/.kube  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  
sudo chown $(id -u):$(id -g) $HOME/.kube/config

###### for ROOT user, apply the config in the Master node.:-
export KUBECONFIG=/etc/kubernetes/admin.conf

#### verify.
kubectl get pods --all-namespaces
kubectl get pods --all-namespaces -o wide
kubectl describe pod coredns-6955765f44-cst4h -n kube-system
kubectl run nginx --image=nginx
