Lab 5: Application Observability and Health Checks
Objectives
By the end of this lab, you will be able to:

• Configure and implement liveness probes to automatically restart unhealthy containers • Set up readiness probes to control when containers receive traffic • Configure startup probes for applications with slow initialization times • Monitor Pod and node resource usage using kubectl top command • Debug failing Pods effectively using kubectl logs and kubectl describe • Understand the relationship between different probe types and their impact on application reliability • Implement comprehensive health checking strategies for Kubernetes applications

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, containers, deployments) • Familiarity with YAML syntax and structure • Knowledge of basic kubectl commands • Understanding of HTTP status codes and basic networking concepts • Experience with command-line interfaces

Ready-to-Use Cloud Machines
Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software. Your lab environment includes:

• A fully configured Kubernetes cluster • kubectl command-line tool • All necessary permissions to create and manage resources • Metrics server enabled for resource monitoring

Lab Tasks
Task 1: Understanding and Implementing Health Probes
Subtask 1.1: Create a Basic Application Pod
First, let's create a simple web application that we can use to demonstrate health checks.

Create a directory for your lab files:
mkdir ~/lab5-observability
cd ~/lab5-observability
Create a basic Pod definition file:
cat > basic-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: basic-web-app
  labels:
    app: web-server
spec:
  containers:
  - name: web-container
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
Deploy the basic Pod:
kubectl apply -f basic-app.yaml
Verify the Pod is running:
kubectl get pods -o wide
Subtask 1.2: Add Liveness Probe
A liveness probe determines if a container is running properly. If the liveness probe fails, Kubernetes will restart the container.

Create an enhanced Pod definition with a liveness probe:
cat > app-with-liveness.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app-with-liveness
  labels:
    app: web-server-liveness
spec:
  containers:
  - name: web-container
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
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
      successThreshold: 1
EOF
Apply the configuration:
kubectl apply -f app-with-liveness.yaml
Monitor the Pod status:
kubectl get pods -w
Press Ctrl+C to stop watching after observing the Pod status.

Check the probe configuration:
kubectl describe pod app-with-liveness
Subtask 1.3: Add Readiness Probe
A readiness probe determines if a container is ready to receive traffic. Unlike liveness probes, failed readiness probes don't restart the container but remove it from service endpoints.

Create a Pod with both liveness and readiness probes:
cat > app-with-readiness.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app-with-readiness
  labels:
    app: web-server-readiness
spec:
  containers:
  - name: web-container
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
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1
EOF
Deploy the Pod:
kubectl apply -f app-with-readiness.yaml
Watch the Pod become ready:
kubectl get pods -w
Notice how the READY column shows the readiness status.

Subtask 1.4: Add Startup Probe
A startup probe is useful for applications that have slow startup times. It disables liveness and readiness probes until the startup probe succeeds.

Create a comprehensive Pod definition with all three probe types:
cat > app-with-all-probes.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: app-with-all-probes
  labels:
    app: web-server-complete
spec:
  containers:
  - name: web-container
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
    startupProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 6
      successThreshold: 1
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1
EOF
Deploy the comprehensive Pod:
kubectl apply -f app-with-all-probes.yaml
Monitor the startup sequence:
kubectl get pods -w
Examine the detailed probe information:
kubectl describe pod app-with-all-probes
Task 2: Monitor Resource Usage with kubectl top
Subtask 2.1: Enable and Verify Metrics Server
The metrics server collects resource usage data from nodes and Pods.

Check if metrics server is running:
kubectl get pods -n kube-system | grep metrics-server
If metrics server is not running, install it:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Wait for metrics server to be ready:
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=60s
Subtask 2.2: Monitor Pod Resource Usage
Create a resource-intensive Pod for monitoring:
cat > resource-intensive-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: resource-intensive-app
  labels:
    app: resource-test
spec:
  containers:
  - name: stress-container
    image: nginx:1.21
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"
        cpu: "200m"
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
EOF
Deploy the resource-intensive Pod:
kubectl apply -f resource-intensive-app.yaml
Wait for metrics to be available (this may take 1-2 minutes):
sleep 60
Monitor Pod resource usage:
kubectl top pods
Monitor specific Pod resource usage:
kubectl top pod resource-intensive-app
Monitor all Pods with labels:
kubectl top pods -l app=resource-test
Subtask 2.3: Monitor Node Resource Usage
View node resource usage:
kubectl top nodes
Get detailed node information:
kubectl describe nodes
Monitor resource usage over time:
watch -n 5 'kubectl top pods && echo "--- Nodes ---" && kubectl top nodes'
Press Ctrl+C to stop the monitoring after observing the output.

Task 3: Debug Failing Pods
Subtask 3.1: Create a Failing Pod for Debugging
Create a Pod that will fail due to incorrect configuration:
cat > failing-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: failing-app
  labels:
    app: debug-test
spec:
  containers:
  - name: failing-container
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
    livenessProbe:
      httpGet:
        path: /nonexistent-path
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2
    readinessProbe:
      httpGet:
        path: /also-nonexistent
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2
EOF
Deploy the failing Pod:
kubectl apply -f failing-app.yaml
Observe the Pod status:
kubectl get pods -w
You should see the Pod failing and restarting. Press Ctrl+C to stop watching.

Subtask 3.2: Debug Using kubectl describe
Get detailed information about the failing Pod:
kubectl describe pod failing-app
Focus on the Events section to understand what's happening:
kubectl describe pod failing-app | grep -A 20 "Events:"
Check the current status:
kubectl get pod failing-app -o yaml | grep -A 10 "status:"
Subtask 3.3: Debug Using kubectl logs
View the container logs:
kubectl logs failing-app
View logs from previous container instances (if the Pod has restarted):
kubectl logs failing-app --previous
Follow logs in real-time:
kubectl logs failing-app -f
Press Ctrl+C to stop following logs.

View logs with timestamps:
kubectl logs failing-app --timestamps
Subtask 3.4: Fix the Failing Pod
Create a corrected version of the Pod:
cat > fixed-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: fixed-app
  labels:
    app: debug-test-fixed
spec:
  containers:
  - name: working-container
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
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
EOF
Deploy the fixed Pod:
kubectl apply -f fixed-app.yaml
Verify the fixed Pod is working:
kubectl get pods
kubectl describe pod fixed-app
Task 4: Advanced Debugging and Monitoring
Subtask 4.1: Create a Deployment for Better Observability
Create a deployment with multiple replicas:
cat > web-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-container
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
EOF
Deploy the deployment:
kubectl apply -f web-deployment.yaml
Monitor the deployment rollout:
kubectl rollout status deployment/web-deployment
Subtask 4.2: Monitor Multiple Pods
View all Pods created by the deployment:
kubectl get pods -l app=web-app
Monitor resource usage across all Pods:
kubectl top pods -l app=web-app
Get logs from all Pods in the deployment:
kubectl logs -l app=web-app --tail=10
Describe all Pods in the deployment:
kubectl describe pods -l app=web-app
Subtask 4.3: Simulate and Debug Issues
Scale the deployment to test resource usage:
kubectl scale deployment web-deployment --replicas=5
Monitor the scaling process:
kubectl get pods -l app=web-app -w
Press Ctrl+C to stop watching.

Check resource usage after scaling:
kubectl top pods -l app=web-app
kubectl top nodes
Create a Pod with resource constraints that will cause issues:
cat > resource-constrained-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: resource-constrained-app
spec:
  containers:
  - name: constrained-container
    image: nginx:1.21
    resources:
      requests:
        memory: "10Mi"
        cpu: "10m"
      limits:
        memory: "20Mi"
        cpu: "20m"
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 1
      failureThreshold: 2
EOF
Deploy and monitor the constrained Pod:
kubectl apply -f resource-constrained-app.yaml
kubectl get pods -w
Debug the constrained Pod:
kubectl describe pod resource-constrained-app
kubectl logs resource-constrained-app
Cleanup
Clean up the resources created during this lab:

kubectl delete pod basic-web-app
kubectl delete pod app-with-liveness
kubectl delete pod app-with-readiness
kubectl delete pod app-with-all-probes
kubectl delete pod resource-intensive-app
kubectl delete pod failing-app
kubectl delete pod fixed-app
kubectl delete pod resource-constrained-app
kubectl delete deployment web-deployment
Troubleshooting Tips
Common Issues and Solutions
Issue: Metrics server not providing data Solution: Wait 1-2 minutes after Pod creation for metrics to become available, or restart metrics server

Issue: Probes failing immediately Solution: Check the probe path, port, and timing configuration. Increase initialDelaySeconds if needed

Issue: Pod stuck in Pending state Solution: Check node resources with kubectl describe nodes and verify resource requests

Issue: Cannot see logs Solution: Ensure the Pod has started and the container is running. Use kubectl get pods to check status

Conclusion
In this lab, you have successfully:

• Implemented comprehensive health checks by configuring liveness, readiness, and startup probes to ensure application reliability and proper traffic routing • Mastered resource monitoring using kubectl top to track CPU and memory usage across Pods and nodes • Developed debugging skills using kubectl logs and kubectl describe to identify and resolve Pod failures • Applied best practices for probe configuration including appropriate timing, thresholds, and probe types

These observability and health check techniques are essential for maintaining robust Kubernetes applications in production environments. Health probes ensure your applications remain available and responsive, while monitoring tools help you optimize resource usage and quickly identify issues. The debugging skills you've learned will be invaluable when troubleshooting real-world application problems.

Understanding these concepts is crucial for the Certified Kubernetes Application Developer (CKAD) certification and for building reliable, observable applications in Kubernetes environments.
