Lab 20: Exploring Cloud Native Application Delivery with GitOps
Objectives
By the end of this lab, you will be able to:

• Understand the core principles and benefits of GitOps methodology • Install and configure ArgoCD as a GitOps tool in a Kubernetes cluster • Create and structure a Git repository for Kubernetes manifests • Deploy applications using GitOps workflows • Observe automatic synchronization between Git repository changes and cluster state • Update applications through Git commits and monitor the deployment process • Troubleshoot common GitOps deployment issues

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with Git version control system • Basic knowledge of YAML syntax • Understanding of container concepts and Docker • Access to command-line interface (CLI) tools

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-installed • Minikube for local Kubernetes cluster • Git client configured and ready to use • Text editor (nano/vim) for file editing

Task 1: Understanding GitOps and Setting Up the Environment
Subtask 1.1: Start Your Kubernetes Cluster
First, let's start our local Kubernetes cluster using Minikube:

# Start Minikube cluster
minikube start --driver=docker --memory=4096 --cpus=2

# Verify cluster is running
kubectl cluster-info

# Check node status
kubectl get nodes
Subtask 1.2: Create Namespace for ArgoCD
Create a dedicated namespace for ArgoCD installation:

# Create argocd namespace
kubectl create namespace argocd

# Verify namespace creation
kubectl get namespaces
Task 2: Installing and Configuring ArgoCD
Subtask 2.1: Install ArgoCD
Install ArgoCD using the official installation manifests:

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Check ArgoCD pods status
kubectl get pods -n argocd
Subtask 2.2: Access ArgoCD UI
Set up port forwarding to access the ArgoCD web interface:

# Port forward ArgoCD server (run this in background)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
Note: Save the password output - you'll need it to log into ArgoCD UI.

Subtask 2.3: Install ArgoCD CLI
Install the ArgoCD command-line interface:

# Download ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Make it executable and move to PATH
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Verify installation
argocd version --client
Subtask 2.4: Login to ArgoCD
Login to ArgoCD using the CLI:

# Login to ArgoCD (use the password from step 2.2)
argocd login localhost:8080 --username admin --password <your-password> --insecure

# Verify login
argocd account get-user-info
Task 3: Creating Git Repository for Kubernetes Manifests
Subtask 3.1: Initialize Local Git Repository
Create a local Git repository to store your Kubernetes manifests:

# Create project directory
mkdir ~/gitops-demo
cd ~/gitops-demo

# Initialize Git repository
git init

# Configure Git user (if not already configured)
git config user.name "GitOps Student"
git config user.email "student@example.com"
Subtask 3.2: Create Application Manifests
Create a sample application with Kubernetes manifests:

# Create application directory structure
mkdir -p apps/sample-app

# Create deployment manifest
cat > apps/sample-app/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
EOF

# Create service manifest
cat > apps/sample-app/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Create namespace manifest
cat > apps/sample-app/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: sample-app
EOF
Subtask 3.3: Commit Initial Manifests
Commit your manifests to the Git repository:

# Add files to Git
git add .

# Commit changes
git commit -m "Initial commit: Add sample application manifests"

# View commit history
git log --oneline
Task 4: Integrating Git Repository with ArgoCD
Subtask 4.1: Create ArgoCD Application
Create an ArgoCD application that monitors your Git repository:

# Create ArgoCD application manifest
cat > argocd-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: file:///home/student/gitops-demo
    targetRevision: HEAD
    path: apps/sample-app
  destination:
    server: https://kubernetes.default.svc
    namespace: sample-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF

# Apply the ArgoCD application
kubectl apply -f argocd-app.yaml
Subtask 4.2: Verify Application Creation
Check that your application was created successfully in ArgoCD:

# List ArgoCD applications
argocd app list

# Get detailed application information
argocd app get sample-app

# Check application status
argocd app status sample-app
Task 5: Deploying Applications Through GitOps
Subtask 5.1: Sync Application
Manually trigger the first synchronization:

# Sync the application
argocd app sync sample-app

# Wait for sync to complete
argocd app wait sample-app --timeout 300
Subtask 5.2: Verify Deployment
Verify that your application has been deployed to the cluster:

# Check if namespace was created
kubectl get namespaces

# Check pods in sample-app namespace
kubectl get pods -n sample-app

# Check services
kubectl get services -n sample-app

# Check deployment status
kubectl get deployments -n sample-app
Subtask 5.3: Test Application Connectivity
Test that your application is running correctly:

# Port forward to test the application
kubectl port-forward -n sample-app svc/sample-app-service 8081:80 &

# Test the application (in a new terminal or after a few seconds)
curl http://localhost:8081

# Stop port forwarding
pkill -f "kubectl port-forward"
Task 6: Updating Applications Through Git Commits
Subtask 6.1: Modify Application Configuration
Update the application by changing the replica count:

# Navigate to repository directory
cd ~/gitops-demo

# Update deployment to use 3 replicas
sed -i 's/replicas: 2/replicas: 3/' apps/sample-app/deployment.yaml

# Verify the change
grep "replicas:" apps/sample-app/deployment.yaml
Subtask 6.2: Update Container Image
Update the nginx image version:

# Update nginx image version
sed -i 's/nginx:1.21/nginx:1.22/' apps/sample-app/deployment.yaml

# Verify the change
grep "image:" apps/sample-app/deployment.yaml
Subtask 6.3: Commit Changes
Commit your changes to trigger GitOps synchronization:

# Add changes to Git
git add apps/sample-app/deployment.yaml

# Commit changes
git commit -m "Update: Increase replicas to 3 and upgrade nginx to 1.22"

# View commit history
git log --oneline -n 3
Task 7: Observing Synchronization Process
Subtask 7.1: Monitor ArgoCD Synchronization
Watch ArgoCD automatically detect and sync the changes:

# Check application status
argocd app get sample-app

# Watch the sync process (press Ctrl+C to stop)
watch -n 2 'argocd app get sample-app | grep -E "(Health|Sync)"'
Subtask 7.2: Verify Changes in Cluster
Confirm that changes have been applied to the cluster:

# Check if replicas increased to 3
kubectl get pods -n sample-app

# Check deployment details
kubectl describe deployment sample-app -n sample-app | grep -E "(Replicas|Image)"

# Verify image version
kubectl get deployment sample-app -n sample-app -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
Subtask 7.3: View Synchronization History
Check the synchronization history in ArgoCD:

# View application history
argocd app history sample-app

# Get detailed sync information
argocd app get sample-app --show-operation
Task 8: Advanced GitOps Operations
Subtask 8.1: Add ConfigMap to Application
Create a ConfigMap for application configuration:

# Create ConfigMap manifest
cat > apps/sample-app/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-app-config
  namespace: sample-app
data:
  app.properties: |
    environment=production
    debug=false
    max_connections=100
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>GitOps Demo App</title>
    </head>
    <body>
        <h1>Welcome to GitOps Demo!</h1>
        <p>This application was deployed using GitOps with ArgoCD.</p>
        <p>Version: 2.0</p>
    </body>
    </html>
EOF
Subtask 8.2: Update Deployment to Use ConfigMap
Modify the deployment to mount the ConfigMap:

# Update deployment to include ConfigMap volume
cat > apps/sample-app/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx:1.22
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: config-volume
        configMap:
          name: sample-app-config
EOF
Subtask 8.3: Commit and Observe Changes
Commit the new changes and observe the synchronization:

# Add all changes
git add .

# Commit changes
git commit -m "Add ConfigMap and update deployment to use custom index.html"

# Monitor the sync process
argocd app sync sample-app

# Wait for sync completion
argocd app wait sample-app
Subtask 8.4: Test Updated Application
Test the updated application with custom content:

# Port forward to test updated application
kubectl port-forward -n sample-app svc/sample-app-service 8082:80 &

# Test the updated application
curl http://localhost:8082

# Clean up port forwarding
pkill -f "kubectl port-forward"
Task 9: Troubleshooting GitOps Deployments
Subtask 9.1: Simulate a Deployment Issue
Create a problematic manifest to see how ArgoCD handles errors:

# Create a deployment with invalid image
cat > apps/sample-app/broken-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  labels:
    app: broken-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: broken-app
        image: nonexistent-image:latest
        ports:
        - containerPort: 80
EOF

# Commit the problematic manifest
git add apps/sample-app/broken-deployment.yaml
git commit -m "Add broken deployment for troubleshooting demo"
Subtask 9.2: Observe Error Handling
Watch how ArgoCD handles the deployment error:

# Check application status
argocd app get sample-app

# View detailed error information
kubectl get events -n sample-app --sort-by='.lastTimestamp'

# Check pod status
kubectl get pods -n sample-app
Subtask 9.3: Fix the Issue
Remove the problematic manifest and restore normal operation:

# Remove the broken deployment file
rm apps/sample-app/broken-deployment.yaml

# Commit the fix
git add -A
git commit -m "Remove broken deployment manifest"

# Sync the application
argocd app sync sample-app
Task 10: Monitoring and Observability
Subtask 10.1: View Application Metrics
Check ArgoCD's built-in monitoring capabilities:

# Get application resource usage
kubectl top pods -n sample-app

# View application logs
kubectl logs -n sample-app -l app=sample-app --tail=20

# Check application health
argocd app get sample-app --show-params
Subtask 10.2: Set Up Application Health Checks
Add health check configuration to your application:

# Update deployment with health checks
cat > apps/sample-app/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx:1.22
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: config-volume
        configMap:
          name: sample-app-config
EOF

# Commit health check updates
git add apps/sample-app/deployment.yaml
git commit -m "Add liveness and readiness probes to deployment"
Cleanup
Clean Up Resources
Remove all resources created during the lab:

# Delete ArgoCD application
argocd app delete sample-app --cascade

# Delete sample-app namespace
kubectl delete namespace sample-app

# Delete ArgoCD namespace (optional)
kubectl delete namespace argocd

# Stop Minikube
minikube stop
Troubleshooting Common Issues
Issue 1: ArgoCD Pods Not Starting
Problem: ArgoCD pods remain in Pending or CrashLoopBackOff state.

Solution:

# Check pod events
kubectl describe pods -n argocd

# Ensure sufficient resources
minikube config set memory 4096
minikube config set cpus 2
minikube delete && minikube start
Issue 2: Application Not Syncing
Problem: ArgoCD application shows "OutOfSync" status but doesn't sync automatically.

Solution:

# Check application configuration
argocd app get sample-app

# Manual sync
argocd app sync sample-app --force

# Check sync policy
kubectl get application sample-app -n argocd -o yaml
Issue 3: Git Repository Access Issues
Problem: ArgoCD cannot access the local Git repository.

Solution:

# Ensure correct repository path
pwd
ls -la ~/gitops-demo

# Check ArgoCD application source configuration
argocd app get sample-app | grep -A 5 "Source:"
Conclusion
Congratulations! You have successfully completed the GitOps lab using ArgoCD. Here's what you accomplished:

Key Achievements: • Installed and configured ArgoCD as a GitOps tool in your Kubernetes cluster • Created a structured Git repository with Kubernetes manifests for application deployment • Implemented GitOps workflows that automatically sync cluster state with Git repository changes • Deployed and updated applications through Git commits, observing the complete synchronization process • Learned troubleshooting techniques for common GitOps deployment issues • Implemented monitoring and health checks for cloud-native applications

Why This Matters: GitOps represents a paradigm shift in how we deploy and manage cloud-native applications. By treating Git as the single source of truth for your infrastructure and applications, you achieve:

• Improved Security: All changes go through Git's audit trail and review process • Better Reliability: Declarative configurations ensure consistent deployments • Enhanced Collaboration: Teams can collaborate using familiar Git workflows • Faster Recovery: Easy rollbacks through Git history • Compliance: Complete audit trail of all infrastructure changes

Real-World Applications: The skills you've learned are directly applicable to: • Enterprise DevOps pipelines using tools like ArgoCD, Flux, or Jenkins X • Multi-environment deployments (development, staging, production) • Microservices architectures with independent service deployments • Infrastructure as Code practices with Kubernetes • Continuous deployment in cloud-native environments

This lab has prepared you for the Kubernetes and Cloud Native Associate (KCNA) certification by providing hands-on experience with GitOps principles and practices that are essential in modern cloud-native development workflows.
