Lab 2: Introduction to Kubernetes
Objectives
By the end of this lab, students will be able to:

• Install and configure Minikube for local Kubernetes development • Deploy a simple web application to a Kubernetes cluster • Understand core Kubernetes concepts including pods, deployments, and services • Demonstrate Kubernetes scaling capabilities by manually scaling applications • Explore Kubernetes self-healing features through pod failure simulation • Compare Kubernetes orchestration capabilities with Docker Swarm • Navigate the Kubernetes command-line interface (kubectl) effectively

Prerequisites
Before starting this lab, students should have:

• Basic understanding of containerization concepts and Docker • Familiarity with Linux command-line operations • Knowledge of YAML file structure and syntax • Understanding of basic networking concepts (ports, IP addresses) • Experience with text editors (nano, vim, or similar)

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your dedicated environment. No need to build your own virtual machine or install additional software on your local computer.

Your cloud machine comes with: • Ubuntu 20.04 LTS operating system • Docker pre-installed and configured • Internet connectivity for downloading required packages • Administrative privileges for system configuration

Task 1: Install Kubernetes Using Minikube
Subtask 1.1: Update System and Install Dependencies
First, ensure your system is up-to-date and install necessary dependencies.

# Update package repository
sudo apt update && sudo apt upgrade -y

# Install curl and wget for downloading packages
sudo apt install -y curl wget apt-transport-https

# Install VirtualBox (required for Minikube)
sudo apt install -y virtualbox virtualbox-ext-pack
Subtask 1.2: Install kubectl
kubectl is the command-line tool for interacting with Kubernetes clusters.

# Download the latest kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make kubectl executable
chmod +x kubectl

# Move kubectl to system PATH
sudo mv kubectl /usr/local/bin/

# Verify kubectl installation
kubectl version --client
Subtask 1.3: Install Minikube
Minikube creates a local Kubernetes cluster for development and testing purposes.

# Download Minikube binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install Minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify Minikube installation
minikube version
Subtask 1.4: Start Minikube Cluster
# Start Minikube with VirtualBox driver
minikube start --driver=virtualbox --memory=2048 --cpus=2

# Verify cluster status
minikube status

# Check cluster information
kubectl cluster-info

# View cluster nodes
kubectl get nodes
Expected Output: You should see one node in Ready status, indicating your Kubernetes cluster is operational.

Task 2: Deploy a Simple Application in Kubernetes
Subtask 2.1: Create Application Deployment
We'll deploy an nginx web server as our sample application.

# Create a deployment using nginx image
kubectl create deployment nginx-app --image=nginx:latest

# Verify deployment creation
kubectl get deployments

# Check pods created by the deployment
kubectl get pods

# Get detailed information about the deployment
kubectl describe deployment nginx-app
Subtask 2.2: Create Application Manifest Files
Create YAML files for better configuration management.

# Create a directory for Kubernetes manifests
mkdir ~/k8s-lab
cd ~/k8s-lab

# Create deployment manifest file
cat > nginx-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
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
Subtask 2.3: Apply Deployment Configuration
# Apply the deployment configuration
kubectl apply -f nginx-deployment.yaml

# Verify the deployment
kubectl get deployments
kubectl get pods -l app=nginx

# Check pod details
kubectl describe pods -l app=nginx
Subtask 2.4: Expose Application with Service
Create a service to make the application accessible.

# Create service manifest file
cat > nginx-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
EOF

# Apply the service configuration
kubectl apply -f nginx-service.yaml

# Verify service creation
kubectl get services

# Get service details
kubectl describe service nginx-service
Subtask 2.5: Access the Application
# Get Minikube IP address
minikube ip

# Get service URL
minikube service nginx-service --url

# Test application accessibility
curl $(minikube service nginx-service --url)
Expected Output: You should see the default nginx welcome page HTML content.

Task 3: Explore Kubernetes Key Features
Subtask 3.1: Demonstrate Scaling Capabilities
Horizontal scaling allows you to increase or decrease the number of application instances.

# Check current number of replicas
kubectl get deployments nginx-deployment

# Scale up to 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Verify scaling operation
kubectl get pods -l app=nginx

# Watch pods being created in real-time
kubectl get pods -l app=nginx -w
Press Ctrl+C to stop watching.

# Scale down to 2 replicas
kubectl scale deployment nginx-deployment --replicas=2

# Verify scale-down operation
kubectl get pods -l app=nginx

# Check deployment status
kubectl get deployments nginx-deployment
Subtask 3.2: Explore Self-Healing Capabilities
Kubernetes automatically restarts failed containers and replaces unhealthy pods.

# List current pods with their names
kubectl get pods -l app=nginx -o wide

# Delete one pod to simulate failure
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD_NAME

# Immediately check pod status
kubectl get pods -l app=nginx

# Watch Kubernetes create a replacement pod
kubectl get pods -l app=nginx -w
Press Ctrl+C to stop watching.

Key Observation: Notice how Kubernetes immediately creates a new pod to maintain the desired replica count.

Subtask 3.3: Explore Pod Logs and Debugging
# View logs from nginx pods
kubectl logs -l app=nginx

# Get logs from a specific pod
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME

# Execute commands inside a pod
kubectl exec -it $POD_NAME -- /bin/bash

# Inside the pod, check nginx status
nginx -t
exit
Subtask 3.4: Resource Monitoring
# Check resource usage of nodes
kubectl top nodes

# Check resource usage of pods
kubectl top pods

# Get detailed resource information
kubectl describe nodes minikube
Task 4: Compare Kubernetes with Docker Swarm
Subtask 4.1: Create Comparison Analysis
Create a comparison document to understand the differences between orchestration tools.

# Create comparison file
cat > orchestration-comparison.md << 'EOF'
# Kubernetes vs Docker Swarm Comparison

## Architecture
**Kubernetes:**
- Master-worker architecture with multiple components
- etcd for distributed storage
- Complex but highly scalable

**Docker Swarm:**
- Simpler architecture integrated with Docker Engine
- Built-in distributed storage
- Easier to set up but less feature-rich

## Learning Curve
**Kubernetes:**
- Steeper learning curve
- More concepts to understand (pods, deployments, services, etc.)
- Extensive documentation and community support

**Docker Swarm:**
- Gentler learning curve
- Familiar Docker commands
- Limited advanced features

## Scaling Capabilities
**Kubernetes:**
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
- Custom metrics scaling
- Advanced scheduling

**Docker Swarm:**
- Basic service scaling
- Simple replica management
- Limited autoscaling options

## Service Discovery
**Kubernetes:**
- DNS-based service discovery
- Service mesh integration
- Advanced networking policies

**Docker Swarm:**
- Built-in service discovery
- Overlay networks
- Basic load balancing

## Ecosystem
**Kubernetes:**
- Vast ecosystem (Helm, Istio, Prometheus)
- Cloud provider integrations
- CNCF graduated project

**Docker Swarm:**
- Limited third-party integrations
- Docker-centric ecosystem
- Simpler toolchain
EOF

# Display the comparison
cat orchestration-comparison.md
Subtask 4.2: Practical Feature Comparison
# Kubernetes deployment command
echo "Kubernetes Deployment:"
echo "kubectl create deployment app --image=nginx --replicas=3"
echo "kubectl expose deployment app --port=80 --type=NodePort"
echo ""

# Docker Swarm equivalent
echo "Docker Swarm Equivalent:"
echo "docker service create --name app --replicas 3 --publish 80:80 nginx"
echo ""

# Show current Kubernetes resources
echo "Current Kubernetes Resources:"
kubectl get all
Subtask 4.3: Decision Matrix Creation
# Create decision matrix
cat > decision-matrix.md << 'EOF'
# Orchestration Tool Decision Matrix

| Feature | Kubernetes | Docker Swarm | Winner |
|---------|------------|--------------|---------|
| Ease of Setup | Complex | Simple | Docker Swarm |
| Learning Curve | Steep | Gentle | Docker Swarm |
| Scalability | Excellent | Good | Kubernetes |
| Community Support | Extensive | Moderate | Kubernetes |
| Enterprise Features | Rich | Basic | Kubernetes |
| Cloud Integration | Excellent | Limited | Kubernetes |
| Monitoring | Advanced | Basic | Kubernetes |
| Security | Comprehensive | Basic | Kubernetes |

## Recommendation
- **Choose Kubernetes** for: Production environments, complex applications, enterprise needs
- **Choose Docker Swarm** for: Simple applications, quick prototypes, Docker-centric workflows
EOF

cat decision-matrix.md
Task 5: Advanced Kubernetes Operations
Subtask 5.1: ConfigMaps and Secrets
# Create a ConfigMap for application configuration
kubectl create configmap nginx-config --from-literal=server_name=myapp.local

# Create a Secret for sensitive data
kubectl create secret generic nginx-secret --from-literal=username=admin --from-literal=password=secretpass

# View created resources
kubectl get configmaps
kubectl get secrets

# Describe the ConfigMap
kubectl describe configmap nginx-config
Subtask 5.2: Health Checks and Probes
Create an enhanced deployment with health checks.

# Create deployment with health probes
cat > nginx-with-probes.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-probes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-probes
  template:
    metadata:
      labels:
        app: nginx-probes
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
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
EOF

# Apply the configuration
kubectl apply -f nginx-with-probes.yaml

# Monitor pod startup
kubectl get pods -l app=nginx-probes -w
Press Ctrl+C to stop watching.

Subtask 5.3: Rolling Updates
# Update the nginx image version
kubectl set image deployment/nginx-with-probes nginx=nginx:1.22

# Watch the rolling update process
kubectl rollout status deployment/nginx-with-probes

# Check rollout history
kubectl rollout history deployment/nginx-with-probes

# Rollback if needed (optional)
# kubectl rollout undo deployment/nginx-with-probes
Task 6: Cleanup and Resource Management
Subtask 6.1: Clean Up Resources
# Delete deployments
kubectl delete deployment nginx-app
kubectl delete deployment nginx-deployment
kubectl delete deployment nginx-with-probes

# Delete services
kubectl delete service nginx-service

# Delete ConfigMaps and Secrets
kubectl delete configmap nginx-config
kubectl delete secret nginx-secret

# Verify cleanup
kubectl get all
Subtask 6.2: Stop Minikube
# Stop the Minikube cluster
minikube stop

# Check Minikube status
minikube status

# Optional: Delete the cluster completely
# minikube delete
Troubleshooting Common Issues
Issue 1: Minikube Won't Start
Problem: Minikube fails to start with VirtualBox driver.

Solution:

# Check VirtualBox installation
vboxmanage --version

# Try starting with different driver
minikube start --driver=docker

# Check system resources
free -h
Issue 2: Pods Stuck in Pending State
Problem: Pods remain in Pending status.

Solution:

# Check node resources
kubectl describe nodes

# Check pod events
kubectl describe pod <pod-name>

# Check if images can be pulled
kubectl get events --sort-by=.metadata.creationTimestamp
Issue 3: Service Not Accessible
Problem: Cannot access application through service.

Solution:

# Check service endpoints
kubectl get endpoints

# Verify pod labels match service selector
kubectl get pods --show-labels

# Test service connectivity from within cluster
kubectl run test-pod --image=busybox --rm -it -- wget -qO- nginx-service
Conclusion
In this comprehensive lab, you have successfully:

• Installed and configured Minikube to create a local Kubernetes development environment • Deployed applications using both imperative commands and declarative YAML manifests • Explored core Kubernetes concepts including pods, deployments, services, and resource management • Demonstrated scaling capabilities by manually adjusting replica counts and observing automatic pod management • Experienced self-healing features through pod failure simulation and automatic recovery • Compared Kubernetes with Docker Swarm to understand different orchestration approaches and use cases • Implemented advanced features such as health probes, rolling updates, and configuration management

Why This Matters: Kubernetes has become the de facto standard for container orchestration in enterprise environments. The skills you've developed in this lab are directly applicable to:

Cloud-native application development and deployment
DevOps practices including CI/CD pipeline integration
Microservices architecture implementation and management
Preparation for KCNA certification and advanced Kubernetes certifications
Real-world production environments where scalability and reliability are critical
The hands-on experience with Minikube provides a solid foundation for working with managed Kubernetes services like Amazon EKS, Google GKE, or Azure AKS. You now understand the fundamental concepts that make Kubernetes a powerful platform for modern application deployment and management.

Continue practicing these concepts and explore additional Kubernetes features such as Ingress controllers, persistent volumes, and cluster monitoring to further enhance your container orchestration expertise.
