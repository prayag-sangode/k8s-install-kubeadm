Set Hostname

    sudo hostnamectl set-hostname k8s-master.example.com

    cat /etc/hosts
    192.168.0.131  k8s-master.example.com k8s-master
    192.168.0.132  worker-node1.example.com worker-node1
    192.168.0.133  worker-node2.example.com worker-node2

    sudo dnf install -y git curl vim iproute-tc

    sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
    sudo setenforce 0

    sudo sed -i '/swap/d' /etc/fstab
    sudo swapoff -a

    lsmod | grep br_netfilter
    modprobe br_netfilter
    modprobe overlay
    lsmod | grep overlay
    lsmod | grep br_netfilter

    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF


    sudo cat << EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter
    sudo sysctl --system
    sudo dnf install dnf-utils -y

    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sudo dnf install -y containerd.io
    sudo mkdir -p /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml

    cat  /etc/containerd/config.toml
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true
    	
    sudo systemctl restart containerd
    sudo systemctl enable containerd

    cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude=kubelet kubeadm kubectl
    EOF

    sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    sudo dnf install yum-plugin-versionlock -y
    sudo dnf versionlock kubelet kubeadm kubectl
    sudo systemctl enable kubelet.service
    sudo systemctl start kubelet.service
    sudo systemctl status kubelet

    sudo kubeadm config print init-defaults | tee ClusterConfiguration.yaml
    sudo sed -i '/name/d' ClusterConfiguration.yaml
    sudo sed -i 's/ advertiseAddress: 1.2.3.4/ advertiseAddress: 10.2.40.85/' ClusterConfiguration.yaml
    sudo sed -i 's/ criSocket: \/var\/run\/dockershim\.sock/ criSocket: \/run\/containerd\/containerd\.sock/' ClusterConfiguration.yaml

    sudo cat << EOF | cat >> ClusterConfiguration.yaml
    
    ---
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    cgroupDriver: systemd
    EOF

    sudo kubeadm config print init-defaults 
    sudo kubeadm init --config=ClusterConfiguration.yaml
    
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    kubectl get nodes



    kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
    
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
    
    watch kubectl get pods -A
    
    kubeadm join 192.168.0.131:6443 --token abcdef.0123456789abcdef \
            --discovery-token-ca-cert-hash sha256:ac1f995158f4f7c0d930746f4a04416bc0452f009d6a755b59c44f959c432412
    
    	
    kubeadm token create --print-join-command
    
    kubectl get nodes
    
    kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
    
    kubectl create deploy nginx --image nginx:latest
    
    kubectl get pods
    
    kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --all-namespaces
