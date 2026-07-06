# CrochetCraft: Spring Boot + MySQL on Kubernetes with Jenkins CI/CD & Docker Hub

This repository contains the **CrochetCraft** application, a Spring Boot Java web application backed by a MySQL database, fully configured for automated cloud deployment on a **multi-node Kubernetes cluster** spanning **three AWS EC2 servers** using a **Jenkins CI/CD pipeline** and **Docker Hub** (`sushmitabhoomannavar-art`).

---

## 🏗️ Architecture & Server Allocation (3 EC2 Servers)

The infrastructure is distributed across three dedicated EC2 servers:
1. **`Jenkins-Server` (EC2 #1)**: CI/CD automation engine. Runs Maven builds, containerizes applications, pushes Docker images to Docker Hub, and executes `kubectl` deployment commands against Kubernetes.
2. **`K8s-Master` (EC2 #2)**: Kubernetes Control Plane (Master Node). Untainted to allow scheduling application pods across **both master and worker nodes**.
3. **`K8s-Worker` (EC2 #3)**: Kubernetes Worker Node. Runs application and database container pods alongside the master node.

---

## 🖥️ Step 0: Launch Three EC2 Instances in AWS Console (From Scratch)

Launch exactly **three EC2 instances** using the same base configuration in your AWS Management Console:

### 1. Open AWS EC2 Dashboard
1. Log into your **AWS Management Console**.
2. Navigate to the **EC2 Dashboard** and click **Launch instances**.

### 2. Configure Instance Settings
- **Name and tags**: Enter `CrochetCraft-Servers` *(We will rename individual servers after launch)*.
- **Number of instances**: Enter **`3`**.
- **Application and OS Images (AMI)**: Select **Ubuntu Server 22.04 LTS (HVM)**, SSD Volume Type.
- **Instance type**: Select **`t3.medium`** *(Recommended: 2 vCPU, 4 GB RAM is required to run the Kubernetes Control Plane, worker pods, and Jenkins builds smoothly)*.
- **Key pair (login)**: Select an existing SSH key pair or click **Create new key pair** (e.g., `crochetcraft-key.pem`). Download and save this `.pem` file.

### 3. Configure Network & Security Group
In the **Network settings** section, click **Edit** and configure:
- **Auto-assign Public IP**: **Enable**.
- **Firewall (security groups)**: Select **Create security group**. Name it `crochetcraft-k8s-sg` with the following inbound rules:
  - **SSH (22)**: Source `Anywhere (0.0.0.0/0)` or `My IP` *(for terminal SSH access)*.
  - **HTTP (8080)**: Source `Anywhere (0.0.0.0/0)` *(for Jenkins Web Dashboard)*.
  - **Custom TCP (30085)**: Source `Anywhere (0.0.0.0/0)` *(Kubernetes NodePort to access the Spring Boot app)*.
  - **Custom TCP (6443)**: Source `Anywhere (0.0.0.0/0)` *(Kubernetes API server communication)*.
  - **Custom TCP (10250 - 10259 & 30000 - 32767)**: Source `Anywhere (0.0.0.0/0)` *(Kubelet API & NodePort range)*.
  - **Custom UDP (8472) / BGP (179) / ICMP**: Source `Anywhere (0.0.0.0/0)` *(Required for Flannel/Calico pod network communication between K8s nodes)*.

### 4. Configure Storage & Launch
- **Root volume**: Set size to **`20 GB`**, Volume type **`gp3`**.
- Click **Launch instance**.

### 5. Rename Your Instances
Once launched, go to your EC2 instances list and rename them for clarity:
1. **Instance 1**: Name it **`Jenkins-Server`**
2. **Instance 2**: Name it **`K8s-Master`**
3. **Instance 3**: Name it **`K8s-Worker`**

---

## ☸️ Step 1: Kubernetes Cluster Setup (On EC2 #2 & #3)

Open two terminal windows on your local computer and SSH into **both `K8s-Master` (EC2 #2)** and **`K8s-Worker` (EC2 #3)**:
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
> At the end of `kubeadm init`, the output will display a join command starting with `kubeadm join ...`. **Copy and save this command**, as you will need it for the Worker node!

### 3. Allow Pod Scheduling on Master Node (Run on `K8s-Master` ONLY)
To satisfy the requirement that MySQL and Spring Boot run across **both worker and master nodes**, remove the default scheduling taint from the master:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 4. Join the Worker Node (Run on `K8s-Worker` ONLY)
Paste the join command copied from step 2 (with `sudo` prepended):
```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```
*Verify on `K8s-Master` by running `kubectl get nodes`. Both nodes should show status `Ready` within 1–2 minutes.*

---

## 🤖 Step 2: Jenkins CI/CD Setup (On EC2 #1)

Open a third terminal window and SSH into your **`Jenkins-Server` (EC2 #1)**:
```bash
ssh -i /path/to/crochetcraft-key.pem ubuntu@<JENKINS_PUBLIC_IP>
```

### 1. Install Java 21, Maven, Docker & Jenkins
```bash
# Install Java 21 (Required by Jenkins 2.555+), Maven & Docker
sudo apt-get update && sudo apt-get install -y openjdk-21-jdk fontconfig maven docker.io

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
# Add Kubernetes official GPG key and repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubectl
sudo apt-get update && sudo apt-get install -y kubectl

# Create kubeconfig directory for Jenkins
sudo mkdir -p /var/lib/jenkins/.kube
```

Now, copy the Kubernetes admin config from your **`K8s-Master`** node to the **`Jenkins-Server`**:
- On **`K8s-Master`**, view the config: `cat ~/.kube/config` and copy its entire text.
- On **`Jenkins-Server`**, create the file and paste the text:
```bash
sudo nano /var/lib/jenkins/.kube/config
# (Paste text, then press Ctrl+O, Enter, Ctrl+X to save)

# Fix ownership so Jenkins can use it
sudo chown -R jenkins:jenkins /var/lib/jenkins/.kube
```
*Test access on your Jenkins server by running: `sudo -u jenkins kubectl get nodes`*

---

## 🔐 Step 3: Configure Jenkins Dashboard & Secrets

1. Open your web browser and navigate to Jenkins: `http://<JENKINS_PUBLIC_IP>:8080`.
2. Retrieve the initial admin password from your terminal on **`Jenkins-Server`**:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Complete the setup wizard, install suggested plugins, and create your admin account.
4. Add your Docker Hub Credentials:
   - Go to **Manage Jenkins -> Credentials -> System -> Global credentials -> Add Credentials**.
   - **Kind**: Username with password
   - **Username**: `sushmitabhoomannavar-art`
   - **Password**: Your Docker Hub password or Personal Access Token (PAT).
   - **ID**: `docker-hub-credentials` *(Must match exactly as configured in the `Jenkinsfile`)*.
   - Click **Create**.

---

## 🚀 Step 4: Run Automated Pipeline

1. In the Jenkins Dashboard, click **New Item**.
2. Enter the name `CrochetCraft-K8s`, select **Pipeline**, and click **OK**.
3. Scroll down to the **Pipeline** section:
   - **Definition**: Select **Pipeline script from SCM**.
   - **SCM**: Select **Git**.
   - **Repository URL**: `https://github.com/sushmitabhoomannavar-art/crochetcraft-based-on-kuberntes.git`
   - **Branch Specifier**: `*/main` *(or `*/master` depending on your default git branch)*.
   - **Script Path**: `Jenkinsfile`
4. Click **Save** and then **Build Now**.

### What the Pipeline Automates (`Jenkinsfile`):
1. **Build JAR**: Compiles the Spring Boot application using Maven (`mvn clean package -DskipTests`).
2. **Build & Push Docker Image**: Containerizes the application and pushes `sushmitabhoomannavar-art/crochetcraft-app:${BUILD_NUMBER}` and `:latest` to your Docker Hub repository.
3. **Deploy to Kubernetes**: Applies MySQL persistent storage (`/mnt/data/mysql`) and deployments, applies the Spring Boot application manifest, and updates the deployment image dynamically using `kubectl set image`.
4. **Verify Rollouts**: Monitors rollout status until pods are healthy across both nodes.

---

## 🧪 Step 5: Verify & Access Your Web Application

1. Check pod scheduling across your Master and Worker nodes:
   ```bash
   kubectl get pods -l app=crochetcraft -o wide
   ```
   You will see both `mysql-container` and `springboot-app` distributed across your **`K8s-Master`** and **`K8s-Worker`** instances.
2. Access your web application in your browser using NodePort `30085` on **any** of your EC2 Public IPs:
   ```http
   http://<ANY_EC2_PUBLIC_IP>:30085/
   ```
   - **Admin Login**: Username: `admin` | Password: `admin`

---

## 📁 Repository Structure

- `k8s/mysql-deployment.yaml` — PersistentVolume (`manual` class), PersistentVolumeClaim, Secret, Deployment, and ClusterIP Service for MySQL 8.0.
- `k8s/app-deployment.yaml` — Spring Boot Deployment referencing Docker Hub (`sushmitabhoomannavar-art/crochetcraft-app:latest`) and NodePort Service (`30085`).
- `k8s/crochetcraft-k8s-all.yaml` — Combined single-file Kubernetes deployment manifest.
- `Jenkinsfile` — Root CI/CD automation pipeline script.
- `DEPLOYMENT_GUIDE.md` — Extended walkthrough documentation.