Lab 19: Advanced Rolling Updates
Objectives
By the end of this lab, you will be able to:

• Deploy applications using Kubernetes Deployment resources with proper configuration • Implement rolling updates with controlled batch sizes and update strategies • Monitor deployment progress and understand rolling update mechanics • Perform rollbacks to previous versions when issues arise • Analyze deployment history and revision management • Configure deployment parameters for optimal update performance • Troubleshoot common rolling update scenarios

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with kubectl command-line tool • Knowledge of YAML configuration files • Understanding of container images and versioning • Basic Linux command-line skills

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Kubernetes cluster (single-node for simplicity) • kubectl configured and ready to use • Docker runtime for container operations • Text editors (nano, vim) for file editing

Lab Tasks
Task 1: Deploy an Application Using a Deployment Resource
Subtask 1.1: Create the Initial Application Deployment
First, let's create a simple web application deployment that we can later update.

Create a directory for your lab files:
mkdir ~/lab19-rolling-updates
cd ~/lab19-rolling-updates
Create the initial deployment YAML file:
nano nginx-deployment.yaml
Add the following content to create a deployment with NGINX:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 2
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
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
Apply the deployment:
kubectl apply -f nginx-deployment.yaml
Verify the deployment is running:
kubectl get deployments
kubectl get pods -l app=nginx
Subtask 1.2: Create a Service to Expose the Application
Create a service YAML file:
nano nginx-service.yaml
Add the service configuration:
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
  type: ClusterIP
Apply the service:
kubectl apply -f nginx-service.yaml
Verify the service:
kubectl get services
kubectl describe service nginx-service
Subtask 1.3: Test the Initial Deployment
Create a test pod to verify connectivity:
kubectl run test-pod --image=busybox --rm -it --restart=Never -- /bin/sh
Inside the test pod, test the service:
wget -qO- nginx-service
exit
Task 2: Perform a Rolling Update with Controlled Batch Sizes
Subtask 2.1: Monitor Current Deployment Status
Check the current deployment details:
kubectl describe deployment nginx-deployment
View the current replica set:
kubectl get replicasets -l app=nginx
Check the deployment rollout status:
kubectl rollout status deployment/nginx-deployment
Subtask 2.2: Perform a Rolling Update
Update the deployment to use a newer NGINX version:
kubectl set image deployment/nginx-deployment nginx=nginx:1.21
Watch the rolling update in real-time:
kubectl rollout status deployment/nginx-deployment --watch=true
In another terminal window, monitor the pods during the update:
watch kubectl get pods -l app=nginx
Observe the replica sets during the update:
kubectl get replicasets -l app=nginx
Subtask 2.3: Analyze the Rolling Update Process
Check the deployment events:
kubectl describe deployment nginx-deployment
View the deployment history:
kubectl rollout history deployment/nginx-deployment
Get detailed information about a specific revision:
kubectl rollout history deployment/nginx-deployment --revision=2
Subtask 2.4: Perform Another Update with Custom Strategy
Create an updated deployment file with modified rolling update strategy:
nano nginx-deployment-v2.yaml
Add the following content with more conservative update settings:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: "v2"
    spec:
      containers:
      - name: nginx
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
        env:
        - name: VERSION
          value: "v2"
Apply the updated deployment:
kubectl apply -f nginx-deployment-v2.yaml
Monitor the slower rolling update:
kubectl rollout status deployment/nginx-deployment --watch=true
Task 3: Roll Back the Update and Analyze Deployment History
Subtask 3.1: Simulate a Problematic Update
Create a deployment with a problematic image:
kubectl set image deployment/nginx-deployment nginx=nginx:invalid-tag
Watch the failed update:
kubectl rollout status deployment/nginx-deployment --timeout=60s
Check the deployment status:
kubectl get deployments
kubectl get pods -l app=nginx
Subtask 3.2: Perform a Rollback
Check the rollout history:
kubectl rollout history deployment/nginx-deployment
Roll back to the previous version:
kubectl rollout undo deployment/nginx-deployment
Monitor the rollback process:
kubectl rollout status deployment/nginx-deployment
Verify the rollback was successful:
kubectl get pods -l app=nginx
kubectl describe deployment nginx-deployment
Subtask 3.3: Roll Back to a Specific Revision
View the complete rollout history:
kubectl rollout history deployment/nginx-deployment
Roll back to a specific revision (e.g., revision 1):
kubectl rollout undo deployment/nginx-deployment --to-revision=1
Verify the rollback:
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -o wide
Subtask 3.4: Analyze Deployment History and Revisions
Create a script to analyze deployment history:
nano analyze-deployment.sh
Add the following script content:
#!/bin/bash

echo "=== Deployment Status ==="
kubectl get deployment nginx-deployment

echo -e "\n=== Rollout History ==="
kubectl rollout history deployment/nginx-deployment

echo -e "\n=== Current Replica Sets ==="
kubectl get replicasets -l app=nginx

echo -e "\n=== Pod Details ==="
kubectl get pods -l app=nginx -o wide

echo -e "\n=== Deployment Description ==="
kubectl describe deployment nginx-deployment | grep -A 10 "RollingUpdateStrategy"
Make the script executable and run it:
chmod +x analyze-deployment.sh
./analyze-deployment.sh
Subtask 3.5: Advanced Rollback Scenarios
Create a deployment with revision limit:
nano nginx-deployment-limited.yaml
Add configuration with revision history limit:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 6
  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 2
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
Apply the configuration:
kubectl apply -f nginx-deployment-limited.yaml
Perform multiple updates to test revision limits:
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
kubectl rollout status deployment/nginx-deployment

kubectl set image deployment/nginx-deployment nginx=nginx:1.23
kubectl rollout status deployment/nginx-deployment

kubectl set image deployment/nginx-deployment nginx=nginx:1.24
kubectl rollout status deployment/nginx-deployment
Check how many revisions are kept:
kubectl rollout history deployment/nginx-deployment
kubectl get replicasets -l app=nginx
Task 4: Advanced Rolling Update Configurations
Subtask 4.1: Configure Readiness and Liveness Probes
Create a deployment with health checks:
nano nginx-deployment-probes.yaml
Add the following configuration:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
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
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
Apply the deployment with probes:
kubectl apply -f nginx-deployment-probes.yaml
Monitor the deployment with health checks:
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
Subtask 4.2: Test Rolling Update with Probes
Update the image and observe the behavior:
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
Watch the pods transition:
watch kubectl get pods -l app=nginx
Check pod readiness during the update:
kubectl describe pods -l app=nginx | grep -A 5 "Conditions"
Troubleshooting Common Issues
Issue 1: Rolling Update Stuck
If a rolling update appears stuck:

# Check deployment status
kubectl describe deployment nginx-deployment

# Check pod events
kubectl get events --sort-by=.metadata.creationTimestamp

# Force restart if needed
kubectl rollout restart deployment/nginx-deployment
Issue 2: Rollback Not Working
If rollback fails:

# Check rollout history
kubectl rollout history deployment/nginx-deployment

# Manually scale down problematic replica set
kubectl scale replicaset <problematic-rs-name> --replicas=0

# Force rollback
kubectl rollout undo deployment/nginx-deployment --force
Issue 3: Resource Constraints
If pods fail to start due to resources:

# Check node resources
kubectl top nodes

# Check pod resource requests
kubectl describe pods -l app=nginx | grep -A 5 "Requests"

# Adjust resource limits in deployment
kubectl patch deployment nginx-deployment -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","resources":{"requests":{"memory":"32Mi","cpu":"100m"}}}]}}}}'
Cleanup
After completing the lab, clean up the resources:

kubectl delete deployment nginx-deployment
kubectl delete service nginx-service
rm -rf ~/lab19-rolling-updates
Conclusion
In this lab, you have successfully:

• Deployed applications using Kubernetes Deployment resources with proper configuration and resource management • Implemented rolling updates with controlled batch sizes using maxUnavailable and maxSurge parameters • Monitored deployment progress and understood how Kubernetes manages pod transitions during updates • Performed rollbacks to previous versions when issues occurred, including both automatic and manual rollback scenarios • Analyzed deployment history and learned how revision management works in Kubernetes • Configured advanced deployment strategies including health probes and resource constraints

Why This Matters: Rolling updates are crucial for maintaining application availability during deployments in production environments. Understanding how to control update strategies, monitor progress, and perform rollbacks ensures you can deploy applications safely without service interruption. These skills are essential for the CKAD certification and real-world Kubernetes operations, where zero-downtime deployments are often a business requirement.

The techniques you've learned help ensure reliable application updates, minimize risk during deployments, and provide quick recovery options when issues arise. This knowledge forms the foundation for advanced deployment patterns and GitOps workflows used in modern cloud-native applications.
