Lab 16: Integrating Kubernetes with CI/CD Pipelines
Objectives
By the end of this lab, you will be able to:

• Set up a complete CI/CD pipeline using GitHub Actions • Automatically build and push Docker images to a container registry • Deploy applications to a Kubernetes cluster through automation • Implement automated testing and deployment verification • Configure rollback procedures for failed deployments • Understand the integration between CI/CD tools and Kubernetes • Apply best practices for container-based deployment workflows

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and images • Familiarity with Kubernetes concepts (pods, deployments, services) • Knowledge of Git version control system • Understanding of YAML configuration files • Basic command-line interface experience • GitHub account (free tier is sufficient)

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 20.04 LTS with Docker pre-installed • kubectl command-line tool configured • Minikube for local Kubernetes cluster • Git client and text editors • All necessary networking configurations

Task 1: Setting Up the Development Environment
Subtask 1.1: Initialize the Project Repository
First, we'll create a sample application and set up version control.

Create a new directory for your project:
mkdir k8s-cicd-lab
cd k8s-cicd-lab
Initialize a Git repository:
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"
Create a simple Node.js application:
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Kubernetes CI/CD Pipeline!',
    version: process.env.APP_VERSION || '1.0.0',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App running on port ${port}`);
});
EOF
Create package.json file:
cat > package.json << 'EOF'
{
  "name": "k8s-cicd-app",
  "version": "1.0.0",
  "description": "Sample app for Kubernetes CI/CD integration",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Running tests...\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
Subtask 1.2: Create Dockerfile
Create a Dockerfile to containerize the application:

cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY . .

EXPOSE 3000

USER node

CMD ["npm", "start"]
EOF
Subtask 1.3: Create Kubernetes Manifests
Create a deployment manifest:
mkdir k8s-manifests
cat > k8s-manifests/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-app
  labels:
    app: cicd-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cicd-app
  template:
    metadata:
      labels:
        app: cicd-app
    spec:
      containers:
      - name: cicd-app
        image: cicd-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: APP_VERSION
          value: "1.0.0"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
Create a service manifest:
cat > k8s-manifests/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: cicd-app-service
spec:
  selector:
    app: cicd-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
EOF
Task 2: Setting Up CI/CD Pipeline with GitHub Actions
Subtask 2.1: Create GitHub Actions Workflow
Create the GitHub Actions directory structure:
mkdir -p .github/workflows
Create the main CI/CD workflow file:
cat > .github/workflows/ci-cd.yaml << 'EOF'
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: docker.io
  IMAGE_NAME: cicd-app

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run tests
      run: npm test

    - name: Run security audit
      run: npm audit --audit-level high

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ secrets.DOCKER_USERNAME }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    - name: Configure kubectl
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config

    - name: Update deployment image
      run: |
        sed -i 's|image: cicd-app:latest|image: ${{ needs.build-and-push.outputs.image-tag }}|g' k8s-manifests/deployment.yaml

    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f k8s-manifests/
        kubectl rollout status deployment/cicd-app --timeout=300s

    - name: Verify deployment
      run: |
        kubectl get pods -l app=cicd-app
        kubectl get services cicd-app-service
EOF
Subtask 2.2: Create Rollback Workflow
Create a separate workflow for handling rollbacks:

cat > .github/workflows/rollback.yaml << 'EOF'
name: Rollback Deployment

on:
  workflow_dispatch:
    inputs:
      revision:
        description: 'Revision number to rollback to (leave empty for previous)'
        required: false
        type: string

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    - name: Configure kubectl
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config

    - name: Rollback deployment
      run: |
        if [ -n "${{ github.event.inputs.revision }}" ]; then
          kubectl rollout undo deployment/cicd-app --to-revision=${{ github.event.inputs.revision }}
        else
          kubectl rollout undo deployment/cicd-app
        fi

    - name: Wait for rollback completion
      run: |
        kubectl rollout status deployment/cicd-app --timeout=300s

    - name: Verify rollback
      run: |
        kubectl get pods -l app=cicd-app
        kubectl describe deployment cicd-app
EOF
Task 3: Setting Up Local Kubernetes Environment
Subtask 3.1: Start Minikube Cluster
Start Minikube with appropriate resources:
minikube start --driver=docker --memory=4096 --cpus=2
Verify cluster is running:
kubectl cluster-info
kubectl get nodes
Enable necessary addons:
minikube addons enable ingress
minikube addons enable metrics-server
Subtask 3.2: Configure Docker Environment
Configure Docker to use Minikube's Docker daemon:

eval $(minikube docker-env)
This allows you to build images directly in Minikube's Docker environment.

Task 4: Manual Testing and Deployment
Subtask 4.1: Build and Test Locally
Before setting up the full CI/CD pipeline, let's test everything manually:

Build the Docker image:
docker build -t cicd-app:v1.0.0 .
Test the container locally:
docker run -d -p 3000:3000 --name test-app cicd-app:v1.0.0
Verify the application is working:
curl http://localhost:3000
curl http://localhost:3000/health
Stop and remove the test container:
docker stop test-app
docker rm test-app
Subtask 4.2: Deploy to Kubernetes Manually
Update the deployment manifest to use the local image:
sed -i 's|image: cicd-app:latest|image: cicd-app:v1.0.0|g' k8s-manifests/deployment.yaml
sed -i 's|imagePullPolicy: Always|imagePullPolicy: Never|g' k8s-manifests/deployment.yaml
Deploy the application:
kubectl apply -f k8s-manifests/
Check deployment status:
kubectl get deployments
kubectl get pods
kubectl get services
Wait for deployment to be ready:
kubectl rollout status deployment/cicd-app
Subtask 4.3: Test the Deployed Application
Get the service URL:
minikube service cicd-app-service --url
Test the application (replace URL with the output from previous command):
SERVICE_URL=$(minikube service cicd-app-service --url)
curl $SERVICE_URL
curl $SERVICE_URL/health
Task 5: Setting Up GitHub Repository and Secrets
Subtask 5.1: Create GitHub Repository
Add all files to Git:
git add .
git commit -m "Initial commit: Add application and CI/CD configuration"
Create a new repository on GitHub (through the web interface):

Go to https://github.com
Click "New repository"
Name it "k8s-cicd-lab"
Make it public or private as preferred
Don't initialize with README (we already have files)
Push to GitHub:

git remote add origin https://github.com/YOUR_USERNAME/k8s-cicd-lab.git
git branch -M main
git push -u origin main
Subtask 5.2: Configure GitHub Secrets
You need to set up the following secrets in your GitHub repository:

Go to your repository on GitHub
Click Settings → Secrets and variables → Actions
Add the following secrets:
DOCKER_USERNAME: Your Docker Hub username DOCKER_PASSWORD: Your Docker Hub password or access token KUBECONFIG: Base64 encoded kubeconfig file

To get the base64 encoded kubeconfig:

cat ~/.kube/config | base64 -w 0
Task 6: Testing the Complete CI/CD Pipeline
Subtask 6.1: Trigger the Pipeline
Make a change to the application:
sed -i 's/version: process.env.APP_VERSION || '\''1.0.0'\''/version: process.env.APP_VERSION || '\''1.1.0'\''/g' app.js
Commit and push the change:
git add app.js
git commit -m "Update application version to 1.1.0"
git push origin main
Monitor the GitHub Actions workflow:
Go to your repository on GitHub
Click the "Actions" tab
Watch the workflow execution
Subtask 6.2: Verify Automated Deployment
Check if the deployment was updated:
kubectl get deployments
kubectl describe deployment cicd-app
Verify the new version is running:
kubectl get pods
SERVICE_URL=$(minikube service cicd-app-service --url)
curl $SERVICE_URL
Task 7: Testing Rollback Procedures
Subtask 7.1: Simulate a Failed Deployment
Create a broken version of the application:
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

// Intentionally broken code
app.get('/', (req, res) => {
  // This will cause an error
  nonExistentFunction();
  res.json({
    message: 'This version is broken!',
    version: '2.0.0'
  });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App running on port ${port}`);
});
EOF
Commit and push the broken version:
git add app.js
git commit -m "Deploy broken version 2.0.0 (for rollback testing)"
git push origin main
Subtask 7.2: Monitor the Failed Deployment
Watch the deployment status:
kubectl rollout status deployment/cicd-app --timeout=60s
Check pod status:
kubectl get pods
kubectl describe pods -l app=cicd-app
Subtask 7.3: Perform Manual Rollback
Check rollout history:
kubectl rollout history deployment/cicd-app
Rollback to previous version:
kubectl rollout undo deployment/cicd-app
Verify rollback success:
kubectl rollout status deployment/cicd-app
SERVICE_URL=$(minikube service cicd-app-service --url)
curl $SERVICE_URL
Subtask 7.4: Test Automated Rollback via GitHub Actions
Go to your GitHub repository
Click Actions → Rollback Deployment → Run workflow
Leave revision empty to rollback to previous version
Click "Run workflow"
Task 8: Advanced Pipeline Features
Subtask 8.1: Add Environment-Specific Deployments
Create separate deployment configurations for different environments:

mkdir -p k8s-manifests/environments/{staging,production}

# Staging environment
cat > k8s-manifests/environments/staging/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../deployment.yaml
- ../../service.yaml

namePrefix: staging-
namespace: staging

replicas:
- name: cicd-app
  count: 1

patches:
- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/env/0/value
      value: "staging-1.0.0"
  target:
    kind: Deployment
    name: cicd-app
EOF

# Production environment
cat > k8s-manifests/environments/production/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../deployment.yaml
- ../../service.yaml

namePrefix: prod-
namespace: production

replicas:
- name: cicd-app
  count: 5

patches:
- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/env/0/value
      value: "production-1.0.0"
  target:
    kind: Deployment
    name: cicd-app
EOF
Subtask 8.2: Create Namespaces
kubectl create namespace staging
kubectl create namespace production
Subtask 8.3: Deploy to Multiple Environments
# Deploy to staging
kubectl apply -k k8s-manifests/environments/staging/

# Deploy to production
kubectl apply -k k8s-manifests/environments/production/

# Verify deployments
kubectl get deployments --all-namespaces
Troubleshooting Common Issues
Issue 1: Docker Build Failures
Problem: Docker build fails with permission errors Solution:

sudo usermod -aG docker $USER
newgrp docker
Issue 2: Kubernetes Deployment Stuck
Problem: Pods remain in Pending state Solution:

kubectl describe pods -l app=cicd-app
kubectl get events --sort-by=.metadata.creationTimestamp
Issue 3: Service Not Accessible
Problem: Cannot access the application through the service Solution:

kubectl port-forward service/cicd-app-service 8080:80
curl http://localhost:8080
Issue 4: GitHub Actions Workflow Fails
Problem: CI/CD pipeline fails due to missing secrets Solution:

Verify all required secrets are set in GitHub repository settings
Check secret names match exactly with workflow file
Ensure Docker Hub credentials are correct
Issue 5: Image Pull Errors
Problem: Kubernetes cannot pull the Docker image Solution:

# For local development with Minikube
eval $(minikube docker-env)
docker build -t cicd-app:latest .
Conclusion
Congratulations! You have successfully completed Lab 16: Integrating Kubernetes with CI/CD Pipelines. In this comprehensive lab, you have accomplished the following:

Key Achievements: • Built a complete CI/CD pipeline using GitHub Actions that automatically builds, tests, and deploys applications • Integrated Docker containerization with Kubernetes deployment workflows • Implemented automated testing and security auditing in your pipeline • Configured multi-environment deployments with staging and production configurations • Mastered rollback procedures both manual and automated for handling deployment failures • Applied DevOps best practices including proper secret management and environment separation

Technical Skills Developed: • Container orchestration with Kubernetes • Continuous Integration and Continuous Deployment (CI/CD) concepts • Infrastructure as Code using YAML manifests • Automated testing and deployment verification • Rollback strategies and disaster recovery procedures • Multi-environment deployment management

Real-World Applications: This lab simulates real-world enterprise scenarios where development teams need to:

Automatically deploy code changes to production environments
Maintain high availability during deployments
Quickly recover from failed deployments
Manage multiple environments with different configurations
Ensure code quality through automated testing
Why This Matters: Modern software development relies heavily on automation to deliver reliable, scalable applications. The CI/CD pipeline you've built represents industry-standard practices used by companies worldwide to:

Reduce manual errors in deployment processes
Increase deployment frequency and reliability
Enable rapid response to issues through automated rollbacks
Maintain consistent environments across development lifecycle
Support agile development methodologies
Next Steps: To further enhance your skills, consider exploring:

Advanced Kubernetes features like Helm charts and operators
Monitoring and observability tools like Prometheus and Grafana
Security scanning integration in CI/CD pipelines
GitOps workflows with tools like ArgoCD or Flux
Service mesh technologies like Istio for advanced traffic management
You now have the foundational knowledge to implement robust CI/CD pipelines in production environments and are well-prepared for the Kubernetes and Cloud Native Associate (KCNA) certification exam.
