# 🚀 Microservices Deployment Blueprint: On-Prem & Cloud-Native

## 🏢 Part 1: On-Premise Setup with VMware + Kubernetes + ArgoCD + Helm

This guide walks you through deploying a 21-module microservices application in a real enterprise, on-premise environment using DevOps best practices. It’s written for beginners, so every step is explained in simple terms.

---

## ⚙️ Prerequisites

Before you start, make sure you have the following:

- 💻 **One physical server** (or more) with **at least 3TB storage**, virtualized using **VMware ESXi** (via **vSphere**)
- 🖥️ Create **virtual machines (VMs)** on this server to simulate different servers (called "nodes")
- ✅ VM OS: Ubuntu 22.04 LTS (recommended)
- ✅ Each VM should have Docker, kubeadm, kubelet, kubectl, and containerd installed

You can install Docker and Kubernetes components like this:
```bash
# Docker
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker

# Kubernetes tools
sudo apt install -y apt-transport-https ca-certificates curl
sudo curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 🧱 Step-by-Step: Setting up Kubernetes On-Prem

### 1. Provision VMs (VMware vSphere)
Create VMs in vSphere:
- 🧠 1 **Master Node** – 4 vCPU, 8GB RAM, 100GB Disk
- 💪 2 or more **Worker Nodes** – 4 vCPU, 8GB RAM, 100GB Disk each

Clone the Ubuntu VM for faster provisioning.

### 2. Initialize Kubernetes Master Node
Run this on the Master Node:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Set up `kubectl` access:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3. Install Pod Network (Flannel)
This lets your containers talk to each other:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 4. Join Worker Nodes
After `kubeadm init`, you’ll get a join command like:
```bash
kubeadm join 192.168.56.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:123abc456def...
```
Run it on each worker node.

This securely connects the worker to the master and allows it to run workloads.

#### Troubleshooting:
- Make sure port `6443` is open (used by kube-apiserver)
- Time sync issue? Install `ntp` or `chrony`
- Container runtime errors? Check `containerd` status

Check if nodes joined:
```bash
kubectl get nodes
```

---

## 🎛️ Deploy Microservices with Helm + ArgoCD

### Folder Structure:
```
charts/
  └── service-a/
  └── service-b/
argo/
  └── applications.yaml
```

### 1. Install ArgoCD (GitOps Tool)
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Access ArgoCD Dashboard
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Visit [http://localhost:8080](http://localhost:8080)

Get ArgoCD admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo
```

### 3. Create Helm Charts
Each microservice needs a chart:
- `Chart.yaml`: Basic info
- `values.yaml`: Config variables
- `templates/deployment.yaml`: Pod spec
- `templates/service.yaml`: Expose app

Example `Chart.yaml`:
```yaml
apiVersion: v2
name: service-a
version: 0.1.0
```

### 4. Deploy via ArgoCD
Create app definition in `argo/applications.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: service-a
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/microservices.git
    targetRevision: HEAD
    path: charts/service-a
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Apply it:
```bash
kubectl apply -f argo/applications.yaml
```

---

## ☁️ Part 2: Cloud-Native Setup

### 🌐 1. Infrastructure as Code (IaC)
Use Terraform to automate cloud resource creation:
```bash
terraform init
terraform plan
terraform apply
```
Define:
- EKS/GKE/AKS clusters
- VPCs, subnets, security groups
- IAM roles and service accounts

### 2. GitOps Workflow
- GitLab CI builds Docker image and pushes to registry
- ArgoCD detects changes in Git and updates Kubernetes

---

## 🔁 CI/CD Pipeline with GitLab

### Sample `.gitlab-ci.yml`
```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  TAG_COMMIT: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  TAG_LATEST: $CI_REGISTRY_IMAGE:latest

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -t $TAG_COMMIT .
    - docker tag $TAG_COMMIT $TAG_LATEST

test:
  stage: test
  script:
    - echo "Run your tests here (JUnit, PyTest, etc.)"

push:
  stage: push
  script:
    - docker push $TAG_COMMIT
    - docker push $TAG_LATEST

deploy:
  stage: deploy
  script:
    - git clone https://gitlab.example.com/infra/k8s-deployments.git
    - yq e -i '.image.tag = "$CI_COMMIT_SHORT_SHA"' k8s-deployments/service-a/values.yaml
    - cd k8s-deployments
    - git config user.name "CI Bot"
    - git config user.email "ci@example.com"
    - git commit -am "chore: update image tag for service-a"
    - git push
```

---

## 🛡️ Security Practices

- 🔍 Use Trivy or Grype to scan Docker images
- 🔐 Use Sealed Secrets or HashiCorp Vault for secrets
- 🔒 Use RBAC (Role Based Access Control)
- 🔥 NetworkPolicies to isolate services
- 📜 Enable audit logs

---

## 📊 Observability Stack

Use Grafana stack for monitoring:
- **Prometheus** for metrics
- **Loki** for logs
- **Tempo** for traces
- **Alertmanager** for notifications

---

## ⚖️ Auto Scaling
- **HPA** (Horizontal Pod Autoscaler) for service scaling
- **Cluster Autoscaler** or **Karpenter** for adding/removing nodes

---

## 🎓 Summary

- Set up K8s cluster with kubeadm and Flannel
- Use Helm to package microservices
- Use ArgoCD for GitOps deployment
- Automate CI/CD with GitLab
- Follow security, observability, and autoscaling best practices

> 🎁 Bonus: This guide is beginner-friendly

