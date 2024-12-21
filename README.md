# Kubernetes Cluster Setup on EC2 Instances (Manual)

This guide outlines the steps to set up a Kubernetes cluster manually on EC2 instances without using EKS. The cluster will have one master node and two worker nodes using Ubuntu as the operating system. The required security group rules are defined in the provided image.

---

## Prerequisites

1. **EC2 Instances**: Launch three EC2 instances of type `t2.medium` running Ubuntu OS.

   - Assign a security group with the following inbound rules:
     - **SSH (TCP/22)**: Allow from your IP.
     - **HTTP (TCP/80)**: Allow from anywhere.
     - **HTTPS (TCP/443)**: Allow from anywhere.
     - **Custom TCP (30000-32767)**: Allow from anywhere (NodePort range).
     - **Custom TCP (6443)**: Allow from anywhere (Kubernetes API server).
     - **Custom TCP (3000-10000)**: Allow as needed.
2. **IAM Role**: Attach an IAM role with sufficient permissions for managing EC2.
3. **Shell Access**: SSH into all three EC2 instances.

---

## Step 1: Configure All Nodes

Run the following script on **all nodes** (master and workers):

```bash
#!/bin/bash

# Update and upgrade the system
sudo apt update && sudo apt upgrade -y

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# Install dependencies
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repository and install containerd
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install containerd.io -y

# Configure containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Add Kubernetes repository
sudo apt update
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes components
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Step 2: Initialize the Master Node

On the master node, run the following commands:

1. **Initialize the control plane**:

   ```bash
   sudo kubeadm init
   ```

   Copy the `kubeadm join` command from the output. This will be used later to connect worker nodes.
2. **Set up kubeconfig**:

   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
3. **Install a network plugin**:

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
   ```
4. **Verify the cluster**:

   ```bash
   kubectl get pods -n kube-system
   kubectl get nodes
   ```

---

## Step 3: Join Worker Nodes

On each worker node, run the `kubeadm join` command copied from the master node's `kubeadm init` output. Example:

```bash
sudo kubeadm join <master-node-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

After joining, verify the nodes on the master node:

```bash
kubectl get nodes
```

---

## Creating a Service Account for CI/CD Tools

To set up a Kubernetes user with appropriate permissions for deployments and programmatic access, follow these steps:

### Step 1: Create a Namespace (Optional)

Create a namespace to organize your applications:

```bash

kubectl create namespace webapps

```

### Step 2: Define a Service Account

Create a `ServiceAccount` YAML manifest (`service-account.yaml`):

```yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-user
  namespace: webapps

```
Apply the manifest:

```bash

kubectl apply -f service-account.yaml

```

### Step 3: Assign Roles

Create a `Role` YAML manifest (`role.yaml`) to define permissions:

```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingres
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicaset
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

```

Apply the manifest:

```bash

kubectl apply -f role.yaml

```

### Step 4: Bind Roles to the Service Account

Create a `RoleBinding` YAML manifest (`rolebinding.yaml`) to bind the role to the service account:

```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: k8s-user

```

Apply the manifest:

```bash

kubectl apply -f rolebinding.yaml

```

### Step 5: Generate a Token for the Service Account

Create a `Secret` YAML manifest (`secret.yaml`) to generate a token for the service account:

```yaml

apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: k8s-secrets
  namespace: webapps
  annotations:
    kubernetes.io/service-account.name: k8s-user

```

Apply the manifest:

```bash

kubectl apply -f secret.yaml

```

Retrieve the token:

```bash

kubectl describe secret k8s-secrets

```

Save the token securely, as it will be used for programmatic access to the Kubernetes cluster.

---

## Summary

After completing these steps:

1. A service account named `k8s-user` is created in the `webapps` namespace.
2. A role (`app-role`) with permissions to manage Kubernetes resources is defined.
3. The role is bound to the `k8s-user` service account using a role binding.
4. A token is generated for the service account, which can be used for programmatic access (e.g., via Jenkins).

This setup ensures proper permissions and access control for your CI/CD tool to deploy applications and manage resources on the Kubernetes cluster.

---
