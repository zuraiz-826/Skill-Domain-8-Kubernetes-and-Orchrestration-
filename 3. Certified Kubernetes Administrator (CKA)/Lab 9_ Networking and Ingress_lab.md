Lab 9: Networking and Ingress
Objectives
By the end of this lab, you will be able to:

• Deploy a web application in a Kubernetes cluster • Create and configure a NodePort service to expose applications • Implement Ingress resources to manage HTTP traffic routing • Configure TLS certificates to secure Ingress routes • Understand the differences between Services and Ingress in Kubernetes networking • Troubleshoot common networking and ingress issues

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with YAML configuration files • Basic knowledge of HTTP/HTTPS protocols • Understanding of DNS concepts • Experience with command-line interface (CLI) operations

Required Tools: • kubectl (Kubernetes command-line tool) • A text editor (nano, vim, or any preferred editor) • Access to a Kubernetes cluster

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment. No need to build your own VM or install additional software - everything is ready to use!

Your lab environment includes: • Ubuntu Linux machine with kubectl pre-installed • Access to a Kubernetes cluster • All required networking tools and utilities

Task 1: Deploy a Web Application and Expose it Using NodePort Service
Subtask 1.1: Create a Simple Web Application Deployment
First, let's create a simple web application that we can use for testing our networking configurations.

Create a new directory for your lab files:
mkdir ~/lab9-networking
cd ~/lab9-networking
Create a deployment file for a simple nginx web application:
cat > web-app-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
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
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-content
        configMap:
          name: web-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Lab 9 - Networking Demo</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; background-color: #f0f8ff; }
            .container { max-width: 800px; margin: 0 auto; padding: 20px; }
            h1 { color: #2c3e50; text-align: center; }
            .info { background-color: #e8f4fd; padding: 15px; border-radius: 5px; margin: 20px 0; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Welcome to Lab 9: Networking and Ingress</h1>
            <div class="info">
                <h3>Application Information:</h3>
                <p><strong>Service:</strong> Web Application Demo</p>
                <p><strong>Version:</strong> 1.0</p>
                <p><strong>Purpose:</strong> Testing Kubernetes Networking and Ingress</p>
            </div>
            <div class="info">
                <h3>Current Status:</h3>
                <p>✅ Application is running successfully</p>
                <p>✅ Connected via Kubernetes Service</p>
                <p id="connection-type">🔍 Checking connection type...</p>
            </div>
        </div>
        <script>
            // Simple script to show connection information
            document.getElementById('connection-type').innerHTML = 
                '🌐 Connected via: ' + window.location.protocol + '//' + window.location.host;
        </script>
    </body>
    </html>
EOF
Deploy the application to your Kubernetes cluster:
kubectl apply -f web-app-deployment.yaml
Verify that the deployment was created successfully:
kubectl get deployments
kubectl get pods -l app=web-app
You should see output similar to:

NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-app   3/3     3            3           30s
Subtask 1.2: Create a NodePort Service
Now let's create a NodePort service to expose our web application.

Create a NodePort service configuration:
cat > nodeport-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
  labels:
    app: web-app
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
    name: http
EOF
Apply the NodePort service:
kubectl apply -f nodeport-service.yaml
Verify the service was created:
kubectl get services
kubectl describe service web-app-nodeport
Subtask 1.3: Test the NodePort Service
Get the node IP address:
kubectl get nodes -o wide
Test the application using the NodePort:
# Replace <NODE_IP> with the actual IP from the previous command
curl http://<NODE_IP>:30080
If you're working in a local environment, you can also test using localhost:

curl http://localhost:30080
Key Concept: NodePort services expose applications on a specific port (30000-32767 range) on all cluster nodes. This allows external access but requires knowledge of node IPs and specific ports.

Task 2: Configure an Ingress Resource to Route HTTP Traffic
Subtask 2.1: Install and Configure Ingress Controller
Before creating Ingress resources, we need an Ingress Controller. We'll use NGINX Ingress Controller.

Install the NGINX Ingress Controller:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
Wait for the ingress controller to be ready:
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
Verify the ingress controller is running:
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
Subtask 2.2: Create a ClusterIP Service for Ingress
Ingress typically works with ClusterIP services, so let's create one for our application.

Create a ClusterIP service:
cat > clusterip-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  labels:
    app: web-app
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
EOF
Apply the ClusterIP service:
kubectl apply -f clusterip-service.yaml
Verify the service:
kubectl get service web-app-service
Subtask 2.3: Create an Ingress Resource
Now let's create an Ingress resource to route HTTP traffic to our application.

Create an Ingress configuration:
cat > web-app-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
  - host: webapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
EOF
Apply the Ingress resource:
kubectl apply -f web-app-ingress.yaml
Verify the Ingress was created:
kubectl get ingress
kubectl describe ingress web-app-ingress
Subtask 2.4: Test the Ingress Configuration
Get the Ingress Controller's external IP or use port forwarding:
# Check if external IP is assigned
kubectl get service -n ingress-nginx ingress-nginx-controller

# If no external IP, use port forwarding for testing
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80 &
Add entries to your hosts file for testing (if using port forwarding):
# Add these entries to /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)
echo "127.0.0.1 webapp.local" | sudo tee -a /etc/hosts
echo "127.0.0.1 webapp.example.com" | sudo tee -a /etc/hosts
Test the Ingress routing:
# Test with different hostnames
curl -H "Host: webapp.local" http://localhost:8080
curl -H "Host: webapp.example.com" http://localhost:8080

# Or if you added hosts entries:
curl http://webapp.local:8080
curl http://webapp.example.com:8080
Key Concept: Ingress provides HTTP and HTTPS routing to services based on hostnames and paths. It's more flexible than NodePort services and provides a single entry point for multiple services.

Task 3: Add TLS Certificates to Secure the Ingress Route
Subtask 3.1: Generate Self-Signed TLS Certificates
For this lab, we'll create self-signed certificates. In production, you would use certificates from a trusted Certificate Authority.

Create a private key and certificate:
# Create private key
openssl genrsa -out webapp-tls.key 2048

# Create certificate signing request
openssl req -new -key webapp-tls.key -out webapp-tls.csr -subj "/CN=webapp.local/O=webapp.local"

# Create self-signed certificate
openssl x509 -req -in webapp-tls.csr -signkey webapp-tls.key -out webapp-tls.crt -days 365

# Create certificate for second domain
openssl req -new -key webapp-tls.key -out webapp-example.csr -subj "/CN=webapp.example.com/O=webapp.example.com"
openssl x509 -req -in webapp-example.csr -signkey webapp-tls.key -out webapp-example.crt -days 365
Verify the certificates:
openssl x509 -in webapp-tls.crt -text -noout | grep -A 1 "Subject:"
openssl x509 -in webapp-example.crt -text -noout | grep -A 1 "Subject:"
Subtask 3.2: Create Kubernetes TLS Secrets
Create TLS secrets for both domains:
# Create secret for webapp.local
kubectl create secret tls webapp-local-tls \
  --cert=webapp-tls.crt \
  --key=webapp-tls.key

# Create secret for webapp.example.com
kubectl create secret tls webapp-example-tls \
  --cert=webapp-example.crt \
  --key=webapp-tls.key
Verify the secrets were created:
kubectl get secrets
kubectl describe secret webapp-local-tls
kubectl describe secret webapp-example-tls
Subtask 3.3: Update Ingress with TLS Configuration
Create an updated Ingress configuration with TLS:
cat > web-app-ingress-tls.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress-tls
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - webapp.local
    secretName: webapp-local-tls
  - hosts:
    - webapp.example.com
    secretName: webapp-example-tls
  rules:
  - host: webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
  - host: webapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
EOF
Apply the updated Ingress:
kubectl apply -f web-app-ingress-tls.yaml
Verify the TLS configuration:
kubectl get ingress web-app-ingress-tls
kubectl describe ingress web-app-ingress-tls
Subtask 3.4: Test HTTPS Access
Set up port forwarding for HTTPS (if not using external load balancer):
# Stop previous port forwarding if running
pkill -f "kubectl port-forward"

# Start port forwarding for both HTTP and HTTPS
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80 8443:443 &
Test HTTPS access:
# Test HTTPS with self-signed certificates (ignore certificate warnings)
curl -k -H "Host: webapp.local" https://localhost:8443
curl -k -H "Host: webapp.example.com" https://localhost:8443

# Test HTTP redirect to HTTPS
curl -v -H "Host: webapp.local" http://localhost:8080
Test with a web browser (optional):
If you have a desktop environment, you can test in a web browser:

Navigate to https://webapp.local:8443
Accept the self-signed certificate warning
Verify the application loads over HTTPS
Subtask 3.5: Verify TLS Certificate Details
Check the certificate details using openssl:
# Check certificate served by the ingress
echo | openssl s_client -servername webapp.local -connect localhost:8443 2>/dev/null | openssl x509 -noout -text | grep -A 1 "Subject:"

# Check certificate expiration
echo | openssl s_client -servername webapp.local -connect localhost:8443 2>/dev/null | openssl x509 -noout -dates
Troubleshooting Common Issues
Issue 1: Ingress Controller Not Ready
Symptoms: Ingress resources created but not accessible

Solution:

# Check ingress controller status
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Restart if necessary
kubectl rollout restart deployment/ingress-nginx-controller -n ingress-nginx
Issue 2: DNS Resolution Problems
Symptoms: Cannot access application using hostnames

Solution:

# Verify hosts file entries
cat /etc/hosts | grep webapp

# Test with IP and Host header instead
curl -H "Host: webapp.local" http://<INGRESS_IP>
Issue 3: TLS Certificate Issues
Symptoms: Certificate warnings or HTTPS not working

Solution:

# Verify TLS secrets
kubectl get secrets
kubectl describe secret webapp-local-tls

# Check certificate validity
openssl x509 -in webapp-tls.crt -noout -dates
Issue 4: Service Not Reachable
Symptoms: 503 Service Temporarily Unavailable

Solution:

# Check if pods are running
kubectl get pods -l app=web-app

# Verify service endpoints
kubectl get endpoints web-app-service

# Check service selector matches pod labels
kubectl describe service web-app-service
Verification and Testing
Complete System Test
Create a comprehensive test script:
cat > test-networking.sh << 'EOF'
#!/bin/bash

echo "=== Lab 9 Networking Test Script ==="
echo

# Test 1: NodePort Service
echo "1. Testing NodePort Service..."
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:30080)
if [ "$response" = "200" ]; then
    echo "✅ NodePort service working (HTTP $response)"
else
    echo "❌ NodePort service failed (HTTP $response)"
fi

# Test 2: HTTP Ingress
echo "2. Testing HTTP Ingress..."
response=$(curl -s -o /dev/null -w "%{http_code}" -H "Host: webapp.local" http://localhost:8080)
if [ "$response" = "200" ] || [ "$response" = "301" ] || [ "$response" = "302" ]; then
    echo "✅ HTTP Ingress working (HTTP $response)"
else
    echo "❌ HTTP Ingress failed (HTTP $response)"
fi

# Test 3: HTTPS Ingress
echo "3. Testing HTTPS Ingress..."
response=$(curl -k -s -o /dev/null -w "%{http_code}" -H "Host: webapp.local" https://localhost:8443)
if [ "$response" = "200" ]; then
    echo "✅ HTTPS Ingress working (HTTP $response)"
else
    echo "❌ HTTPS Ingress failed (HTTP $response)"
fi

# Test 4: TLS Certificate
echo "4. Testing TLS Certificate..."
cert_subject=$(echo | openssl s_client -servername webapp.local -connect localhost:8443 2>/dev/null | openssl x509 -noout -subject 2>/dev/null)
if [[ "$cert_subject" == *"webapp.local"* ]]; then
    echo "✅ TLS certificate valid for webapp.local"
else
    echo "❌ TLS certificate issue"
fi

echo
echo "=== Test Complete ==="
EOF

chmod +x test-networking.sh
./test-networking.sh
Resource Summary
Check all created resources:
echo "=== Lab 9 Resource Summary ==="
echo
echo "Deployments:"
kubectl get deployments

echo
echo "Services:"
kubectl get services

echo
echo "Ingress Resources:"
kubectl get ingress

echo
echo "TLS Secrets:"
kubectl get secrets | grep tls

echo
echo "ConfigMaps:"
kubectl get configmaps
Cleanup (Optional)
If you want to clean up the resources created in this lab:

# Delete Ingress resources
kubectl delete ingress web-app-ingress web-app-ingress-tls

# Delete services
kubectl delete service web-app-nodeport web-app-service

# Delete deployment and configmap
kubectl delete -f web-app-deployment.yaml

# Delete TLS secrets
kubectl delete secret webapp-local-tls webapp-example-tls

# Remove hosts file entries (if added)
sudo sed -i '/webapp.local/d' /etc/hosts
sudo sed -i '/webapp.example.com/d' /etc/hosts

# Stop port forwarding
pkill -f "kubectl port-forward"

# Clean up certificate files
rm -f webapp-*.key webapp-*.crt webapp-*.csr

# Remove lab directory
cd ~
rm -rf lab9-networking
Conclusion
Congratulations! You have successfully completed Lab 9: Networking and Ingress. In this lab, you accomplished the following:

Key Achievements:

• Deployed a Web Application: Created a multi-replica nginx deployment with custom content using ConfigMaps • Configured NodePort Service: Exposed your application using NodePort, learning how Kubernetes provides external access through specific ports on cluster nodes • Implemented Ingress Resources: Set up NGINX Ingress Controller and created Ingress rules for HTTP traffic routing based on hostnames • Secured with TLS: Generated TLS certificates, created Kubernetes secrets, and configured HTTPS access with automatic HTTP-to-HTTPS redirection

Why This Matters:

Production Readiness: Understanding Kubernetes networking is crucial for deploying applications in production environments. The skills you've learned here are directly applicable to real-world scenarios where applications need to be accessible to users.

Security Best Practices: By implementing TLS encryption, you've learned how to secure communication between clients and your applications, which is essential for protecting sensitive data.

Scalability and Flexibility: Ingress provides a scalable way to manage external access to multiple services through a single entry point, making it easier to manage complex applications with multiple components.

Cost Efficiency: Using Ingress instead of multiple LoadBalancer services can significantly reduce cloud infrastructure costs while providing more sophisticated routing capabilities.

Career Relevance: These networking concepts are fundamental for the Certified Kubernetes Application Developer (CKAD) certification and are essential skills for DevOps engineers, Platform engineers, and Kubernetes administrators.

You now have hands-on experience with the complete networking stack in Kubernetes, from basic service exposure to advanced ingress configuration with TLS termination. These skills will serve as a foundation for more advanced topics like service mesh, advanced load balancing, and multi-cluster networking.
