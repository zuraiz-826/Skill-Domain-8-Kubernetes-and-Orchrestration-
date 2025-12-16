Lab 3: Advanced Deployment Strategies
Objectives
By the end of this lab, students will be able to:

• Deploy applications using Kubernetes Deployment resources with proper configuration • Implement blue/green deployment strategies to minimize downtime during updates • Configure and execute canary deployments with traffic splitting capabilities • Monitor deployment rollouts and understand rollback procedures • Apply advanced deployment patterns for production-ready applications • Troubleshoot common deployment issues and implement best practices

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with YAML configuration files • Basic command-line interface experience • Understanding of containerization concepts • Knowledge of kubectl commands for basic operations

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Kubernetes cluster (minikube or kind) • kubectl command-line tool • Docker runtime • Text editor (nano/vim) • All required networking components

Task 1: Deploy a Sample Application Using Deployment Resource
Subtask 1.1: Create the Application Deployment
First, we'll create a simple web application deployment that we can use throughout this lab.

Create a working directory for the lab:
mkdir ~/advanced-deployments
cd ~/advanced-deployments
Create the initial application deployment file:
cat > app-deployment-v1.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app-v1
  labels:
    app: sample-app
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
      version: v1
  template:
    metadata:
      labels:
        app: sample-app
        version: v1
    spec:
      containers:
      - name: web-server
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v1.0"
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-content
        configMap:
          name: app-content-v1
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content-v1
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Sample App v1</title>
        <style>
            body { font-family: Arial; text-align: center; background-color: #e3f2fd; }
            .container { margin-top: 100px; }
            .version { color: #1976d2; font-size: 24px; font-weight: bold; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Sample Application</h1>
            <p class="version">Version 1.0 - Blue Environment</p>
            <p>This is the initial version of our application</p>
        </div>
    </body>
    </html>
EOF
Apply the deployment:
kubectl apply -f app-deployment-v1.yaml
Verify the deployment is running:
kubectl get deployments
kubectl get pods -l app=sample-app
Subtask 1.2: Create a Service to Expose the Application
Create a service configuration:
cat > app-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  labels:
    app: sample-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  selector:
    app: sample-app
EOF
Apply the service:
kubectl apply -f app-service.yaml
Verify the service is created:
kubectl get services
kubectl describe service sample-app-service
Subtask 1.3: Test the Application
Get the cluster IP and test the application:
# If using minikube
minikube ip

# Test the application (replace <MINIKUBE_IP> with actual IP)
curl http://<MINIKUBE_IP>:30080

# Alternative: Port forward for testing
kubectl port-forward service/sample-app-service 8080:80 &
curl http://localhost:8080
Task 2: Implement Blue/Green Deployment Strategy
Subtask 2.1: Create the Green Environment (Version 2)
Blue/Green deployment involves running two identical production environments. We'll create a second version (Green) while keeping the first version (Blue) running.

Create the green environment deployment:
cat > app-deployment-v2.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app-v2
  labels:
    app: sample-app
    version: v2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
      version: v2
  template:
    metadata:
      labels:
        app: sample-app
        version: v2
    spec:
      containers:
      - name: web-server
        image: nginx:1.22
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v2.0"
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-content
        configMap:
          name: app-content-v2
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content-v2
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Sample App v2</title>
        <style>
            body { font-family: Arial; text-align: center; background-color: #e8f5e8; }
            .container { margin-top: 100px; }
            .version { color: #2e7d32; font-size: 24px; font-weight: bold; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Sample Application</h1>
            <p class="version">Version 2.0 - Green Environment</p>
            <p>This is the updated version with new features</p>
            <ul style="text-align: left; display: inline-block;">
                <li>Enhanced performance</li>
                <li>New user interface</li>
                <li>Bug fixes</li>
            </ul>
        </div>
    </body>
    </html>
EOF
Deploy the green environment:
kubectl apply -f app-deployment-v2.yaml
Verify both environments are running:
kubectl get deployments
kubectl get pods -l app=sample-app --show-labels
Subtask 2.2: Test Both Environments Separately
Create separate services for testing each version:
cat > blue-green-services.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: sample-app-blue
  labels:
    app: sample-app
    version: v1
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30081
  selector:
    app: sample-app
    version: v1
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app-green
  labels:
    app: sample-app
    version: v2
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30082
  selector:
    app: sample-app
    version: v2
EOF
Apply the services:
kubectl apply -f blue-green-services.yaml
Test both versions:
# Test Blue version (v1)
curl http://<MINIKUBE_IP>:30081

# Test Green version (v2)
curl http://<MINIKUBE_IP>:30082

# Or use port forwarding
kubectl port-forward service/sample-app-blue 8081:80 &
kubectl port-forward service/sample-app-green 8082:80 &

curl http://localhost:8081
curl http://localhost:8082
Subtask 2.3: Switch Traffic from Blue to Green
Update the main service to point to the green environment:
kubectl patch service sample-app-service -p '{"spec":{"selector":{"app":"sample-app","version":"v2"}}}'
Verify the traffic switch:
kubectl describe service sample-app-service
curl http://<MINIKUBE_IP>:30080
Monitor the switch and verify no downtime occurred:
# Check service endpoints
kubectl get endpoints sample-app-service

# Verify pods are healthy
kubectl get pods -l app=sample-app,version=v2
Subtask 2.4: Implement Rollback Capability
Create a script for quick rollback:
cat > rollback-to-blue.sh << 'EOF'
#!/bin/bash
echo "Rolling back to Blue environment (v1)..."
kubectl patch service sample-app-service -p '{"spec":{"selector":{"app":"sample-app","version":"v1"}}}'
echo "Rollback completed. Verifying..."
kubectl get endpoints sample-app-service
EOF

chmod +x rollback-to-blue.sh
Test the rollback:
./rollback-to-blue.sh
curl http://<MINIKUBE_IP>:30080
Switch back to green for the next task:
kubectl patch service sample-app-service -p '{"spec":{"selector":{"app":"sample-app","version":"v2"}}}'
Task 3: Configure Canary Deployment with Traffic Splitting
Subtask 3.1: Prepare for Canary Deployment
Canary deployment gradually shifts traffic from the old version to the new version, allowing you to monitor the new version's performance with real traffic.

Reset the main service to serve all traffic to v1 (Blue):
kubectl patch service sample-app-service -p '{"spec":{"selector":{"app":"sample-app","version":"v1"}}}'
Scale down v2 deployment temporarily:
kubectl scale deployment sample-app-v2 --replicas=0
Create a new version (v3) for canary testing:
cat > app-deployment-v3.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app-v3
  labels:
    app: sample-app
    version: v3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
      version: v3
  template:
    metadata:
      labels:
        app: sample-app
        version: v3
    spec:
      containers:
      - name: web-server
        image: nginx:1.23
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v3.0"
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-content
        configMap:
          name: app-content-v3
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-content-v3
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Sample App v3</title>
        <style>
            body { font-family: Arial; text-align: center; background-color: #fff3e0; }
            .container { margin-top: 100px; }
            .version { color: #f57c00; font-size: 24px; font-weight: bold; }
            .canary { background-color: #ffcc02; padding: 10px; border-radius: 5px; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Sample Application</h1>
            <p class="version">Version 3.0 - Canary Release</p>
            <div class="canary">
                <p><strong>🐤 CANARY VERSION</strong></p>
                <p>This version includes experimental features</p>
            </div>
            <ul style="text-align: left; display: inline-block;">
                <li>Advanced analytics</li>
                <li>Real-time monitoring</li>
                <li>Enhanced security</li>
                <li>Performance optimizations</li>
            </ul>
        </div>
    </body>
    </html>
EOF
Deploy the canary version:
kubectl apply -f app-deployment-v3.yaml
Subtask 3.2: Implement 10% Traffic Split
Configure the service for canary deployment by removing version selector:
kubectl patch service sample-app-service -p '{"spec":{"selector":{"app":"sample-app"}}}'
Calculate pod ratios for 10% traffic split:

Stable version (v1): 9 replicas (90%)
Canary version (v3): 1 replica (10%)
Scale the deployments accordingly:

# Scale stable version to 9 replicas
kubectl scale deployment sample-app-v1 --replicas=9

# Ensure canary version has 1 replica
kubectl scale deployment sample-app-v3 --replicas=1

# Verify the scaling
kubectl get deployments
Verify the traffic distribution:
# Check all pods
kubectl get pods -l app=sample-app --show-labels

# Test traffic distribution with multiple requests
for i in {1..20}; do
  echo "Request $i:"
  curl -s http://<MINIKUBE_IP>:30080 | grep -o "Version [0-9.]*"
  sleep 1
done
Subtask 3.3: Monitor the Canary Deployment
Create a monitoring script:
cat > monitor-canary.sh << 'EOF'
#!/bin/bash
echo "Monitoring Canary Deployment..."
echo "================================"

# Function to get version distribution
get_version_stats() {
    local total_requests=50
    local v1_count=0
    local v3_count=0
    
    echo "Sending $total_requests requests to measure traffic distribution..."
    
    for i in $(seq 1 $total_requests); do
        response=$(curl -s http://localhost:8080 | grep -o "Version [0-9.]*" | grep -o "[0-9.]*")
        if [[ "$response" == "1.0" ]]; then
            ((v1_count++))
        elif [[ "$response" == "3.0" ]]; then
            ((v3_count++))
        fi
    done
    
    echo "Results:"
    echo "  Version 1.0 (Stable): $v1_count requests ($(( v1_count * 100 / total_requests ))%)"
    echo "  Version 3.0 (Canary): $v3_count requests ($(( v3_count * 100 / total_requests ))%)"
}

# Monitor pod health
echo "Pod Status:"
kubectl get pods -l app=sample-app -o wide

echo ""
echo "Service Endpoints:"
kubectl get endpoints sample-app-service

echo ""
# Set up port forwarding if not already running
kubectl port-forward service/sample-app-service 8080:80 > /dev/null 2>&1 &
PORT_FORWARD_PID=$!
sleep 2

get_version_stats

# Clean up
kill $PORT_FORWARD_PID 2>/dev/null
EOF

chmod +x monitor-canary.sh
Run the monitoring script:
./monitor-canary.sh
Subtask 3.4: Gradually Increase Canary Traffic
Increase canary traffic to 25%:
# Scale to achieve 25% canary traffic
# Stable: 3 replicas (75%), Canary: 1 replica (25%)
kubectl scale deployment sample-app-v1 --replicas=3
kubectl scale deployment sample-app-v3 --replicas=1
Monitor the new distribution:
./monitor-canary.sh
Increase to 50% traffic split:
# Equal distribution: 2 replicas each
kubectl scale deployment sample-app-v1 --replicas=2
kubectl scale deployment sample-app-v3 --replicas=2
Final monitoring:
./monitor-canary.sh
Subtask 3.5: Complete Canary Rollout or Rollback
Create rollout completion script:
cat > complete-canary-rollout.sh << 'EOF'
#!/bin/bash
echo "Completing canary rollout..."
echo "Scaling canary version to full capacity..."

# Scale canary to full capacity
kubectl scale deployment sample-app-v3 --replicas=3

# Scale down old version
kubectl scale deployment sample-app-v1 --replicas=0

echo "Rollout completed!"
kubectl get deployments
EOF

chmod +x complete-canary-rollout.sh
Create rollback script:
cat > rollback-canary.sh << 'EOF'
#!/bin/bash
echo "Rolling back canary deployment..."
echo "Scaling stable version back to full capacity..."

# Scale stable version back
kubectl scale deployment sample-app-v1 --replicas=3

# Scale down canary version
kubectl scale deployment sample-app-v3 --replicas=0

echo "Rollback completed!"
kubectl get deployments
EOF

chmod +x rollback-canary.sh
Choose your path - Complete rollout or rollback:
# To complete the canary rollout:
./complete-canary-rollout.sh

# OR to rollback (if issues were detected):
# ./rollback-canary.sh
Advanced Monitoring and Troubleshooting
Monitor Deployment Health
Create a comprehensive monitoring script:
cat > deployment-health-check.sh << 'EOF'
#!/bin/bash
echo "=== Deployment Health Check ==="
echo ""

echo "1. Deployment Status:"
kubectl get deployments -l app=sample-app

echo ""
echo "2. Pod Status:"
kubectl get pods -l app=sample-app -o wide

echo ""
echo "3. Service Status:"
kubectl get services -l app=sample-app

echo ""
echo "4. Recent Events:"
kubectl get events --sort-by=.metadata.creationTimestamp | tail -10

echo ""
echo "5. Resource Usage:"
kubectl top pods -l app=sample-app 2>/dev/null || echo "Metrics server not available"

echo ""
echo "6. Deployment History:"
kubectl rollout history deployment/sample-app-v1
kubectl rollout history deployment/sample-app-v3
EOF

chmod +x deployment-health-check.sh
Run the health check:
./deployment-health-check.sh
Common Troubleshooting Commands
# Check deployment rollout status
kubectl rollout status deployment/sample-app-v3

# View deployment details
kubectl describe deployment sample-app-v3

# Check pod logs
kubectl logs -l app=sample-app,version=v3

# Check service endpoints
kubectl describe service sample-app-service

# View resource quotas and limits
kubectl describe limitrange
kubectl describe resourcequota
Cleanup
Clean up all resources created in this lab:
cat > cleanup-lab.sh << 'EOF'
#!/bin/bash
echo "Cleaning up lab resources..."

# Delete deployments
kubectl delete deployment sample-app-v1 sample-app-v2 sample-app-v3

# Delete services
kubectl delete service sample-app-service sample-app-blue sample-app-green

# Delete configmaps
kubectl delete configmap app-content-v1 app-content-v2 app-content-v3

# Kill any background processes
pkill -f "kubectl port-forward"

echo "Cleanup completed!"
EOF

chmod +x cleanup-lab.sh
./cleanup-lab.sh
Verify cleanup:
kubectl get all -l app=sample-app
Conclusion
In this comprehensive lab, you have successfully:

• Deployed applications using Kubernetes Deployment resources with proper configuration and scaling • Implemented blue/green deployment strategies that enable zero-downtime deployments by maintaining two identical production environments • Configured canary deployments with precise traffic splitting, allowing you to test new versions with a small percentage of real user traffic • Monitored deployment rollouts using various Kubernetes tools and custom scripts to ensure application health and performance • Practiced rollback procedures to quickly revert to stable versions when issues are detected

Why This Matters:

These advanced deployment strategies are essential for production environments because they:

Minimize downtime during application updates
Reduce risk by allowing gradual rollouts and quick rollbacks
Enable continuous delivery with confidence in deployment processes
Provide better user experience through seamless updates
Support A/B testing and feature flag implementations
Real-World Applications:

E-commerce platforms use blue/green deployments during peak shopping seasons
Financial services employ canary deployments to test critical updates with minimal risk
Social media platforms leverage these strategies to deploy features to millions of users safely
SaaS applications use these patterns to maintain high availability while continuously improving their services
You now have the practical skills to implement these deployment strategies in production environments, making you better prepared for the Certified Kubernetes Application Developer (CKAD) certification and real-world DevOps scenarios.
