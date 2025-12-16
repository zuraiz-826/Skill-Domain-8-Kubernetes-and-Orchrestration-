Lab 10: Deploying a Stand-Alone Application in Kubernetes
Objectives
By the end of this lab, you will be able to:

• Create and deploy a Kubernetes manifest for a stand-alone application • Configure and deploy a NodePort service to expose applications externally • Monitor application logs using kubectl commands • View and analyze resource metrics for deployed applications • Understand the relationship between Deployments, Pods, and Services in Kubernetes • Troubleshoot common deployment issues in Kubernetes environments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of containerization concepts (Docker) • Familiarity with YAML file structure and syntax • Basic knowledge of Linux command line operations • Understanding of networking concepts (ports, IP addresses) • Previous experience with kubectl commands (recommended but not required)

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-installed • Minikube cluster ready for use • All necessary tools and dependencies configured • Internet access for pulling container images

Task 1: Write and Deploy a Kubernetes Manifest for a Stand-Alone Application
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure your Kubernetes cluster is running properly.

Open a terminal in your lab environment
Check the cluster status:
kubectl cluster-info
Verify that nodes are ready:
kubectl get nodes
You should see output similar to:

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1d    v1.28.3
Subtask 1.2: Create the Application Deployment Manifest
We'll deploy a simple web application using NGINX as our stand-alone application.

Create a new directory for your lab files:
mkdir ~/k8s-lab10
cd ~/k8s-lab10
Create the deployment manifest file:
nano nginx-deployment.yaml
Copy and paste the following YAML content:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-standalone-app
  labels:
    app: nginx-standalone
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-standalone
  template:
    metadata:
      labels:
        app: nginx-standalone
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        env:
        - name: NGINX_PORT
          value: "80"
Save and exit the file (Ctrl+X, then Y, then Enter)
Subtask 1.3: Deploy the Application
Apply the deployment manifest:
kubectl apply -f nginx-deployment.yaml
Verify the deployment was created successfully:
kubectl get deployments
Check the status of the pods:
kubectl get pods -l app=nginx-standalone
Wait for all pods to be in the "Running" status. You can watch the status in real-time:
kubectl get pods -l app=nginx-standalone -w
Press Ctrl+C to stop watching once all pods are running.

Subtask 1.4: Verify Application Details
Get detailed information about the deployment:
kubectl describe deployment nginx-standalone-app
Check the replica set created by the deployment:
kubectl get replicasets -l app=nginx-standalone
Task 2: Expose the Application Externally Using a NodePort Service
Subtask 2.1: Create the NodePort Service Manifest
Create a service manifest file:
nano nginx-service.yaml
Add the following YAML content:
apiVersion: v1
kind: Service
metadata:
  name: nginx-standalone-service
  labels:
    app: nginx-standalone
spec:
  type: NodePort
  selector:
    app: nginx-standalone
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
    name: http
Save and exit the file
Subtask 2.2: Deploy the Service
Apply the service manifest:
kubectl apply -f nginx-service.yaml
Verify the service was created:
kubectl get services
Get detailed information about the service:
kubectl describe service nginx-standalone-service
Subtask 2.3: Test External Access
Get the cluster IP address:
minikube ip
Test the application accessibility using curl:
curl http://$(minikube ip):30080
You should see the default NGINX welcome page HTML content.

Alternatively, you can access the application through your browser. Get the full URL:
echo "Access your application at: http://$(minikube ip):30080"
Subtask 2.4: Verify Service Endpoints
Check the service endpoints to ensure they're pointing to your pods:
kubectl get endpoints nginx-standalone-service
Compare the endpoint IPs with your pod IPs:
kubectl get pods -l app=nginx-standalone -o wide
Task 3: Monitor Application Logs and Resource Metrics
Subtask 3.1: Monitor Application Logs
View logs from all pods in the deployment:
kubectl logs -l app=nginx-standalone
Follow logs in real-time from a specific pod:
# First, get a pod name
kubectl get pods -l app=nginx-standalone

# Then follow logs (replace POD_NAME with actual pod name)
kubectl logs -f POD_NAME
Generate some traffic to create log entries:
# In a new terminal, run this command multiple times
curl http://$(minikube ip):30080
View logs from the last 10 minutes:
kubectl logs -l app=nginx-standalone --since=10m
Subtask 3.2: Monitor Resource Usage
Check resource usage for your pods:
kubectl top pods -l app=nginx-standalone
Monitor resource usage for the entire cluster:
kubectl top nodes
Get detailed resource information for a specific pod:
# Replace POD_NAME with an actual pod name
kubectl describe pod POD_NAME
Subtask 3.3: Monitor Application Health
Check the readiness and liveness of your pods:
kubectl get pods -l app=nginx-standalone -o wide
View events related to your deployment:
kubectl get events --field-selector involvedObject.name=nginx-standalone-app
Monitor the deployment status:
kubectl rollout status deployment/nginx-standalone-app
Subtask 3.4: Create a Custom HTML Page
Let's customize our application to make monitoring more interesting.

Create a custom HTML file:
cat > custom-index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Kubernetes Lab 10 - Stand-Alone App</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background-color: #f0f8ff; }
        .container { max-width: 800px; margin: 0 auto; padding: 20px; background: white; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        h1 { color: #2c3e50; text-align: center; }
        .info { background: #e8f4fd; padding: 15px; border-radius: 5px; margin: 20px 0; }
        .success { color: #27ae60; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Kubernetes Stand-Alone Application</h1>
        <div class="info">
            <h3>Lab 10 Deployment Successful!</h3>
            <p><span class="success">✓</span> Application deployed using Kubernetes Deployment</p>
            <p><span class="success">✓</span> Service exposed via NodePort (30080)</p>
            <p><span class="success">✓</span> Multiple replicas running for high availability</p>
            <p><span class="success">✓</span> Resource limits and requests configured</p>
        </div>
        <p><strong>Hostname:</strong> <span id="hostname">Loading...</span></p>
        <p><strong>Timestamp:</strong> <span id="timestamp"></span></p>
    </div>
    <script>
        document.getElementById('hostname') = window.location.hostname;
        document.getElementById('timestamp').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
EOF
Create a ConfigMap with the custom HTML:
kubectl create configmap nginx-custom-html --from-file=index.html=custom-index.html
Update the deployment to use the custom HTML:
cat > nginx-deployment-updated.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-standalone-app
  labels:
    app: nginx-standalone
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-standalone
  template:
    metadata:
      labels:
        app: nginx-standalone
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
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
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: nginx-custom-html
EOF
Apply the updated deployment:
kubectl apply -f nginx-deployment-updated.yaml
Wait for the rollout to complete:
kubectl rollout status deployment/nginx-standalone-app
Test the updated application:
curl http://$(minikube ip):30080
Troubleshooting Common Issues
Issue 1: Pods Not Starting
If your pods are not starting, check the following:

# Check pod status and events
kubectl describe pods -l app=nginx-standalone

# Check if the image can be pulled
kubectl get events --sort-by=.metadata.creationTimestamp
Issue 2: Service Not Accessible
If you can't access the service externally:

# Verify service configuration
kubectl get svc nginx-standalone-service -o yaml

# Check if minikube tunnel is needed (for some environments)
minikube service nginx-standalone-service --url
Issue 3: Resource Issues
If pods are pending due to resource constraints:

# Check node resources
kubectl describe nodes

# Check resource requests vs available
kubectl top nodes
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Delete the service
kubectl delete service nginx-standalone-service

# Delete the deployment
kubectl delete deployment nginx-standalone-app

# Delete the ConfigMap
kubectl delete configmap nginx-custom-html

# Verify cleanup
kubectl get all -l app=nginx-standalone
Conclusion
Congratulations! You have successfully completed Lab 10: Deploying a Stand-Alone Application in Kubernetes.

What You Accomplished
In this lab, you have:

• Created and deployed a Kubernetes Deployment - You learned how to write YAML manifests to define application deployments with multiple replicas, resource limits, and container specifications

• Exposed applications externally - You successfully configured a NodePort service to make your application accessible from outside the Kubernetes cluster

• Monitored application health and performance - You gained hands-on experience with kubectl commands for viewing logs, checking resource usage, and monitoring application status

• Implemented best practices - You applied resource requests and limits, used labels for organization, and configured proper service selectors

Why This Matters
Understanding how to deploy stand-alone applications in Kubernetes is fundamental for:

• Production deployments - These skills form the foundation for deploying real-world applications in Kubernetes environments

• Cloud-native development - Modern applications increasingly rely on container orchestration platforms like Kubernetes

• Career advancement - These skills are essential for roles in DevOps, Site Reliability Engineering, and Cloud Architecture

• KCNA certification preparation - This lab directly supports your preparation for the Kubernetes and Cloud Native Associate certification

The concepts you've learned here - Deployments, Services, resource management, and monitoring - are building blocks for more advanced Kubernetes topics like StatefulSets, Ingress controllers, and service meshes. You now have practical experience with the core workflow of deploying and managing applications in Kubernetes, which will serve as a solid foundation for your continued learning in cloud-native technologies
