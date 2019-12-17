# kubeadmLearnings
### My experience with setting up Kubenetes multi node setup with kubeadm on GCP Compute Engine VMs.

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

#### Installing a pod network add-on
swapoff -a  
kubeadm reset  
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml  
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=[kubeadmin host static ip], copy the join syntax required for worker nodes.  
systemctl status kubelet  
###### For NON-ROOT Users.
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

###### for ROOT user:-
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
