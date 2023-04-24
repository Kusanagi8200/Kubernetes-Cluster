
Kubernetes Cluster

Installing Vagrant by downloading the file from the HashiCorp website and adding the repository to the list of package sources.
Enabling bridged traffic for IPtables by adding a module and sysctl parameters.
Installing CRI-O by adding a module and sysctl parameters, then adding the sources for the CRI-O repositories and installing the necessary packages.
Installing Kubelet, Kubeadm, and Kubectl by adding the Kubernetes repositories to the list of package sources and installing the necessary packages.
Initializing Kubeadm using the API server IP address, the pods network CIDR, and the node name, then configuring the kubelet to use the IP address and CIDR.
Installing the Calico network plugin by applying the corresponding YAML file from GitHub.
Deploying Nginx by applying a corresponding YAML file and configuring a NodePort service.

--> VAGRANT INSTALLATION
https://developer.hashicorp.com/vagrant/downloads

wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg]
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

apt update && apt install vagrant

Cf: Vagrantfile

--> ENABLE IPTABLES BRIDGED TRAFFIC 
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1 EOF

# Apply sysctl params without reboot
sudo sysctl --system

swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true

--> RUNTIME CRI-O
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1

OS="xUbuntu_20.04"
VERSION="1.23"
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/
Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo
apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -

apt-get install cri-o cri-o-runc cri-tools -y
systemctl daemon-reload
systemctl enable crio --now

--> INSTALL KUBEADM & KUBELET & KUBECTL
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-
xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt-get install -y kubelet kubeadm kubectl
apt-get install -y jq
local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty
end else empty end')" cat > /etc/default/kubelet << EOF

IPADDR="192.168.56.10"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"

kubeadm init --apiserver-advertise-address=$IPADDR --apiserver-cert-extra-sans=$IPADDR --pod-network-
cidr=$POD_CIDR --node-name $NODENAME

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get po -n kube-system
kubectl get --raw='/readyz?verbose'
kubectl cluster-info

--> INSTALL CALICO NETWORK PLUGIN
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
kubectl get po -n kube-system

kubeadm join 192.168.56.10:6443 --token XXXXXXXXXXXX \ --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXX
kubectl get nodes

--> DEPLOY NGINX
kubectl apply -f - file.yaml
Cf : deploy.yaml
Cf : nodeport.yaml

kubectl describe deploy 

