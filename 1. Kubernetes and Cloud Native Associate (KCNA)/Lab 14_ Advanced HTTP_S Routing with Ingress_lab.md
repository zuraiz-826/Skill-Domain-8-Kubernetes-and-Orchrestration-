Lab 14: Advanced HTTP/S Routing with Ingress
Objectives
By the end of this lab, you will be able to:

• Deploy multiple applications in a Kubernetes cluster • Configure Ingress controllers for HTTP/S traffic management • Implement path-based routing using Ingress resources • Secure applications with TLS certificates and SSL termination • Verify and troubleshoot Ingress routing rules • Understand the relationship between Services, Ingress, and external traffic

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with YAML configuration files • Knowledge of HTTP/HTTPS protocols and routing concepts • Basic command-line interface experience • Understanding of DNS and domain name concepts

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-installed • Minikube cluster ready for use • NGINX Ingress Controller available for installation • All necessary tools and dependencies

Lab Tasks
Task 1: Environment Setup and Preparation
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure your Kubernetes cluster is running properly.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Check if minikube is running
minikube status
If minikube is not running, start it:

# Start minikube cluster
minikube start --driver=docker

# Enable ingress addon
minikube addons enable ingress
Subtask 1.2: Verify Ingress Controller Installation
Check if the NGINX Ingress Controller is installed and running:

# Check ingress controller pods
kubectl get pods -n ingress-nginx

# Verify ingress controller service
kubectl get svc -n ingress-nginx
If the ingress controller is not installed, enable it:

# Enable ingress addon for minikube
minikube addons enable ingress

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
Task 2: Deploy Two Sample Applications
Subtask 2.1: Create Application Namespaces
Create separate namespaces for better organization:

# Create namespace for applications
kubectl create namespace web-apps
Subtask 2.2: Deploy First Application (App1)
Create the first application deployment and service:

# Create app1-deployment.yaml
cat > app1-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  namespace: web-apps
  labels:
    app: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: app1-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-html
  namespace: web-apps
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Application 1</title>
        <style>
            body { font-family: Arial, sans-serif; background-color: #e3f2fd; text-align: center; padding: 50px; }
            h1 { color: #1976d2; }
        </style>
    </head>
    <body>
        <h1>Welcome to Application 1</h1>
        <p>This is the first application served via Ingress path-based routing.</p>
        <p>Path: /app1</p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
  namespace: web-apps
spec:
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Apply the configuration
kubectl apply -f app1-deployment.yaml
Subtask 2.3: Deploy Second Application (App2)
Create the second application deployment and service:

# Create app2-deployment.yaml
cat > app2-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-deployment
  namespace: web-apps
  labels:
    app: app2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: app2-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app2-html
  namespace: web-apps
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Application 2</title>
        <style>
            body { font-family: Arial, sans-serif; background-color: #f3e5f5; text-align: center; padding: 50px; }
            h1 { color: #7b1fa2; }
        </style>
    </head>
    <body>
        <h1>Welcome to Application 2</h1>
        <p>This is the second application served via Ingress path-based routing.</p>
        <p>Path: /app2</p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: app2-service
  namespace: web-apps
spec:
  selector:
    app: app2
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Apply the configuration
kubectl apply -f app2-deployment.yaml
Subtask 2.4: Verify Application Deployments
Check that both applications are running correctly:

# Check deployments
kubectl get deployments -n web-apps

# Check pods
kubectl get pods -n web-apps

# Check services
kubectl get svc -n web-apps

# Verify pods are ready
kubectl wait --for=condition=ready pod -l app=app1 -n web-apps --timeout=60s
kubectl wait --for=condition=ready pod -l app=app2 -n web-apps --timeout=60s
Task 3: Configure Ingress for Path-Based Routing
Subtask 3.1: Create Basic Ingress Resource
Create an Ingress resource that routes traffic based on URL paths:

# Create ingress-basic.yaml
cat > ingress-basic.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-apps-ingress
  namespace: web-apps
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: myapps.local
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
EOF

# Apply the Ingress configuration
kubectl apply -f ingress-basic.yaml
Subtask 3.2: Verify Ingress Configuration
Check the Ingress resource and get the external IP:

# Check ingress resource
kubectl get ingress -n web-apps

# Get detailed ingress information
kubectl describe ingress web-apps-ingress -n web-apps

# Get minikube IP for testing
minikube ip
Subtask 3.3: Configure Local DNS Resolution
Add entries to your local hosts file for testing:

# Get minikube IP
MINIKUBE_IP=$(minikube ip)
echo "Minikube IP: $MINIKUBE_IP"

# Add entry to hosts file (requires sudo)
echo "$MINIKUBE_IP myapps.local" | sudo tee -a /etc/hosts

# Verify the entry was added
grep myapps.local /etc/hosts
Subtask 3.4: Test Path-Based Routing
Test the routing configuration using curl:

# Test default path (should route to app1)
curl -H "Host: myapps.local" http://$(minikube ip)/

# Test app1 path
curl -H "Host: myapps.local" http://$(minikube ip)/app1

# Test app2 path
curl -H "Host: myapps.local" http://$(minikube ip)/app2

# Alternative testing using the domain name
curl http://myapps.local/app1
curl http://myapps.local/app2
Task 4: Secure Ingress with TLS Certificate
Subtask 4.1: Generate Self-Signed TLS Certificate
Create a self-signed certificate for testing purposes:

# Create private key
openssl genrsa -out tls.key 2048

# Create certificate signing request
openssl req -new -key tls.key -out tls.csr -subj "/CN=myapps.local/O=myapps.local"

# Generate self-signed certificate
openssl x509 -req -days 365 -in tls.csr -signkey tls.key -out tls.crt

# Verify certificate
openssl x509 -in tls.crt -text -noout | head -20
Subtask 4.2: Create Kubernetes TLS Secret
Store the certificate and key in a Kubernetes secret:

# Create TLS secret
kubectl create secret tls myapps-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n web-apps

# Verify secret creation
kubectl get secrets -n web-apps
kubectl describe secret myapps-tls-secret -n web-apps
Subtask 4.3: Update Ingress with TLS Configuration
Modify the Ingress resource to include TLS configuration:

# Create ingress-tls.yaml
cat > ingress-tls.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-apps-ingress
  namespace: web-apps
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapps.local
    secretName: myapps-tls-secret
  rules:
  - host: myapps.local
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
EOF

# Apply the updated Ingress configuration
kubectl apply -f ingress-tls.yaml
Subtask 4.4: Verify TLS Configuration
Check that the TLS configuration is working:

# Check ingress with TLS
kubectl describe ingress web-apps-ingress -n web-apps

# Wait for ingress to be ready
sleep 30

# Test HTTPS connection (ignore certificate warnings for self-signed cert)
curl -k https://myapps.local/app1
curl -k https://myapps.local/app2

# Test HTTP redirect to HTTPS
curl -v http://myapps.local/app1
Task 5: Advanced Routing and Verification
Subtask 5.1: Add Header-Based Routing
Create an advanced Ingress configuration with additional routing rules:

# Create ingress-advanced.yaml
cat > ingress-advanced.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-apps-ingress-advanced
  namespace: web-apps
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Served-By: Kubernetes-Ingress";
      more_set_headers "X-App-Version: 1.0";
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapps.local
    - api.myapps.local
    secretName: myapps-tls-secret
  rules:
  - host: myapps.local
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: api.myapps.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
EOF

# Apply the advanced configuration
kubectl apply -f ingress-advanced.yaml
Subtask 5.2: Update DNS Configuration
Add the new subdomain to your hosts file:

# Add API subdomain
MINIKUBE_IP=$(minikube ip)
echo "$MINIKUBE_IP api.myapps.local" | sudo tee -a /etc/hosts

# Verify both entries
grep myapps.local /etc/hosts
Subtask 5.3: Test Advanced Routing
Test the advanced routing configuration:

# Test main domain paths
curl -k -I https://myapps.local/app1
curl -k -I https://myapps.local/app2

# Test API subdomain
curl -k -I https://api.myapps.local/

# Check custom headers
curl -k -I https://myapps.local/app1 | grep "X-Served-By"
curl -k -I https://myapps.local/app1 | grep "X-App-Version"
Subtask 5.4: Monitor Ingress Logs
Check the Ingress controller logs to verify traffic routing:

# Get ingress controller pod name
INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')

# View ingress controller logs
kubectl logs -n ingress-nginx $INGRESS_POD --tail=50

# Follow logs in real-time (run in separate terminal)
kubectl logs -n ingress-nginx $INGRESS_POD -f
Task 6: Verification and Testing
Subtask 6.1: Comprehensive Testing Script
Create a comprehensive testing script:

# Create test-ingress.sh
cat > test-ingress.sh << 'EOF'
#!/bin/bash

echo "=== Ingress Testing Script ==="
echo "Testing HTTP/HTTPS routing and SSL termination"
echo

# Test HTTP redirect to HTTPS
echo "1. Testing HTTP to HTTPS redirect:"
curl -s -o /dev/null -w "HTTP Status: %{http_code}, Redirect URL: %{redirect_url}\n" http://myapps.local/app1
echo

# Test HTTPS app1
echo "2. Testing HTTPS app1 path:"
curl -k -s https://myapps.local/app1 | grep -o "<title>.*</title>"
echo

# Test HTTPS app2
echo "3. Testing HTTPS app2 path:"
curl -k -s https://myapps.local/app2 | grep -o "<title>.*</title>"
echo

# Test API subdomain
echo "4. Testing API subdomain:"
curl -k -s https://api.myapps.local/ | grep -o "<title>.*</title>"
echo

# Test SSL certificate
echo "5. Testing SSL certificate:"
echo | openssl s_client -servername myapps.local -connect $(minikube ip):443 2>/dev/null | openssl x509 -noout -subject
echo

# Test custom headers
echo "6. Testing custom headers:"
curl -k -s -I https://myapps.local/app1 | grep "X-Served-By"
curl -k -s -I https://myapps.local/app1 | grep "X-App-Version"
echo

echo "=== Testing Complete ==="
EOF

# Make script executable
chmod +x test-ingress.sh

# Run the test script
./test-ingress.sh
Subtask 6.2: Verify Ingress Resources
Check all Ingress-related resources:

# List all ingress resources
kubectl get ingress --all-namespaces

# Check ingress class
kubectl get ingressclass

# Verify services are accessible
kubectl get endpoints -n web-apps

# Check pod status
kubectl get pods -n web-apps -o wide
Subtask 6.3: Performance and Load Testing
Perform basic load testing to verify routing performance:

# Install apache2-utils for ab command (if not available)
sudo apt-get update && sudo apt-get install -y apache2-utils

# Perform load test on app1
ab -n 100 -c 10 -k https://myapps.local/app1

# Perform load test on app2
ab -n 100 -c 10 -k https://myapps.local/app2
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Ingress Controller Not Ready

# Check ingress controller status
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx -l app.kubernetes.io/component=controller

# Restart ingress controller if needed
kubectl delete pod -n ingress-nginx -l app.kubernetes.io/component=controller
Issue 2: DNS Resolution Problems

# Verify hosts file entries
cat /etc/hosts | grep myapps

# Test DNS resolution
nslookup myapps.local
ping myapps.local
Issue 3: Certificate Issues

# Check TLS secret
kubectl describe secret myapps-tls-secret -n web-apps

# Verify certificate validity
openssl x509 -in tls.crt -noout -dates
Issue 4: Service Not Accessible

# Check service endpoints
kubectl get endpoints -n web-apps

# Test service directly
kubectl port-forward -n web-apps svc/app1-service 8080:80
curl http://localhost:8080
Cleanup
When you're finished with the lab, clean up the resources:

# Delete ingress resources
kubectl delete ingress --all -n web-apps

# Delete applications
kubectl delete -f app1-deployment.yaml
kubectl delete -f app2-deployment.yaml

# Delete TLS secret
kubectl delete secret myapps-tls-secret -n web-apps

# Delete namespace
kubectl delete namespace web-apps

# Remove hosts file entries
sudo sed -i '/myapps.local/d' /etc/hosts

# Clean up certificate files
rm -f tls.key tls.csr tls.crt

# Clean up YAML files
rm -f *.yaml test-ingress.sh
Conclusion
Congratulations! You have successfully completed Lab 14: Advanced HTTP/S Routing with Ingress. In this lab, you accomplished several important tasks:

Key Achievements:

• Deployed Multiple Applications: You created two distinct web applications with different visual themes and deployed them in a Kubernetes cluster using Deployments and Services.

• Configured Path-Based Routing: You implemented sophisticated HTTP routing rules using Kubernetes Ingress resources, allowing different applications to be accessed via different URL paths (/app1, /app2).

• Implemented SSL/TLS Security: You generated self-signed certificates, created Kubernetes TLS secrets, and configured HTTPS termination at the Ingress level, ensuring secure communication.

• Advanced Routing Features: You explored advanced Ingress features including custom headers, multiple hostnames, and automatic HTTP-to-HTTPS redirects.

• Verification and Testing: You performed comprehensive testing using various tools and methods to verify that routing rules work correctly and SSL termination functions properly.

Why This Matters:

Ingress controllers are crucial components in modern Kubernetes deployments because they:

Provide External Access: Enable external users to access applications running inside the cluster
Centralize Traffic Management: Offer a single point of control for HTTP/HTTPS traffic routing
Enable SSL Termination: Handle certificate management and encryption/decryption at the edge
Support Advanced Features: Provide load balancing, path rewriting, and custom headers
Reduce Complexity: Eliminate the need for multiple LoadBalancer services
Real-World Applications:

The skills you've learned are directly applicable to:

Microservices Architecture: Routing traffic to different services based on URL paths
Multi-Tenant Applications: Serving different customers via different hostnames
API Gateway Patterns: Managing API traffic and implementing security policies
Blue-Green Deployments: Routing traffic between different application versions
Development Environments: Providing easy access to multiple applications in development clusters
This lab has provided you with practical experience in managing HTTP/HTTPS traffic in Kubernetes environments, a critical skill for the Kubernetes and Cloud Native Associate (KCNA) certification and real-world container orchestration scenarios.
