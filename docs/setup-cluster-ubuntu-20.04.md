# Setup Kubernetes Cluster On Ubuntu-20.04

Update packages and disable firewall : 

    sudo ufw disable ; sudo apt-get update
  
Turn off swap space  : 

    sudo swapoff -a; sed -i '/swap/d' /etc/fstab
  
Update hostname of master and worker nodes respectively : 

    sudo hostname kmaster

Give a static ip-address to make sure that node doesn't gets another ip on reboot. Give Address 192.168.200.10 to master and 192.168.200.11, 192.168.200.12 and so on to nodes :

    sudo echo 'network:
    ethernets:
      enp0s3:
        addresses: [192.168.200.10/24]
        gateway4: 192.168.200.1
        nameservers:
          addresses: [4.2.2.2, 8.8.8.8]
        optional: true
    version: 2' >  /etc/netplan/00-installer-config.yaml ;  sudo netplan apply

Map host addresses to names :

    sudo echo '192.168.200.10 kmaster
    192.168.200.11 knode1
    192.168.200.12 knode2' >> /etc/hosts

Install docker :

    sudo apt-get install -y docker.io curl gnupg ; docker --version

Kubernetes setup :

    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
    sudo apt-get install software-properties-common
    sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl 
    sudo echo 'Environment="cgroup-driver=systemd/cgroup-driver=cgroupfs"' >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# On kmaster node

Initiate cluster from master

    sudo kubeadm init --apiserver-advertise-address=192.168.200.10 --pod-network-cidr=192.168.200.0/24
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    kubectl get nodes

Make node ready with Pod Network and bring DNS up

    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    kubectl get nodes
    kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

Nodes must now be ready

Setup Kubernetes Dashboard

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
    kubectl create serviceaccount dashboard -n defaul
    kubectl create clusterrolebinding dashboard-admin -n default \
    --clusterrole=cluster-admin --serviceaccount=default:dashboard
    
Get secret and run dashboard
   
    kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 –decode
    kubectl proxy  --address=192.168.200.10

# Join the worker node

Use the command and token shared while cluster initialization

    sudo kubeadm join 192.168.200.10:6443 --token uz3u7c.p62fs5fb6xqaabtp --discovery-token-ca-cert-hash sha256:653846a3c4970145867d456bbbf6395d1c8b86b105a8c02258a990e51bf00d7b 
