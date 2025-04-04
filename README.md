# 🚀 DevSecOps Pipeline Deployment using Kubernetes

This project demonstrates a **DevSecOps pipeline** built for a Netflix Clone UI, deployed on **Amazon EKS (Elastic Kubernetes Service)** with a complete CI/CD setup using **Jenkins**, **SonarQube**, **Trivy**, **OWASP Dependency-Check**, **Docker**, **Prometheus-Grafana**, and **Argo CD**.

> ⚠️ *Note: The frontend UI was forked from an open-source project: [chiragpuniyanii/nextflix](https://github.com/chiragpuniyanii/nextflix). Focus of this project is on deployment automation and security integrations.*

---

## 🧰 Tools & Technologies Used

| Category          | Tool/Tech                         | Purpose |
|-------------------|-----------------------------------|---------|
| 🐳 Containerization | Docker                           | Containerization |
| ☸️ Orchestration    | Kubernetes                       | App deployment & scaling |
| ⚙️ CI/CD            | Jenkins                          | Automation of build & deployment |
| 🧠 Code Quality     | SonarQube                        | Static Code Analysis |
| 🔐 Security         | OWASP Dependency-Check           | Vulnerability scanning |
| 🚀 GitOps           | Argo CD                          | Continuous deployment |
| 📈 Monitoring       | Prometheus + Grafana             | Real-time monitoring |
| 💻 VCS              | GitHub                           | Source control & Webhook trigger |

---

## 🔧 EC2 Setup

1. **Launch EC2 Instance**  
   Type: `t2.large`

2. **Open Security Group Ports**
   - `22` (SSH)
   - `80` (HTTP)
   - `3000` (Frontend)
   - `8080` (Jenkins)
   - `9090` (Prometheus/Grafana)

3. **Install Docker**
```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker $USER
```

---

## 🐳 Docker Image Creation

1. **Clone Netflix Frontend Code**
```bash
git clone https://github.com/chiragpuniyanii/nextflix.git
cd nextflix
```

2. **Get TMDB API Key**
   - Sign up on [https://www.themoviedb.org/](https://www.themoviedb.org/)
   - Use API key in Docker build.

3. **Build Docker Image**
```bash
docker build --build-arg API_KEY=2af0904de8242d48e8527eeedc3e19d9 -t netflix .
```
> 🔑 *Feel free to use this API key for testing.*

---

## ⚙️ Jenkins Setup

1. **Install Java & Jenkins**
```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre
wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

---

## 🔍 Static Code Analysis

### 🐳 SonarQube Setup (via Docker)
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
- Open [http://your-ip:9000](http://your-ip:9000)
- Create token under admin -> user settings

### 🔐 Trivy Setup
[Install Steps](https://aquasecurity.github.io/trivy/v0.55/getting-started/installation/)
```bash
sudo apt install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy
```

### 🛡️ OWASP Dependency Check
- Add `OWASP Dependency-Check` plugin in Jenkins
- Install and configure under Jenkins Tools as `DP-Check`

---

## 🔁 Jenkins Pipeline Example

```groovy
pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/chiragpuniyanii/nextflix.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix
                    """
                }
            }
        }

        stage('OWASP Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit -n', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy Image Scan") {
            steps {
                script {
                    try {
                        sh "trivy image zomato > trivyimage.txt"
                        input(message: "Proceed with image scan results?", ok: "Yes")
                    } catch (Exception e) {
                        input(message: "Trivy image scan failed. Proceed anyway?", ok: "Yes")
                    }
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                sh "docker build --build-arg API_KEY=2af0904de8242d48e8527eeedc3e19d9 -t netflix ."
            }
        }

        stage("Trivy Scan on Built Image") {
            steps {
                sh "trivy image netflix > trivyimage.txt"
                input(message: "Proceed with built image analysis?", ok: "Proceed")
            }
        }

        stage("Push to DockerHub") {
            steps {
                withDockerRegistry(credentialsId: 'docker') {
                    sh "docker tag netflix chiragg619/netflix:latest"
                    sh "docker push chiragg619/netflix:latest"
                }
            }
        }
    }
}
```

---

## ☸️ Kubernetes on EKS

> 📌 Follow EKS setup from this repo: [AWS-EKS-Cluster-Setup](https://github.com/chiragpuniyanii/AWS-EKS-Cluster-Setup-with-MySQL-and-Load-Balancer.git)

### Prometheus & Grafana
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus # Change to LoadBalancer
kubectl edit svc stable-grafana -n prometheus # Change to LoadBalancer
```
- Default Grafana login:
  - **Username:** `admin`
  - **Password:** `prom-operator`
- Import dashboard ID: `12740`

### ArgoCD Setup
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd -o json
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

- Access dashboard via LoadBalancer URL
- Create application:
  - **Repo URL:** your GitHub repo
  - **Path:** ConfigurationFiles (folder)

---

## ✅ Final Steps

- Use `kubectl get all` to verify services are running.
- LoadBalancer IP from ArgoCD and Grafana can be used to access frontend & monitoring.

---

## 📎 Useful Links

- Project Repo (Forked UI): https://github.com/chiragpuniyanii/nextflix
- EKS Setup Guide: https://github.com/chiragpuniyanii/AWS-EKS-Cluster-Setup-with-MySQL-and-Load-Balancer
- Docker Image: https://hub.docker.com/repository/docker/chiragg619/netflix

---

## 👤 Author
**Chirag Puniyani**  
🔗 [GitHub](https://github.com/chiragpuniyanii)  
📧 chiragpuniyani7@gmail.com
