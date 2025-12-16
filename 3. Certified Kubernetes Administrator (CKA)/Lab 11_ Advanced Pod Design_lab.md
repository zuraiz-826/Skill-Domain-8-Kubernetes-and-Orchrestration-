Lab 11: Advanced Pod Design
Objectives
By the end of this lab, you will be able to:

• Understand and implement the sidecar pattern in Kubernetes Pods • Configure and deploy the ambassador pattern for container communication • Create multi-container Pods with shared resources and networking • Configure inter-container communication using shared volumes and localhost networking • Test and validate Pod behavior using kubectl commands and container logs • Troubleshoot common issues in multi-container Pod deployments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, containers, kubectl) • Familiarity with YAML configuration files • Knowledge of Linux command-line operations • Understanding of Docker containers and container networking • Completed previous Kubernetes labs or equivalent experience

Lab Environment Setup
Good News! Al Nafi provides ready-to-use Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install additional software.

Your lab environment includes: • Kubernetes cluster (minikube or similar) • kubectl command-line tool • Text editor (nano/vim) • All required container images

Task 1: Understanding Multi-Container Pod Patterns
Subtask 1.1: Learn About Container Patterns
Multi-container Pods in Kubernetes follow specific design patterns:

Sidecar Pattern: A helper container that extends the functionality of the main container • Example: Log collection, monitoring, or data synchronization • Containers share the same network and storage

Ambassador Pattern: A proxy container that handles external communications • Example: Database connections, API gateways, or load balancing • Simplifies networking for the main application container

Subtask 1.2: Verify Your Environment
First, let's ensure your Kubernetes environment is ready:

# Check if kubectl is working
kubectl version --client

# Verify cluster connection
kubectl cluster-info

# Check available nodes
kubectl get nodes
Expected output should show your cluster information and at least one ready node.

Task 2: Create a Multi-Container Pod with Sidecar Pattern
Subtask 2.1: Design the Sidecar Pod
We'll create a web application with a logging sidecar container that collects and processes logs.

Create a new directory for your lab files:

mkdir ~/advanced-pod-lab
cd ~/advanced-pod-lab
Subtask 2.2: Create the Sidecar Pod Configuration
Create a YAML file for the sidecar pattern Pod:

nano sidecar-pod.yaml
Add the following configuration:

apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
  labels:
    app: web-with-sidecar
spec:
  containers:
  # Main application container
  - name: web-app
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
    - name: web-content
      mountPath: /usr/share/nginx/html
  
  # Sidecar container for log processing
  - name: log-processor
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      while true; do
        echo "$(date): Processing logs..." >> /var/log/shared/processor.log
        if [ -f /var/log/nginx/access.log ]; then
          tail -n 5 /var/log/nginx/access.log >> /var/log/shared/processed-access.log
        fi
        sleep 10
      done
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
    - name: processed-logs
      mountPath: /var/log/shared
  
  # Init container to set up web content
  initContainers:
  - name: content-setup
    image: busybox:1.35
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo '<html><body><h1>Sidecar Pattern Demo</h1><p>This page is served by nginx with a log-processing sidecar!</p></body></html>' > /web-content/index.html
    volumeMounts:
    - name: web-content
      mountPath: /web-content
  
  volumes:
  - name: shared-logs
    emptyDir: {}
  - name: processed-logs
    emptyDir: {}
  - name: web-content
    emptyDir: {}
Subtask 2.3: Deploy the Sidecar Pod
Deploy the Pod to your cluster:

# Apply the configuration
kubectl apply -f sidecar-pod.yaml

# Verify the Pod is running
kubectl get pods -w
Wait until the Pod status shows Running for all containers.

Subtask 2.4: Test Inter-Container Communication
Test the shared volume communication between containers:

# Check the main web application
kubectl port-forward sidecar-pod 8080:80 &

# Test the web service (in a new terminal or stop port-forward with Ctrl+C first)
curl http://localhost:8080

# Check logs from both containers
kubectl logs sidecar-pod -c web-app
kubectl logs sidecar-pod -c log-processor

# Execute commands in the sidecar container
kubectl exec -it sidecar-pod -c log-processor -- ls -la /var/log/shared/
kubectl exec -it sidecar-pod -c log-processor -- cat /var/log/shared/processor.log
Task 3: Implement Ambassador Pattern
Subtask 3.1: Create Ambassador Pod Configuration
The ambassador pattern uses a proxy container to handle external communications. Create the configuration:

nano ambassador-pod.yaml
Add the following configuration:

apiVersion: v1
kind: Pod
metadata:
  name: ambassador-pod
  labels:
    app: app-with-ambassador
spec:
  containers:
  # Main application container
  - name: main-app
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      while true; do
        echo "$(date): Main app making request to database via ambassador..."
        # Connect to ambassador on localhost
        nc -z localhost 3306 && echo "Database connection successful via ambassador" || echo "Database connection failed"
        sleep 15
      done
  
  # Ambassador container (database proxy)
  - name: db-ambassador
    image: alpine/socat:1.7.4.3
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "Starting database ambassador proxy..."
      # Proxy connections from localhost:3306 to external database
      socat TCP-LISTEN:3306,fork,reuseaddr TCP:mysql-service.default.svc.cluster.local:3306
    ports:
    - containerPort: 3306
      name: mysql-proxy
  
  # Configuration container
  - name: config-provider
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "database_host=localhost" > /shared-config/app.conf
      echo "database_port=3306" >> /shared-config/app.conf
      echo "proxy_enabled=true" >> /shared-config/app.conf
      echo "Configuration written. Sleeping..."
      sleep infinity
    volumeMounts:
    - name: shared-config
      mountPath: /shared-config
  
  volumes:
  - name: shared-config
    emptyDir: {}
Subtask 3.2: Create a Mock Database Service
Since we need a database for the ambassador to connect to, let's create a mock MySQL service:

nano mock-database.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mock-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mock-mysql
  template:
    metadata:
      labels:
        app: mock-mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword"
        - name: MYSQL_DATABASE
          value: "testdb"
        ports:
        - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mock-mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
Subtask 3.3: Deploy Ambassador Pattern
Deploy the database and ambassador Pod:

# Deploy the mock database first
kubectl apply -f mock-database.yaml

# Wait for database to be ready
kubectl wait --for=condition=ready pod -l app=mock-mysql --timeout=120s

# Deploy the ambassador pod
kubectl apply -f ambassador-pod.yaml

# Check the status
kubectl get pods
Subtask 3.4: Test Ambassador Communication
Test the ambassador pattern functionality:

# Check logs from all containers
kubectl logs ambassador-pod -c main-app
kubectl logs ambassador-pod -c db-ambassador
kubectl logs ambassador-pod -c config-provider

# Verify the configuration sharing
kubectl exec -it ambassador-pod -c main-app -- cat /shared-config/app.conf 2>/dev/null || echo "Config not mounted in main-app"

# Test network connectivity within the pod
kubectl exec -it ambassador-pod -c main-app -- netstat -tlnp 2>/dev/null || kubectl exec -it ambassador-pod -c main-app -- ss -tlnp
Task 4: Advanced Inter-Container Communication
Subtask 4.1: Create a Complex Multi-Container Pod
Let's create a more sophisticated example that combines both patterns:

nano advanced-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: advanced-pod
  labels:
    app: advanced-multi-container
spec:
  containers:
  # Main web application
  - name: web-server
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: web-content
      mountPath: /usr/share/nginx/html
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
    - name: shared-logs
      mountPath: /var/log/nginx
  
  # Sidecar: Log aggregator
  - name: log-aggregator
    image: fluent/fluent-bit:2.0
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
      readOnly: true
    - name: fluentbit-config
      mountPath: /fluent-bit/etc
  
  # Ambassador: Metrics proxy
  - name: metrics-proxy
    image: prom/prometheus:v2.40.0
    ports:
    - containerPort: 9090
    volumeMounts:
    - name: prometheus-config
      mountPath: /etc/prometheus
    command:
    - /bin/prometheus
    - --config.file=/etc/prometheus/prometheus.yml
    - --storage.tsdb.path=/prometheus
    - --web.console.libraries=/etc/prometheus/console_libraries
    - --web.console.templates=/etc/prometheus/consoles
  
  # Sidecar: Configuration manager
  - name: config-manager
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      # Create nginx configuration
      cat > /nginx-config/default.conf << 'EOF'
      server {
          listen 80;
          server_name localhost;
          
          location / {
              root /usr/share/nginx/html;
              index index.html;
          }
          
          location /metrics {
              proxy_pass http://localhost:9090/metrics;
          }
          
          access_log /var/log/nginx/access.log;
          error_log /var/log/nginx/error.log;
      }
      EOF
      
      # Create Fluent Bit configuration
      cat > /fluentbit-config/fluent-bit.conf << 'EOF'
      [SERVICE]
          Flush         1
          Log_Level     info
          Daemon        off
      
      [INPUT]
          Name              tail
          Path              /var/log/nginx/access.log
          Parser            nginx
          Tag               nginx.access
      
      [OUTPUT]
          Name              stdout
          Match             *
      EOF
      
      # Create Prometheus configuration
      cat > /prometheus-config/prometheus.yml << 'EOF'
      global:
        scrape_interval: 15s
      
      scrape_configs:
        - job_name: 'nginx'
          static_configs:
            - targets: ['localhost:80']
      EOF
      
      # Create web content
      cat > /web-content/index.html << 'EOF'
      <!DOCTYPE html>
      <html>
      <head>
          <title>Advanced Pod Demo</title>
      </head>
      <body>
          <h1>Advanced Multi-Container Pod</h1>
          <p>This demonstrates:</p>
          <ul>
              <li>Sidecar pattern with log aggregation</li>
              <li>Ambassador pattern with metrics proxy</li>
              <li>Inter-container communication</li>
              <li>Shared volumes and networking</li>
          </ul>
          <p><a href="/metrics">View Metrics</a></p>
      </body>
      </html>
      EOF
      
      echo "Configuration setup complete. Monitoring for changes..."
      sleep infinity
    volumeMounts:
    - name: web-content
      mountPath: /web-content
    - name: nginx-config
      mountPath: /nginx-config
    - name: fluentbit-config
      mountPath: /fluentbit-config
    - name: prometheus-config
      mountPath: /prometheus-config
  
  volumes:
  - name: web-content
    emptyDir: {}
  - name: nginx-config
    emptyDir: {}
  - name: fluentbit-config
    emptyDir: {}
  - name: prometheus-config
    emptyDir: {}
  - name: shared-logs
    emptyDir: {}
Subtask 4.2: Deploy and Test Advanced Pod
Deploy the advanced Pod:

# Apply the configuration
kubectl apply -f advanced-pod.yaml

# Wait for the pod to be ready
kubectl wait --for=condition=ready pod/advanced-pod --timeout=120s

# Check pod status
kubectl get pod advanced-pod
kubectl describe pod advanced-pod
Subtask 4.3: Validate Inter-Container Communication
Test all aspects of the advanced Pod:

# Test web server
kubectl port-forward advanced-pod 8080:80 &
curl http://localhost:8080

# Test metrics endpoint (stop previous port-forward first)
pkill -f "kubectl port-forward"
kubectl port-forward advanced-pod 9090:9090 &
curl http://localhost:9090/metrics

# Check logs from all containers
echo "=== Web Server Logs ==="
kubectl logs advanced-pod -c web-server

echo "=== Log Aggregator Logs ==="
kubectl logs advanced-pod -c log-aggregator

echo "=== Metrics Proxy Logs ==="
kubectl logs advanced-pod -c metrics-proxy

echo "=== Config Manager Logs ==="
kubectl logs advanced-pod -c config-manager

# Verify shared volumes
kubectl exec -it advanced-pod -c web-server -- ls -la /usr/share/nginx/html/
kubectl exec -it advanced-pod -c web-server -- ls -la /etc/nginx/conf.d/
Task 5: Testing and Validation
Subtask 5.1: Comprehensive Pod Testing
Create a test script to validate all Pod functionality:

nano test-pods.sh
chmod +x test-pods.sh
#!/bin/bash

echo "=== Advanced Pod Design Lab - Validation Script ==="
echo

# Function to check pod status
check_pod_status() {
    local pod_name=$1
    echo "Checking status of pod: $pod_name"
    kubectl get pod $pod_name -o wide
    echo
}

# Function to test connectivity
test_connectivity() {
    local pod_name=$1
    local container_name=$2
    local test_command=$3
    
    echo "Testing connectivity in $pod_name/$container_name"
    kubectl exec -it $pod_name -c $container_name -- $test_command
    echo
}

# Test all pods
echo "1. Checking Pod Status"
echo "======================"
check_pod_status "sidecar-pod"
check_pod_status "ambassador-pod"
check_pod_status "advanced-pod"

echo "2. Testing Sidecar Pattern"
echo "=========================="
kubectl exec sidecar-pod -c log-processor -- ls -la /var/log/shared/ 2>/dev/null || echo "Sidecar volumes not ready yet"
kubectl logs sidecar-pod -c log-processor --tail=5

echo "3. Testing Ambassador Pattern"
echo "============================="
kubectl logs ambassador-pod -c main-app --tail=5
kubectl logs ambassador-pod -c db-ambassador --tail=5

echo "4. Testing Advanced Pod Communication"
echo "===================================="
kubectl exec advanced-pod -c config-manager -- ls -la /web-content/ 2>/dev/null || echo "Advanced pod not ready yet"

echo "5. Network Connectivity Tests"
echo "============================="
# Test localhost communication within pods
kubectl exec advanced-pod -c web-server -- curl -s http://localhost:9090/metrics | head -5 2>/dev/null || echo "Metrics endpoint not ready"

echo
echo "=== Validation Complete ==="
Run the validation script:

./test-pods.sh
Subtask 5.2: Performance and Resource Monitoring
Monitor resource usage of your multi-container Pods:

# Check resource usage
kubectl top pods

# Get detailed resource information
kubectl describe pod sidecar-pod | grep -A 10 "Containers:"
kubectl describe pod ambassador-pod | grep -A 10 "Containers:"
kubectl describe pod advanced-pod | grep -A 10 "Containers:"

# Check events for any issues
kubectl get events --sort-by=.metadata.creationTimestamp
Task 6: Troubleshooting Common Issues
Subtask 6.1: Common Problems and Solutions
Problem 1: Pod stuck in Pending state

# Check pod events
kubectl describe pod <pod-name>

# Check node resources
kubectl describe nodes
Problem 2: Container communication failures

# Verify shared volumes
kubectl exec -it <pod-name> -c <container-name> -- ls -la /shared/path/

# Test localhost connectivity
kubectl exec -it <pod-name> -c <container-name> -- netstat -tlnp
Problem 3: Init container failures

# Check init container logs
kubectl logs <pod-name> -c <init-container-name>

# Describe pod for init container status
kubectl describe pod <pod-name>
Subtask 6.2: Debug Your Pods
Run debugging commands on your Pods:

# Debug sidecar pod
kubectl exec -it sidecar-pod -c web-app -- ps aux
kubectl exec -it sidecar-pod -c log-processor -- ps aux

# Debug ambassador pod
kubectl exec -it ambassador-pod -c main-app -- nslookup mysql-service.default.svc.cluster.local
kubectl exec -it ambassador-pod -c db-ambassador -- ps aux

# Debug advanced pod
kubectl exec -it advanced-pod -c web-server -- nginx -t
kubectl exec -it advanced-pod -c config-manager -- ls -la /web-content/
Task 7: Cleanup and Resource Management
Subtask 7.1: Clean Up Lab Resources
Remove all created resources:

# Delete pods
kubectl delete pod sidecar-pod
kubectl delete pod ambassador-pod
kubectl delete pod advanced-pod

# Delete database deployment and service
kubectl delete -f mock-database.yaml

# Verify cleanup
kubectl get pods
kubectl get services
kubectl get deployments

# Clean up local files
cd ~
rm -rf advanced-pod-lab
Subtask 7.2: Verify Complete Cleanup
# Final verification
kubectl get all
echo "Lab cleanup complete!"
Conclusion
Congratulations! You have successfully completed the Advanced Pod Design lab. Here's what you accomplished:

Key Achievements: • Mastered Multi-Container Patterns: You implemented both sidecar and ambassador patterns, understanding how they solve different architectural challenges • Configured Inter-Container Communication: You set up shared volumes, localhost networking, and container coordination • Built Complex Pod Architectures: You created sophisticated multi-container Pods with multiple communication patterns • Validated Pod Behavior: You tested and debugged Pod functionality using various kubectl commands and techniques

Why This Matters: • Real-World Applications: These patterns are essential for microservices architectures, logging systems, and service mesh implementations • CKAD Certification: Multi-container Pod design is a key topic in the Certified Kubernetes Application Developer exam • Production Readiness: Understanding these patterns prepares you for designing resilient, scalable applications in production Kubernetes environments • Career Development: These skills are highly valued in DevOps, Platform Engineering, and Cloud Native development roles

Next Steps: • Practice creating your own multi-container Pod designs • Explore Kubernetes operators that use these patterns • Study service mesh technologies like Istio that implement ambassador patterns • Learn about monitoring and observability patterns in Kubernetes

You now have the knowledge and hands-on experience to design and implement advanced Pod architectures in Kubernetes environments!
