# Installing Kubernetes (kubeadm) on AWS EC2 Ubuntu 24.04 with Docker Engine

## Prerequisites

* AWS EC2 Instances:
    - 1 t2.medium instance (Control Plane)
    - 2 t2.micro instances (worker node)
* Create a security group and allow all traffic for now, or you can enable the below ports:
    - Control Plane : 
    ```

        Protocol	Direction	    Port Range	        Purpose	                Used By
        TCP	        Inbound	        6443	       Kubernetes API server	    All
        TCP	        Inbound	        2379-2380      etcd server client API	    kube-apiserver, etcd
        TCP	        Inbound	        10250	       Kubelet API                  Self, Control plane
        TCP	        Inbound	        10259	       kube-scheduler	            Self
        TCP	        Inbound	        10257	       kube-controller-manager	    Self
        Although etcd ports are included in control plane section, you can also host your own etcd cluster externally or on custom ports.
    ```

    - Worker node(s)
    ```
        Protocol	Direction	    Port Range	        Purpose	                Used By
        TCP	        Inbound	        10250	            Kubelet API	            Self, Control plane
        TCP	        Inbound	        10256	            kube-proxy	            Self, Load balancers
        TCP	        Inbound	        30000-32767         NodePort Services†	    All
    ```

**Ensure the MAC address and product_uuid are unique for every node. Kubernetes uses these values to uniquely identify the nodes in the cluster. If these values are not unique, the installation process may fail.**

### Verify the MAC address and product_uuid:
```
ip link
sudo apt install net-tools
ifconfig -a
sudo cat /sys/class/dmi/id/product_uuid
```

### Enable IP forwarding:
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

## Docker Engine Installation

**Note: These instructions assume that you are using the cri-dockerd adapter to integrate Docker Engine with Kubernetes.**

### Remove any old Docker packages:
```
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

### Update the package index:
```
sudo apt-get update
```

### Install required packages:
```
sudo apt-get install ca-certificates curl
```

### Add Docker's official GPG key:
```
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Set up the Docker repository:
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### Install Docker Engine:
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

### Install cri-dockerd:
```
sudo apt update
sudo apt install cri-dockerd
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd_0.3.14.3-0.ubuntu-jammy_amd64.deb
sudo apt install ./cri-dockerd_0.3.14.3-0.ubuntu-jammy_amd64.deb
cri-dockerd --version
sudo apt-get update
```

## Installing kubeadm, kubelet, and kubectl

### Install required packages:
```
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

### Add Kubernetes' official GPG key:
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings kubernetes-apt-keyring.gpg
```

### Add Kubernetes repository:
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

### Install kubeadm, kubelet, and kubectl:
```
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
ip route show
```

## Initializing the Master Node

### Initialize the Kubernetes cluster:
```
sudo kubeadm init --cri-socket=/var/run/cri-dockerd.sock
```

### Set up kubeconfig for the root user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Verify the nodes:
```
kubectl get nodes
```

### Apply a Pod network:
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Get the join command for worker nodes:
```
kubeadm token create --print-join-command
```

### Verify the cluster status:
```
kubectl get nodes
kubectl get pods --all-namespaces
kubeadm token list
```
## Joining Worker Nodes to the Master Node

### Run the join command on each worker node:
```
sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket=/var/run/cri-dockerd.sock
```

### Verify the nodes on the master node:
```
kubectl get nodes
```





**Refer to the official documentation for more details on [Docker Engine][docker-doc] and [Kubernetes][k8s-doc].**

**[docker-doc]: https://docs.docker.com/engine/install/**
**[k8s-doc]: https://kubernetes.io/docs/setup/**
