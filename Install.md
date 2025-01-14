edit /etc/hosts
add entries on machine
example
my master ip is 192.168.0.19
worker01 ip is 192.168.0.22

ADD
192.168.0.19 master01
192.168.0.22 worker01

Step 1 - Run this on all the machines
Kubeadm | kubectl | kubelet install

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update -y
sudo apt -y install vim git curl wget kubelet=1.23.0-00 kubeadm=1.23.0-00 kubectl=1.23.0-00
sudo apt-mark hold kubelet kubeadm kubectl

Disable swap - swap should be disabled in order for kubelet to work properly.

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a


sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system


                                            
Setup Containerd
                                            
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

#Install and configure containerd 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update -y
sudo apt install -y containerd.io
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

#Start containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

         
         
sudo kubeadm config images pull --cri-socket /run/containerd/containerd.sock --kubernetes-version v1.23.0

         
Run on Master Only
sudo kubeadm init   --pod-network-cidr=10.244.0.0/16   --upload-certs --kubernetes-version=v1.23.0   --control-plane-endpoint=master01 --ignore-preflight-errors=Mem  --cri-socket /run/containerd/containerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
         
         
Step 3 Run the join command on all the worker nodes
COPY
kubeadm join master01:6443 --token tokenhere \
> --discovery-token-ca-cert-hash sha256:cahere
