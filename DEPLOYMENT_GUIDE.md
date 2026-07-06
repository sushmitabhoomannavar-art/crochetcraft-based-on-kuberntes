# CrochetCraft Complete Deployment Guide (From Scratch on 3 EC2 Instances)

This guide provides an end-to-end walkthrough for manually launching **three AWS EC2 instances** from scratch, configuring a Kubernetes cluster across Master and Worker nodes, and automating the deployment of the **CrochetCraft** Spring Boot application and MySQL database using **Jenkins CI/CD** and **Docker Hub** (`sushmitabhoomannavar-art`).

---

## 🖥️ Step 0: Launching Three EC2 Instances in AWS Console (Manual Setup)

You will need to launch exactly **three EC2 instances**. You can launch them all at once using the same configuration and name them afterward.

### 1. Open AWS EC2 Dashboard
1. Log into your AWS Management Console.
2. Navigate to the **EC2 Dashboard** and click **Launch instances**.

### 2. Configure Instance Settings
- **Name and tags**: Name them `CrochetCraft-Servers` (We will rename individual servers after launch: `Jenkins-Server`, `K8s-Master`, `K8s-Worker`).
- **Number of instances**: Enter **`3`**.
- **Application and OS Images (Amazon Machine Image)**: Select **Ubuntu Server 22.04 LTS (HVM)**, SSD Volume Type.
- **Instance type**: Select **`t3.medium`** (Recommended: 2 vCPU, 4 GB RAM is needed to comfortably run Kubernetes control plane, worker pods, and Jenkins builds). *If on a strict budget, you can use `t3.small` for Worker and Jenkins, but `t3.medium` is required for K8s Master.*
- **Key pair (login)**: Select an existing SSH key pair or click **Create new key pair** (e.g., `crochetcraft-key.pem`). Make sure to download and save this `.pem` file.

### 3. Configure Network & Security Group
In the **Network settings** section, click **Edit** and configure:
- **Auto-assign Public IP**: **Enable**.
- **Firewall (security groups)**: Select **Create security group**. Name it `crochetcraft-k8s-sg` with the following inbound rules:
  - **SSH (22)**: Source `Anywhere (0.0.0.0/0)` or `My IP` (for terminal SSH access).
  - **HTTP (8080)**: Source `Anywhere (0.0.0.0/0)` (for Jenkins Web Dashboard).
  - **Custom TCP (8085)**: Source `Anywhere (0.0.0.0/0)` (for standalone Docker app testing).
  - **Custom TCP (30085)**: Source `Anywhere (0.0.0.0/0)` (Kubernetes NodePort to access the Spring Boot app).
  - **Custom TCP (6443)**: Source `Anywhere (0.0.0.0/0)` or Subnet CIDR (Kubernetes API server communication).
  - **Custom TCP (10250 - 10259)**: Source `Anywhere (0.0.0.0/0)` (Kubelet API and node communication).
  - **Custom TCP (30000 - 32767)**: Source `Anywhere (0.0.0.0/0)` (Kubernetes NodePort range).
  - **All ICMP / Custom UDP (8472) / BGP (179)**: Source `Anywhere (0.0.0.0/0)` (Required for Flannel/Calico pod network communication between nodes).

### 4. Configure Storage
- **Root volume**: Set size to **`20 GB`**, Volume type **`gp3`**.
- Click **Launch instance**.

### 5. Assign Roles to Instances
Once launched, go to your EC2 instances list and rename them:
1. **Instance 1**: Name it **`Jenkins-Server`**
2. **Instance 2**: Name it **`K8s-Master`**
3. **Instance 3**: Name it **`K8s-Worker`**

---

## ☸️ Step 1: Kubernetes Cluster Setup (On EC2 #2 & #3)

Open two terminal tabs and SSH into **both `K8s-Master` (EC2 #2)** and **`K8s-Worker` (EC2 #3)**:
```bash
ssh -i /path/to/crochetcraft-key.pem ubuntu@<PUBLIC_IP>
```

### 1. Configure Kernel Modules & Install K8s Components (Run on BOTH Master and Worker)
```bash
# 1. Load required kernel modules for Kubernetes networking
sudo modprobe overlay
sudo modprobe br_netfilter

# 2. Persist kernel modules and enable IP forwarding / iptables bridge traffic
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# 3. Update system & install Docker and containerd
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl docker.io

# Configure containerd for Kubernetes CRI
sudo rm -f /etc/containerd/config.toml
sudo systemctl restart containerd

# Enable and start Docker
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu

# 4. Add Kubernetes official GPG key and repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 5. Install kubelet, kubeadm, and kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2. Initialize Control Plane (Run on `K8s-Master` ONLY)
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl access for the ubuntu user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI pod network
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
> [!IMPORTANT]
> At the end of `kubeadm init`, you will see a command starting with `kubeadm join ...`. **Copy this command**, as you will need it for the Worker node!

### 3. Allow Pod Scheduling on Master Node (Run on `K8s-Master` ONLY)
To satisfy the requirement that MySQL and Spring Boot run across **both worker and master nodes**, remove the control-plane taint:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 4. Join the Worker Node (Run on `K8s-Worker` ONLY)
Paste the join command copied from step 2 (with `sudo`):
```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```
*Verify on `K8s-Master` by running `kubectl get nodes`. Both nodes should show status `Ready` within 1-2 minutes.*

---

## 🤖 Step 2: Jenkins CI/CD Setup (On EC2 #1)

SSH into your **`Jenkins-Server` (EC2 #1)**:
```bash
ssh -i /path/to/crochetcraft-key.pem ubuntu@<JENKINS_PUBLIC_IP>
```

### 1. Install Java 17, Maven, Docker & Jenkins
```bash
# Install Java, Maven & Docker
sudo apt-get update && sudo apt-get install -y openjdk-17-jdk maven docker.io

# Enable Docker and add ubuntu user
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu

# Add Jenkins GPG key and repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt-get update && sudo apt-get install -y jenkins

# Grant Jenkins user Docker permissions
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 2. Install & Configure `kubectl` for Jenkins
```bash
# Install kubectl
sudo apt-get install -y kubectl

# Create kubeconfig directory for Jenkins
sudo mkdir -p /var/lib/jenkins/.kube
```

Now, copy the Kubernetes admin config from your **`K8s-Master`** node to the **`Jenkins-Server`**:
- On **`K8s-Master`**, view the config: `cat ~/.kube/config` and copy its entire text.
- On **`Jenkins-Server`**, create the file and paste the contents:
```bash
sudo nano /var/lib/jenkins/.kube/config
# (Paste text, then press Ctrl+O, Enter, Ctrl+X to save)

# Fix ownership so Jenkins can use it
sudo chown -R jenkins:jenkins /var/lib/jenkins/.kube
```
*Test access by running: `sudo -u jenkins kubectl get nodes`*

---

## 🔐 Step 3: Configure Jenkins Dashboard & Secrets

1. Open your browser and access Jenkins: `http://<JENKINS_PUBLIC_IP>:8080`.
2. Retrieve the initial admin password from the server:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Complete setup, install suggested plugins, and create an admin user.
4. Add your Docker Hub Credentials:
   - Go to **Manage Jenkins -> Credentials -> System -> Global credentials -> Add Credentials**.
   - **Kind**: Username with password
   - **Username**: `sushmitabhoomannavar-art`
   - **Password**: Your Docker Hub password or Personal Access Token (PAT).
   - **ID**: `docker-hub-credentials` *(Must match exactly)*.
   - Click **Create**.

---

## 🚀 Step 4: Run Automated Pipeline

1. In Jenkins Dashboard, click **New Item**.
2. Enter name `CrochetCraft-K8s`, select **Pipeline**, and click **OK**.
3. Scroll down to the **Pipeline** section:
   - **Definition**: Select **Pipeline script from SCM**.
   - **SCM**: Select **Git**.
   - **Repository URL**: `https://github.com/sushmitabhoomannavar-art/crochetcraft-based-on-kuberntes.git`
   - **Branch Specifier**: `*/main` (or `*/master` depending on default branch).
   - **Script Path**: `Jenkinsfile`
4. Click **Save** and then **Build Now**.

### Pipeline Execution Flow:
1. **Build JAR**: Compiles Spring Boot application (`spring_app_sak`).
2. **Build & Push Docker Image**: Packages the Docker image and pushes `sushmitabhoomannavar-art/crochetcraft-app:${BUILD_NUMBER}` and `:latest` to your Docker Hub.
3. **Deploy to Kubernetes**: Applies MySQL persistent storage and deployment, applies Spring Boot app deployment, and updates the pod image version dynamically.
4. **Verify Rollouts**: Checks status until pods are healthy across both Master and Worker nodes.

---

## 🧪 Step 5: Verify & Access Your Web Application

1. Check pod scheduling across your Master and Worker nodes:
   ```bash
   kubectl get pods -l app=crochetcraft -o wide
   ```
   You will see the pods running across both `K8s-Master` and `K8s-Worker`.
2. Access the application in your web browser using the NodePort `30085` on any of your EC2 Public IPs:
   ```http
   http://<ANY_EC2_PUBLIC_IP>:30085/
   ```
   - **Admin Login**: Username: `admin` | Password: `admin`
