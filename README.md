# Jenkins CI/CD Pipeline with DockerHub and Amazon EKS

## 🧱 Prerequisites

* AWS account with admin privileges
* Docker Hub account
* Jenkins server (EC2 or on-prem)
* IAM user/role with EKS and ECR/DockerHub permissions
* `eksctl`, `kubectl`, `awscli`, and `docker` installed on Jenkins node or controller

---

## 🐳 1. Install Docker and Jenkins Docker Plugin

### ✅ Intention:

Install Docker so Jenkins can build container images, and add the Docker plugin so Jenkins can interact with Docker natively in the pipeline.

### Install Docker on Jenkins Server:

```bash
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
newgrp docker
```

### Install Docker Plugin in Jenkins:

1. Navigate to **Jenkins Dashboard → Manage Jenkins → Plugin Manager**.
2. Search for **Docker Pipeline Plugin** and install it.

---

## 🔐 2. Configure DockerHub Credentials in Jenkins

### ✅ Intention:

Allow Jenkins to push Docker images securely to your Docker Hub account.

1. Go to **Jenkins → Manage Jenkins → Credentials → Global**.
2. Add a **Username & Password** credential:

   * ID: `docker-hub-creds`
   * Username: your DockerHub username
   * Password: your DockerHub password or access token

---

## 📦 3. Install AWS & Kubernetes Tools

### ✅ Intention:

Install CLI tools required for provisioning and interacting with the EKS cluster from Jenkins.

```bash
# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

---

## ☸️ 4. Create EKS Cluster

### ✅ Intention:

Use `eksctl` to provision an EKS cluster automatically with worker nodes for deploying your app.

```bash
aws configure  # Provide credentials

eksctl create cluster \
  --name ecommerce-cluster \
  --region us-east-1 \
  --nodegroup-name ecommerce-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

---

## 🔗 5. Connect `kubectl` to EKS

### ✅ Intention:

Configure `kubectl` to interact with the EKS cluster.

```bash
aws eks --region us-east-1 update-kubeconfig --name ecommerce-cluster
kubectl get nodes
```

---

## 🚀 6. Jenkinsfile (CI/CD Pipeline)

### ✅ Intention:

Automate your build → push → deploy process using declarative Jenkins Pipeline.

```groovy
pipeline {
    agent any // Jenkins will run this on any available agent node

    environment {
        DOCKER_IMAGE = 'your-dockerhub-username/my-app' // 👈 Sets Docker image name for reuse
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm // ✅ Pulls source code from GitHub or configured SCM
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build(DOCKER_IMAGE) // ✅ Builds Docker image using Dockerfile in the repo
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-creds') {
                        docker.image(DOCKER_IMAGE).push('latest') // ✅ Pushes image to DockerHub with "latest" tag
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // ✅ Applies Kubernetes manifests to deploy your app to EKS cluster
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
            }
        }
    }

    post {
        success {
            echo '✅ Deployment Successful!' // ✅ Printed on successful pipeline run
        }
        failure {
            echo '❌ Deployment Failed!' // ❌ Printed on failure (e.g., image build or k8s apply fails)
        }
    }
}
```

### 📘 Jenkins Pipeline Syntax Explanation:

* `pipeline {}`: Declares the start of the Jenkins pipeline
* `agent any`: Runs the pipeline on any available agent
* `environment {}`: Defines environment variables like the Docker image name
* `stages {}`: Main block to define different steps (checkout, build, push, deploy)
* `checkout scm`: Pulls the source code from your SCM like GitHub
* `docker.build(...)`: Builds a Docker image using your `Dockerfile`
* `docker.withRegistry(...)`: Authenticates to Docker Hub using stored credentials
* `docker.image(...).push(...)`: Pushes the built Docker image to Docker Hub
* `sh 'kubectl apply ...'`: Deploys your app to Kubernetes using manifests
* `post { success { ... } failure { ... } }`: Defines actions based on pipeline result

---

## 📁 Project Directory Structure

### ✅ Intention:

Keep project files organized for easy management and deployment.

```
CICD/
├── app.py
├── Dockerfile
├── Jenkinsfile
├── requirements.txt
├── deployment.yaml
├── service.yaml
└── templates/
    └── index.html
```

---

## 📦 Dockerfile

### ✅ Intention:

Define how to build your Flask-based e-commerce app inside a Docker image.

```Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

---

## 📜 Notes

* Make sure the Jenkins server has `kubectl` and `eksctl` configured properly
* You can add build triggers such as GitHub webhooks for auto-deploy on code push
* Use `LoadBalancer` type service to expose app publicly
* Secure sensitive credentials using Jenkins Credentials plugin

