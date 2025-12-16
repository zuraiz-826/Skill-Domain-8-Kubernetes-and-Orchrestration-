Lab 16: Probes and Debugging
Objectives
By the end of this lab, you will be able to:

• Configure readiness and liveness probes for Kubernetes Pods to ensure application health monitoring • Understand the difference between readiness and liveness probes and their use cases • Simulate failing probe scenarios to understand probe behavior • Debug Pod issues using kubectl describe command effectively • Analyze container logs to identify and troubleshoot application problems • Implement best practices for probe configuration in production environments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with YAML syntax and Kubernetes manifest files • Knowledge of basic Linux commands and container concepts • Experience with kubectl command-line tool • Understanding of HTTP status codes and basic networking concepts

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes cluster already set up. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes - everything is ready to use!

Your lab environment includes: • Kubernetes cluster with master and worker nodes • kubectl command-line tool pre-installed and configured • All necessary permissions to create and manage Kubernetes resources

Lab Tasks
Task 1: Understanding and Configuring Readiness Probes
Subtask 1.1: Create a Simple Application with Readiness Probe
First, let's create a simple web application that we can use to demonstrate readiness probes.

Create a new directory for your lab files:
mkdir ~/lab16-probes
cd ~/lab16-probes
Create a simple web application manifest with a readiness probe:
cat > readiness-probe-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-app
  labels:
    app: readiness-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: readiness-app
  template:
    metadata:
      labels:
        app: readiness-app
    spec:
      containers:
      - name: web-server
        image: nginx:1.21
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-service
spec:
  selector:
    app: readiness-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Deploy the application:
kubectl apply -f readiness-probe-app.yaml
Check the deployment status:
kubectl get deployments
kubectl get pods -l app=readiness-app
Subtask 1.2: Observe Readiness Probe Behavior
Watch the pods as they start up:
kubectl get pods -l app=readiness-app -w
Note: Press Ctrl+C to stop watching after you see the pods become Ready.

Describe one of the pods to see probe details:
# Get pod name first
POD_NAME=$(kubectl get pods -l app=readiness-app -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD_NAME
Check the service endpoints:
kubectl get endpoints readiness-service
Key Concept: Notice how pods only appear in the service endpoints after their readiness probe succeeds.

Task 2: Configuring Liveness Probes
Subtask 2.1: Create Application with Both Readiness and Liveness Probes
Create a more comprehensive application with both probe types:
cat > liveness-readiness-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo-app
  labels:
    app: probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probe-demo
  template:
    metadata:
      labels:
        app: probe-demo
    spec:
      containers:
      - name: web-app
        image: nginx:1.21
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 20
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
EOF
Deploy the new application:
kubectl apply -f liveness-readiness-app.yaml
Monitor the pod startup:
kubectl get pods -l app=probe-demo -w
Subtask 2.2: Understanding Probe Parameters
Let's examine what each probe parameter means:

Readiness Probe Parameters: • initialDelaySeconds: 5 - Wait 5 seconds before first probe • periodSeconds: 10 - Check every 10 seconds • timeoutSeconds: 3 - Wait 3 seconds for response • failureThreshold: 3 - Mark as not ready after 3 failures

Liveness Probe Parameters: • initialDelaySeconds: 30 - Wait 30 seconds before first probe • periodSeconds: 20 - Check every 20 seconds • timeoutSeconds: 5 - Wait 5 seconds for response • failureThreshold: 3 - Restart container after 3 failures

Task 3: Simulating Failing Probe Scenarios
Subtask 3.1: Create an Application That Can Simulate Failures
Create a custom application that can simulate probe failures:
cat > failing-probe-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: failing-app
  labels:
    app: failing-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: failing-app
  template:
    metadata:
      labels:
        app: failing-app
    spec:
      containers:
      - name: test-app
        image: busybox:1.35
        command:
        - /bin/sh
        - -c
        - |
          # Create a simple HTTP server that can fail
          mkdir -p /tmp/www
          echo "Application is healthy" > /tmp/www/index.html
          echo "Application is ready" > /tmp/www/ready
          
          # Start simple HTTP server in background
          cd /tmp/www
          while true; do
            echo -e "HTTP/1.1 200 OK\n\n$(cat index.html)" | nc -l -p 8080 -q 1
          done
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
          timeoutSeconds: 3
          failureThreshold: 2
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 2
EOF
Deploy the failing application:
kubectl apply -f failing-probe-app.yaml
Note: This busybox-based example might not work perfectly due to netcat limitations. Let's use a more reliable approach.

Delete the previous deployment and create a better test application:
kubectl delete -f failing-probe-app.yaml
Create a Python-based test application:
cat > python-probe-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-probe-app
  labels:
    app: python-probe-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: python-probe-app
  template:
    metadata:
      labels:
        app: python-probe-app
    spec:
      containers:
      - name: python-app
        image: python:3.9-slim
        command:
        - /bin/sh
        - -c
        - |
          cat > /app.py << 'PYEOF'
          import http.server
          import socketserver
          import os
          import time
          from urllib.parse import urlparse
          
          class HealthHandler(http.server.SimpleHTTPRequestHandler):
              def do_GET(self):
                  if self.path == '/health':
                      if os.path.exists('/tmp/healthy'):
                          self.send_response(200)
                          self.end_headers()
                          self.wfile.write(b'Healthy')
                      else:
                          self.send_response(500)
                          self.end_headers()
                          self.wfile.write(b'Unhealthy')
                  elif self.path == '/ready':
                      if os.path.exists('/tmp/ready'):
                          self.send_response(200)
                          self.end_headers()
                          self.wfile.write(b'Ready')
                      else:
                          self.send_response(503)
                          self.end_headers()
                          self.wfile.write(b'Not Ready')
                  else:
                      self.send_response(200)
                      self.end_headers()
                      self.wfile.write(b'Hello from Python app')
          
          # Initially healthy and ready
          os.system('touch /tmp/healthy /tmp/ready')
          
          with socketserver.TCPServer(("", 8080), HealthHandler) as httpd:
              print("Server running on port 8080")
              httpd.serve_forever()
          PYEOF
          
          python /app.py
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
          timeoutSeconds: 3
          failureThreshold: 2
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 2
EOF
Deploy the Python application:
kubectl apply -f python-probe-app.yaml
Wait for the pod to be ready:
kubectl get pods -l app=python-probe-app -w
Subtask 3.2: Simulate Readiness Probe Failure
Get the pod name:
POD_NAME=$(kubectl get pods -l app=python-probe-app -o jsonpath='{.items[0].metadata.name}')
echo "Pod name: $POD_NAME"
Check current pod status:
kubectl get pod $POD_NAME
Simulate readiness probe failure by removing the ready file:
kubectl exec $POD_NAME -- rm -f /tmp/ready
Watch the pod status change:
kubectl get pod $POD_NAME -w
Note: You should see the pod become "Not Ready" after a few failed readiness checks.

Check the pod events:
kubectl describe pod $POD_NAME
Restore readiness:
kubectl exec $POD_NAME -- touch /tmp/ready
Subtask 3.3: Simulate Liveness Probe Failure
Simulate liveness probe failure:
kubectl exec $POD_NAME -- rm -f /tmp/healthy
Watch for container restart:
kubectl get pod $POD_NAME -w
Note: After the liveness probe fails, Kubernetes will restart the container.

Check the restart count:
kubectl get pod $POD_NAME
Examine the events:
kubectl describe pod $POD_NAME
Task 4: Debugging with kubectl describe
Subtask 4.1: Comprehensive Pod Analysis
Create a problematic application for debugging practice:
cat > debug-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debug-app
  labels:
    app: debug-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: debug-app
  template:
    metadata:
      labels:
        app: debug-app
    spec:
      containers:
      - name: debug-container
        image: nginx:1.21
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /nonexistent
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 2
        readinessProbe:
          httpGet:
            path: /also-nonexistent
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 2
EOF
Deploy the problematic application:
kubectl apply -f debug-app.yaml
Watch the pod behavior:
kubectl get pods -l app=debug-app -w
Subtask 4.2: Using kubectl describe for Debugging
Get detailed information about the problematic pod:
DEBUG_POD=$(kubectl get pods -l app=debug-app -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $DEBUG_POD
Key Areas to Focus On: • Events section - Shows what happened chronologically • Conditions section - Shows current pod conditions • Container Status - Shows restart count and reasons • Probe configurations - Verify probe settings

Analyze the output systematically:
echo "=== POD STATUS ==="
kubectl get pod $DEBUG_POD -o wide

echo "=== POD EVENTS ==="
kubectl get events --field-selector involvedObject.name=$DEBUG_POD --sort-by='.lastTimestamp'

echo "=== POD CONDITIONS ==="
kubectl get pod $DEBUG_POD -o jsonpath='{.status.conditions[*]}' | jq
Note: If jq is not available, use:

kubectl get pod $DEBUG_POD -o yaml | grep -A 20 "conditions:"
Subtask 4.3: Debugging Different Probe Issues
Create a comprehensive debugging checklist by examining our failing pod:
echo "=== DEBUGGING CHECKLIST ==="
echo "1. Pod Phase and Ready Status:"
kubectl get pod $DEBUG_POD -o jsonpath='{.status.phase} - Ready: {.status.conditions[?(@.type=="Ready")].status}'
echo ""

echo "2. Container Restart Count:"
kubectl get pod $DEBUG_POD -o jsonpath='{.status.containerStatuses[0].restartCount}'
echo ""

echo "3. Last Termination Reason:"
kubectl get pod $DEBUG_POD -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
echo ""

echo "4. Current Container State:"
kubectl get pod $DEBUG_POD -o jsonpath='{.status.containerStatuses[0].state}'
echo ""
Task 5: Analyzing Container Logs
Subtask 5.1: Examining Application Logs
Check logs from our Python probe application:
PYTHON_POD=$(kubectl get pods -l app=python-probe-app -o jsonpath='{.items[0].metadata.name}')
kubectl logs $PYTHON_POD
Follow logs in real-time:
kubectl logs -f $PYTHON_POD
Note: Press Ctrl+C to stop following logs.

Check logs from previous container instance (if restarted):
kubectl logs $PYTHON_POD --previous
Subtask 5.2: Log Analysis for Probe Issues
Create an application that logs probe attempts:
cat > logging-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logging-app
  labels:
    app: logging-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logging-app
  template:
    metadata:
      labels:
        app: logging-app
    spec:
      containers:
      - name: logging-container
        image: nginx:1.21
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                echo "Container started at $(date)" >> /var/log/nginx/access.log
                echo "Probes configured - Liveness: 30s interval, Readiness: 10s interval" >> /var/log/nginx/access.log
EOF
Deploy the logging application:
kubectl apply -f logging-app.yaml
Monitor the logs:
LOGGING_POD=$(kubectl get pods -l app=logging-app -o jsonpath='{.items[0].metadata.name}')
kubectl logs $LOGGING_POD -f
Subtask 5.3: Advanced Log Analysis Techniques
Use log filtering to find specific issues:
# Check for error patterns
kubectl logs $LOGGING_POD | grep -i error

# Check for probe-related entries
kubectl logs $LOGGING_POD | grep -i probe

# Get last 20 lines of logs
kubectl logs $LOGGING_POD --tail=20
Analyze logs with timestamps:
kubectl logs $LOGGING_POD --timestamps=true
Compare logs from multiple containers (if applicable):
# List all containers in a pod
kubectl get pod $LOGGING_POD -o jsonpath='{.spec.containers[*].name}'

# Get logs from specific container
kubectl logs $LOGGING_POD -c logging-container
Task 6: Best Practices and Troubleshooting
Subtask 6.1: Implementing Probe Best Practices
Create an optimized application with proper probe configuration:
cat > optimized-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
  labels:
    app: optimized-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: optimized-app
  template:
    metadata:
      labels:
        app: optimized-app
    spec:
      containers:
      - name: web-app
        image: nginx:1.21
        ports:
        - containerPort: 80
        # Startup probe for slow-starting applications
        startupProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 12  # Allow up to 60 seconds for startup
        # Liveness probe with conservative settings
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 0  # Startup probe handles initial delay
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        # Readiness probe with frequent checks
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 0  # Startup probe handles initial delay
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
EOF
Deploy the optimized application:
kubectl apply -f optimized-app.yaml
Observe the startup behavior:
kubectl get pods -l app=optimized-app -w
Subtask 6.2: Common Troubleshooting Scenarios
Scenario 1: Pod stuck in "Not Ready" state
# Check readiness probe configuration
kubectl get pod -l app=optimized-app -o yaml | grep -A 10 readinessProbe

# Check if the application is actually responding
OPTIMIZED_POD=$(kubectl get pods -l app=optimized-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $OPTIMIZED_POD -- curl -I localhost:80
Scenario 2: Container keeps restarting
# Check restart count and reason
kubectl get pod $OPTIMIZED_POD -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl describe pod $OPTIMIZED_POD | grep -A 5 "Last State"
Scenario 3: Probe timeout issues
# Test probe endpoint manually
kubectl exec $OPTIMIZED_POD -- time curl -m 3 localhost:80
Subtask 6.3: Creating a Debugging Toolkit
Create a comprehensive debugging script:
cat > debug-probes.sh << 'EOF'
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: $0 <pod-name>"
    exit 1
fi

POD_NAME=$1

echo "=== DEBUGGING POD: $POD_NAME ==="
echo ""

echo "1. Pod Status:"
kubectl get pod $POD_NAME -o wide
echo ""

echo "2. Pod Conditions:"
kubectl get pod $POD_NAME -o jsonpath='{.status.conditions[*]}' | python3 -m json.tool 2>/dev/null || kubectl get pod $POD_NAME -o yaml | grep -A 20 "conditions:"
echo ""

echo "3. Container Status:"
kubectl get pod $POD_NAME -o jsonpath='{.status.containerStatuses[*]}' | python3 -m json.tool 2>/dev/null || kubectl get pod $POD_NAME -o yaml | grep -A 10 "containerStatuses:"
echo ""

echo "4. Recent Events:"
kubectl get events --field-selector involvedObject.name=$POD_NAME --sort-by='.lastTimestamp' | tail -10
echo ""

echo "5. Probe Configuration:"
kubectl get pod $POD_NAME -o yaml | grep -A 15 "Probe:"
echo ""

echo "6. Recent Logs (last 20 lines):"
kubectl logs $POD_NAME --tail=20
echo ""

echo "=== DEBUG COMPLETE ==="
EOF

chmod +x debug-probes.sh
Test the debugging script:
./debug-probes.sh $OPTIMIZED_POD
Task 7: Cleanup and Verification
Subtask 7.1: Clean Up Resources
Remove all created deployments:
kubectl delete deployment readiness-app probe-demo-app python-probe-app debug-app logging-app optimized-app
Remove services:
kubectl delete service readiness-service
Verify cleanup:
kubectl get deployments
kubectl get pods
kubectl get services
Subtask 7.2: Final Verification Exercise
Create one final test to verify your understanding:
cat > final-test-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: final-test
  labels:
    app: final-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: final-test
  template:
    metadata:
      labels:
        app: final-test
    spec:
      containers:
      - name: test-container
        image: nginx:1.21
        ports:
        - containerPort: 80
        startupProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 6
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
EOF
Deploy and monitor:
kubectl apply -f final-test-app.yaml
kubectl get pods -l app=final-test -w
Use your debugging skills to analyze the pod:
FINAL_POD=$(kubectl get pods -l app=final-test -o jsonpath='{.items[0].metadata.name}')
./debug-probes.sh $FINAL_POD
Clean up the final test:
kubectl delete -f final-test-app.yaml
Conclusion
Congratulations! You have successfully completed Lab 16: Probes and Debugging. In this comprehensive lab, you have accomplished the following:

What You Learned
Probe Configuration Mastery: • Configured readiness probes to control when pods receive traffic • Implemented liveness probes to automatically restart unhealthy containers • Used startup probes to handle slow-starting applications • Understood the relationship between different probe types

Debugging Skills Development: • Used kubectl describe to analyze pod events and conditions • Interpreted probe failure messages and container restart reasons • Analyzed container logs to identify application issues • Created systematic debugging approaches for probe-related problems

Best Practices Implementation: • Learned optimal probe timing and threshold configurations • Understood the importance of proper resource limits with probes • Implemented comprehensive health checking strategies • Created reusable debugging tools and scripts

Why This Matters
Production Readiness: Proper probe configuration is essential for production Kubernetes deployments. Without correct health checks, your applications may receive traffic when they're not ready or continue running when they're unhealthy.

Reliability: Liveness and readiness probes are fundamental to building self-healing applications that can automatically recover from failures and maintain service availability.

Debugging Efficiency: The debugging skills you've developed will help you quickly identify and resolve issues in real-world Kubernetes environments, reducing downtime and improving system reliability.

CKAD Certification: These probe configuration and debugging skills are crucial for the Certified Kubernetes Application Developer exam and will serve you well in professional Kubernetes development.

Next Steps
• Practice configuring probes for different types of applications (databases, microservices, etc.) • Explore advanced probe configurations like TCP and exec probes • Learn about Pod Disruption Budgets and how they interact with readiness probes • Study monitoring and alerting strategies for probe failures in production environments

You now have the knowledge and practical experience to implement robust health checking and debugging strategies in your Kubernetes applications!
