# GitOps Deployment Pipeline using ArgoCD & K3s

This repository contains the Kubernetes declarative manifests and deployment workflows for implementing a complete **GitOps CD Pipeline** on a Lightweight Kubernetes (K3s) cluster using **ArgoCD**.

---

## 🏗️ Architecture & Concept

In this setup, Git acts as the **Single Source of Truth**. Any changes pushed to the Kubernetes manifest files (`deployment.yaml`) in this repository are automatically detected and synchronized to the K3s cluster by ArgoCD.

```text
[ Developer / Admin ] 
        │
        ▼  (git push)
[ GitHub Repository ] ◄── Declarative Manifests
        │
        ▼  (Auto-Sync / Continuous Reconciliation)
[ ArgoCD Controller (K3s) ]
        │
        ▼  (Reconciles Cluster State)
[ Kubernetes Cluster (K3s Pods & Services) ]
🚀 Step-by-Step Implementation Guide
Prerequisites
Linux-based Host Machine / Ubuntu

K3s Kubernetes Cluster installed and running

kubectl CLI configured with cluster admin permissions

GitHub Personal Access Token (PAT) for authenticating git operations

Step 1: K3s Permissions & Environment Setup
Ensure kubectl has administrative access to the K3s cluster config:

Bash
# Set appropriate permissions for the K3s kubeconfig
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# Export the configuration path
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Permanently persist the environment variable
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bashrc
source ~/.bashrc

# Verify cluster admin access
kubectl cluster-info
Step 2: Handle Port Conflicts (Free Port 8080)
If port 8080 is occupied by background services (e.g., Jenkins), stop the conflicting process:

Bash
# Stop Jenkins system service (if running)
sudo systemctl stop jenkins

# Force kill any remaining process running on port 8080
sudo kill -9 $(sudo lsof -t -i:8080)
Step 3: Install ArgoCD on the Kubernetes Cluster
Create a dedicated namespace for ArgoCD and apply the official manifests:

Bash
# Create namespace
kubectl create namespace argocd

# Apply ArgoCD installation manifests
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

# Check pods status until all show 'Running'
kubectl get pods -n argocd
Step 4: Access ArgoCD Web UI & Retrieve Credentials
Bash
# 1. Port-Forward the ArgoCD Server Service to local port 8080
kubectl port-forward svc/argocd-server -n argocd 8080:443
Access UI in browser at: https://localhost:8080

Bash
# 2. Extract initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
Username: admin

Step 5: Configure Local Project & Push Manifests to GitHub
Bash
# Navigate to project directory
cd ~/Devops-project/observability-gitops

# Create deployment.yaml file
cat <<EOF> deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-gitops-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

# Initialize Git and push manifests using GitHub Personal Access Token (PAT)
git init
git add .
git commit -m "Initial commit: Add Nginx deployment manifest"
git branch -M main
git remote add origin https://<YOUR_GITHUB_TOKEN>@[github.com/AfaqAhmad30/observability-gitops.git](https://github.com/AfaqAhmad30/observability-gitops.git)
git push -u origin main
Step 6: Connect Repository to ArgoCD Web UI
Open ArgoCD Web UI (https://localhost:8080).

Click + New App and fill in the parameters:

Application Name: my-gitops-app

Project: default

Sync Policy: Automatic

Repository URL: https://github.com/AfaqAhmad30/observability-gitops.git

Revision: HEAD

Path: .

Cluster URL: https://kubernetes.default.svc

Namespace: default

Click Create.

Step 7: Test Scaling & Verification (GitOps Flow Validation)
1. Update Replicas from 2 to 6
Modify deployment.yaml to scale application pods:

Bash
# Update replicas field in deployment.yaml to 6 and push changes
sed -i 's/replicas: 2/replicas: 6/' deployment.yaml
git add deployment.yaml
git commit -m "Scale up application replicas to 6"
git push origin main
2. Verify Pod Scaling in Cluster
Run the following command to see all 6 pods running on K3s:

Bash
kubectl get pods -l app=web
3. Test Self-Healing (Drift Remediation)
Manually delete a running pod to verify automated recovery:

Bash
kubectl delete pod -l app=web --field-selector status.phase=Running --max-delete-pvc=1
kubectl get pods -l app=web -w
🛠️ Troubleshooting Commands
Bash
# Restart K3s CoreDNS if ArgoCD experiences GitHub DNS resolution issues
kubectl rollout restart deployment coredns -n kube-system

# Check ArgoCD Application Status via CLI
kubectl get applications -n argocd
