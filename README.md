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
        TCP	        Inbound	        2379-2380	   etcd server client API	    kube-apiserver, etcd
        TCP	        Inbound	        10250	       Kubelet API	                Self, Control plane
        TCP	        Inbound	        10259	       kube-scheduler	            Self
        TCP	        Inbound	        10257	       kube-controller-manager	    Self
        Although etcd ports are included in control plane section, you can also host your own etcd cluster externally or on custom ports.
    ```

    - Worker node(s)
    ```
        Protocol	Direction	    Port Range	        Purpose	                Used By
        TCP	        Inbound	        10250	            Kubelet API	            Self, Control plane
        TCP	        Inbound	        10256	            kube-proxy	            Self, Load balancers
        TCP	        Inbound	        30000-32767	        NodePort Services†	    All
    ```