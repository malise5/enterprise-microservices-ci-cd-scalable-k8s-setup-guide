Certainly! Below is a detailed **README** guide for your students, which covers everything from setting up a Kubernetes cluster from scratch to deploying a simple application (like a web app) on it. This guide assumes that students have basic knowledge of servers and Linux but may be new to Kubernetes.

---

# Kubernetes Cluster Setup and Deployment Guide

## Overview

This guide will walk you through setting up a **Kubernetes cluster** from scratch using **kubeadm**, installing a **CNI plugin** (like **Flannel**), and deploying a simple web application on the cluster. By the end of this guide, you will have a working Kubernetes cluster and a deployed application, running on multiple nodes.

### Prerequisites
- 3 servers (either physical or virtual machines), each with:
  - At least **2 GB of RAM**
  - **2 CPUs**
  - A **Linux-based OS** (Ubuntu 20.04 or CentOS 7+ recommended)
  - **Root privileges** (or `sudo` access)
  - **Internet access** to download Kubernetes components

- Basic knowledge of the **Linux command line** and **networking**.

---

## Table of Contents

1. [Set Up Kubernetes Master Node](#set-up-kubernetes-master-node)
2. [Install Kubernetes on Worker Nodes](#install-kubernetes-on-worker-nodes)
3. [Install a CNI Plugin (Flannel)](#install-a-cni-plugin-flannel)
4. [Join Worker Nodes to the Cluster](#join-worker-nodes-to-the-cluster)
5. [Deploy a Web Application](#deploy-a-web-application)
6. [Verify the Cluster and Application](#verify-the-cluster-and-application)

---

## Step 1: Set Up Kubernetes Master Node

### 1.1. Initialize the Master Node

On the **Master Node (Node 1)**, you will initialize the Kubernetes cluster using `kubeadm`. This command will set up the Kubernetes **control plane** (the components that manage the cluster) and prepare the node to be the master.

- Run the following command to initialize the master node:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

- **Explanation**:
  - `--pod-network-cidr=10.244.0.0/16`: This flag specifies the IP range that the **pods** will use in the cluster. We are using Flannel as our CNI plugin, which will use this CIDR block to assign IPs to pods.

- The output will give you the **join token** and a **discovery token** hash, which you will use later to join worker nodes to the cluster. Example output:

```bash
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now install a pod network (like Calico or Flannel) using the following command:
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

Then you can join any number of worker nodes by running the following on each as root:
  kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:abcdef12345...
```

### 1.2. Set Up `kubectl`

To interact with the cluster from the master node, you need to set up `kubectl`:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now, `kubectl` is configured to interact with the Kubernetes cluster from the master node.

---

## Step 2: Install Kubernetes on Worker Nodes

### 2.1. Install Kubernetes Components on Worker Nodes

On each **worker node (Node 2 and Node 3)**, you need to install **kubeadm**, **kubelet**, and **kubectl**:

1. **Install Docker** (for container runtime):

   On each worker node, run:

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker.io
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

2. **Install Kubernetes packages**:

   ```bash
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt-get update
   sudo apt-get install -y kubeadm kubelet kubectl
   sudo systemctl enable kubelet
   ```

---

## Step 3: Install a CNI Plugin (Flannel)

After the **master node** is initialized, you need to set up a **CNI plugin** for pod networking. We will use **Flannel** in this guide.

### 3.1. Install Flannel on the Master Node

On the **master node**, run the following command to deploy Flannel:

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

This command applies the Flannel network configuration, which ensures that pods across the nodes can communicate with each other.

### 3.2. Verify Flannel Installation

To check the status of the Flannel pods, run:

```bash
kubectl get pods -n kube-system
```

You should see the **Flannel pods** running on the master node and scheduled on worker nodes.

---

## Step 4: Join Worker Nodes to the Cluster

### 4.1. Join Worker Nodes Using `kubeadm join`

On each worker node (Node 2 and Node 3), run the `kubeadm join` command generated earlier when you ran `kubeadm init` on the master node:

```bash
sudo kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:abcdef12345...
```

- **Explanation**: This command joins the worker node to the master node, connecting it to the cluster. The worker nodes will start communicating with the master node and begin running application pods.

### 4.2. Verify Worker Nodes

On the master node, check that the worker nodes have joined the cluster:

```bash
kubectl get nodes
```

You should see all three nodes listed, with their status showing **Ready**:

```bash
NAME           STATUS   ROLES    AGE   VERSION
master-node    Ready    master   10m   v1.25.0
worker-node-1  Ready    <none>   5m    v1.25.0
worker-node-2  Ready    <none>   5m    v1.25.0
```

---

## Step 5: Deploy a Web Application

### 5.1. Create a Simple Web Application Deployment

Now that your cluster is set up and all nodes are connected, you can deploy a simple **web application** (using `nginx` as an example).

1. Create a file named `web-app-deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:latest
        ports:
        - containerPort: 80
```

- **Explanation**:
  - This YAML file defines a deployment of 3 replicas of an `nginx` web server.
  - Kubernetes will schedule these replicas on the worker nodes.

### 5.2. Apply the Deployment

On the master node, run the following command to apply the deployment:

```bash
kubectl apply -f web-app-deployment.yaml
```

This will create the **web app** (nginx) and deploy it across the worker nodes.

### 5.3. Expose the Web Application

To access the web app from outside the cluster, you can expose it using a **Service**:

```bash
kubectl expose deployment web-app --type=LoadBalancer --name=web-app-service
```

- **Explanation**: This command exposes the `web-app` deployment as a **service**, allowing external traffic to reach the application.

### 5.4. Get the External IP

To get the external IP of the service (if using a cloud provider or VM setup):

```bash
kubectl get svc
```

---

## Step 6: Verify the Cluster and Application

### 6.1. Verify the Deployment

Check the status of your deployment:

```bash
kubectl get pods
```

You should see 3 pods running the web application on different worker nodes:

```bash
NAME                        READY   STATUS    RESTARTS   AGE   NODE
web-app-xyz123              1/1     Running   0          5m    worker-node-1
web-app-abc456              1/1     Running   0          5m    worker-node-2
web-app-def789              1/