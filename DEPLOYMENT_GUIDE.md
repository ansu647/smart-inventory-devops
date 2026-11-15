# End-to-End AWS EC2 Deployment & Jenkins CI/CD Setup Guide

This guide provides step-by-step instructions for provisioning AWS EC2, installing Jenkins LTS and Docker Engine, configuring the CI/CD pipeline, and deploying the **Smart Inventory Management System**.

---

## 1. AWS EC2 Architecture & Specifications

### Recommended Instance Specification
| Parameter | Setting | Reason |
| :--- | :--- | :--- |
| **Cloud Provider** | Amazon Web Services (AWS) | Primary host |
| **Region** | `us-east-1` (N. Virginia) | Enabled lab environment region |
| **AMI** | Ubuntu Server 24.04 LTS (HVM), SSD Volume Type (`ami-08b275ac6cc5dbefb`) | Stable LTS support |
| **Architecture** | 64-bit (x86_64) | Standard compatibility for Jenkins & Docker |
| **Instance Type** | `t3.medium` (2 vCPU, 4.0 GiB Memory) | Handles concurrent Jenkins builds & Docker runtime |
| **Storage** | 40 GiB gp3 General Purpose SSD | Ample capacity for Docker layers and Jenkins workspace |
| **Key Pair** | RSA SSH key pair (`smart-inventory-key.pem`) | Secure SSH administration |

---

## 2. Security Group Network Rules

Configure the following ingress rules for the instance Security Group:

| Type | Port Range | Protocol | Source | Description |
| :--- | :--- | :--- | :--- | :--- |
| **SSH** | `22` | TCP | `YOUR_MY_IP/32` | Secure SSH console access (restricted) |
| **HTTP** | `80` | TCP | `0.0.0.0/0` | **Public Application Access** |
| **Custom TCP** | `8080` | TCP | `YOUR_MY_IP/32` | Jenkins Dashboard (restricted to admin) |

> **Security Mandate**: Never open port 8080 or port 22 to `0.0.0.0/0`. Only port 80 should be publicly accessible.

---

## 3. Server Initialization (EC2 Ubuntu 24.04)

SSH into your newly launched EC2 instance:
```bash
ssh -i /path/to/your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Update system packages:
```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl wget git fontconfig gnupg software-properties-common ca-certificates
```

---

## 4. Docker Engine Installation on EC2

Install official Docker Engine on Ubuntu:
```bash
# Add Docker's official GPG key:
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin

# Start and enable Docker service
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
```

Verify Docker:
```bash
docker --version
```

---

## 5. Jenkins LTS & Java Installation on EC2

Jenkins LTS requires OpenJDK 21+:
```bash
# Install OpenJDK 21
sudo apt-get install -y openjdk-21-jdk openjdk-21-jre
java -version

# Add Jenkins repository keyring (2026/stable)
sudo install -m 0755 -d /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins LTS
sudo apt-get update
sudo apt-get install -y jenkins

# Enable Jenkins user to execute Docker commands without sudo
sudo usermod -aG docker jenkins

# Restart Jenkins service to apply group permissions
sudo systemctl enable --now jenkins
sudo systemctl restart jenkins
```

Verify Jenkins service:
```bash
sudo systemctl status jenkins
```

Retrieve the initial administrator unlock key:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 6. Jenkins Web Dashboard Setup

1. Open your browser and navigate to:
   ```
   http://<EC2_PUBLIC_IP>:8080
   ```
2. Paste the initial administrator password retrieved above.
3. Select **Install suggested plugins** (includes Git, Pipeline, Credentials).
4. Create your administrator account and finish setup.
5. Verify Python3 & venv tools are available to the Jenkins agent:
   ```bash
   sudo apt-get install -y python3-venv python3-pip
   ```

---

## 7. Configuring the CI/CD Pipeline Job

1. In the Jenkins dashboard, click **New Item**.
2. Enter Item Name: `Smart-Inventory-CICD`
3. Select **Pipeline** and click **OK**.
4. In the Job Configuration:
   - Check **GitHub hook trigger for GITScm polling** (enables real-time push builds via `/github-webhook/`).
   - Check **Poll SCM** under Build Triggers as a fallback: Schedule: `H/2 * * * *`.
5. In the **Pipeline** definition section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/ansu647/smart-inventory-devops.git`
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
6. Click **Save**.

---

## 8. Pipeline Execution & Continuous Deployment

1. The pipeline automatically triggers upon code push to `main` (or click **Build Now**).
2. The pipeline executes 5 stages:
   - **Stage 1 (Checkout Source)**: Clones the latest commit from the `main` branch.
   - **Stage 2 (Environment & Unit Tests)**: Prepares an isolated virtual environment, installs dependencies, and executes all 20 Pytest cases.
   - **Stage 3 (Build Docker Image)**: Builds the production image with `smart-inventory:${BUILD_NUMBER}`.
   - **Stage 4 (Deploy Application)**: 
     - Creates/maintains the persistent named volume `inventory_data`.
     - Gracefully replaces the existing container.
     - Maps host port `80` to container port `5000`.
     - Mounts `inventory_data:/data`.
   - **Stage 5 (Health Verification & Rollback)**:
     - Polls `http://localhost:80/health`.
     - Validates HTTP 200 and healthy JSON payload.
     - Automatically rolls back to the prior working image if a failure is detected.

---

## 9. Persistent Storage Verification

Verify that SQLite database records survive container recreation:
```bash
# Check Docker volumes on EC2
docker volume ls | grep inventory_data

# Inspect volume mount on running container
docker inspect smart-inventory-app --format '{{ range .Mounts }}{{ .Name }} -> {{ .Destination }}{{ end }}'
# Expected output: inventory_data -> /data
```

---

## 10. Troubleshooting Common Issues

### Issue 1: Jenkins pipeline fails at Docker build (`permission denied`)
- **Fix**: The `jenkins` user needs access to `/var/run/docker.sock`.
  ```bash
  sudo usermod -aG docker jenkins
  sudo systemctl restart jenkins
  ```

### Issue 2: Port 80 not responding externally
- **Fix**: Check AWS Security Group inbound rules to ensure TCP port 80 is open to `0.0.0.0/0`.
- Also ensure Ubuntu UFW firewall is either inactive or allows HTTP:
  ```bash
  sudo ufw allow 80/tcp
  ```

### Issue 3: Container unhealthy after startup
- **Fix**: Check container logs:
  ```bash
  docker logs smart-inventory-app
  ```

---

## 11. Teardown & Resource Cleanup

To stop running billable AWS resources after demonstration:
```bash
# Stop application container
docker stop smart-inventory-app
docker rm smart-inventory-app

# Terminate EC2 instance via AWS CLI
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
```

