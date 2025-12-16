Lab 1: Building and Managing Container Images
Objectives
By the end of this lab, you will be able to:

• Create a Dockerfile for a simple web application from scratch • Implement multi-stage builds to optimize container image size and security • Build and tag container images using Docker • Set up and configure a private container registry using Harbor • Push container images to a private registry • Configure Kubernetes to authenticate and pull images from private registries • Deploy applications using private container images in Kubernetes • Understand best practices for container image management and security

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Linux command line operations • Familiarity with web applications and HTTP concepts • Basic knowledge of containerization concepts • Understanding of YAML file structure • No prior Docker or Kubernetes experience required - we'll guide you through everything

Lab Environment Setup
Good News! Al Nafi provides you with ready-to-use Linux-based cloud machines. Simply click Start Lab and you'll have access to a fully configured environment with all necessary tools pre-installed including:

• Docker Engine • kubectl (Kubernetes command-line tool) • Harbor container registry • A single-node Kubernetes cluster (minikube) • Text editors (nano, vim)

No need to build your own VM or install any software!

Task 1: Create a Dockerfile for a Simple Web Application
Subtask 1.1: Create the Application Directory Structure
First, let's create a workspace for our web application:

mkdir -p ~/container-lab/webapp
cd ~/container-lab/webapp
Subtask 1.2: Create a Simple Web Application
Create a basic HTML file for our web application:

cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Container Lab Web App</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            text-align: center;
            padding: 50px;
        }
        .container {
            max-width: 600px;
            margin: 0 auto;
            background: rgba(255,255,255,0.1);
            padding: 30px;
            border-radius: 10px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Welcome to Container Lab!</h1>
        <p>This web application is running inside a Docker container.</p>
        <p>Container ID: <span id="hostname"></span></p>
        <p>Current Time: <span id="time"></span></p>
    </div>
    <script>
        document.getElementById('hostname').textContent = window.location.hostname;
        document.getElementById('time').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
EOF
Create a simple configuration file for nginx:

cat > nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;
    
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
EOF
Subtask 1.3: Create the Initial Dockerfile
Create a basic Dockerfile that will serve our web application:

cat > Dockerfile << 'EOF'
# Basic Dockerfile (we'll optimize this later)
FROM nginx:1.25-alpine

# Copy our web application files
COPY index.html /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
EOF
Subtask 1.4: Build and Test the Basic Image
Build the container image:

docker build -t webapp-basic:v1.0 .
Run the container to test it:

docker run -d --name webapp-test -p 8080:80 webapp-basic:v1.0
Test the application:

curl http://localhost:8080
You should see the HTML content of your web application. Clean up the test container:

docker stop webapp-test
docker rm webapp-test
Task 2: Use Multi-Stage Builds to Optimize Image Size
Subtask 2.1: Create an Enhanced Web Application
Let's create a more complex application that requires a build process. First, create a simple Node.js application:

cat > package.json << 'EOF'
{
  "name": "container-lab-webapp",
  "version": "1.0.0",
  "description": "A simple web application for container lab",
  "main": "server.js",
  "scripts": {
    "build": "mkdir -p dist && cp -r public/* dist/ && node build.js",
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "uglify-js": "^3.17.4"
  }
}
EOF
Create the build script:

cat > build.js << 'EOF'
const fs = require('fs');
const UglifyJS = require('uglify-js');

// Read the source JavaScript file
const sourceCode = fs.readFileSync('public/app.js', 'utf8');

// Minify the JavaScript
const minified = UglifyJS.minify(sourceCode);

if (minified.error) {
    console.error('Minification error:', minified.error);
    process.exit(1);
}

// Write the minified code to dist directory
fs.writeFileSync('dist/app.min.js', minified.code);

// Copy and optimize HTML
let html = fs.readFileSync('public/index.html', 'utf8');
html = html.replace('app.js', 'app.min.js');
html = html.replace(/\s+/g, ' ').trim(); // Basic HTML minification

fs.writeFileSync('dist/index.html', html);

console.log('Build completed successfully!');
EOF
Create the public directory and files:

mkdir -p public

cat > public/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Optimized Container Lab App</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            margin: 0;
            padding: 20px;
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .container {
            max-width: 800px;
            background: rgba(255,255,255,0.1);
            padding: 40px;
            border-radius: 15px;
            box-shadow: 0 8px 32px rgba(0,0,0,0.3);
            backdrop-filter: blur(10px);
            text-align: center;
        }
        .stats {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            margin-top: 30px;
        }
        .stat-card {
            background: rgba(255,255,255,0.1);
            padding: 20px;
            border-radius: 10px;
            border: 1px solid rgba(255,255,255,0.2);
        }
        .refresh-btn {
            background: #4CAF50;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
            margin-top: 20px;
            font-size: 16px;
        }
        .refresh-btn:hover {
            background: #45a049;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Optimized Container Lab Application</h1>
        <p>This application demonstrates multi-stage Docker builds and optimization techniques.</p>
        
        <div class="stats">
            <div class="stat-card">
                <h3>Container Info</h3>
                <p><strong>Hostname:</strong> <span id="hostname"></span></p>
                <p><strong>User Agent:</strong> <span id="userAgent"></span></p>
            </div>
            <div class="stat-card">
                <h3>Time Info</h3>
                <p><strong>Current Time:</strong> <span id="currentTime"></span></p>
                <p><strong>Uptime:</strong> <span id="uptime"></span></p>
            </div>
            <div class="stat-card">
                <h3>Performance</h3>
                <p><strong>Load Time:</strong> <span id="loadTime"></span>ms</p>
                <p><strong>Memory Usage:</strong> <span id="memoryUsage"></span>MB</p>
            </div>
        </div>
        
        <button class="refresh-btn" onclick="updateStats()">🔄 Refresh Stats</button>
    </div>
    
    <script src="app.js"></script>
</body>
</html>
EOF
Create the JavaScript application:

cat > public/app.js << 'EOF'
// Application start time
const startTime = Date.now();

// Update statistics function
function updateStats() {
    // Update hostname
    document.getElementById('hostname').textContent = window.location.hostname || 'localhost';
    
    // Update user agent (simplified)
    const ua = navigator.userAgent;
    const browser = ua.includes('Chrome') ? 'Chrome' : 
                   ua.includes('Firefox') ? 'Firefox' : 
                   ua.includes('Safari') ? 'Safari' : 'Other';
    document.getElementById('userAgent').textContent = browser;
    
    // Update current time
    document.getElementById('currentTime').textContent = new Date().toLocaleString();
    
    // Calculate uptime
    const uptime = Math.floor((Date.now() - startTime) / 1000);
    document.getElementById('uptime').textContent = uptime + ' seconds';
    
    // Calculate load time
    const loadTime = Date.now() - startTime;
    document.getElementById('loadTime').textContent = loadTime;
    
    // Estimate memory usage (approximate)
    const memoryUsage = performance.memory ? 
        Math.round(performance.memory.usedJSHeapSize / 1024 / 1024 * 100) / 100 : 
        'N/A';
    document.getElementById('memoryUsage').textContent = memoryUsage;
}

// Initialize stats when page loads
document.addEventListener('DOMContentLoaded', function() {
    updateStats();
    
    // Auto-refresh every 30 seconds
    setInterval(updateStats, 30000);
});

// Add some interactive features
document.addEventListener('click', function(e) {
    // Create ripple effect on clicks
    const ripple = document.createElement('div');
    ripple.style.position = 'fixed';
    ripple.style.borderRadius = '50%';
    ripple.style.background = 'rgba(255,255,255,0.3)';
    ripple.style.transform = 'scale(0)';
    ripple.style.animation = 'ripple 0.6s linear';
    ripple.style.left = (e.clientX - 25) + 'px';
    ripple.style.top = (e.clientY - 25) + 'px';
    ripple.style.width = '50px';
    ripple.style.height = '50px';
    ripple.style.pointerEvents = 'none';
    
    document.body.appendChild(ripple);
    
    setTimeout(() => {
        document.body.removeChild(ripple);
    }, 600);
});

// Add CSS for ripple animation
const style = document.createElement('style');
style.textContent = `
    @keyframes ripple {
        to {
            transform: scale(4);
            opacity: 0;
        }
    }
`;
document.head.appendChild(style);

console.log('Container Lab Application loaded successfully!');
EOF
Subtask 2.2: Create the Multi-Stage Dockerfile
Now create an optimized Dockerfile using multi-stage builds:

cat > Dockerfile.multistage << 'EOF'
# Multi-stage Dockerfile for optimized image size

# Stage 1: Build stage
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev dependencies)
RUN npm install

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Stage 2: Production stage
FROM nginx:1.25-alpine AS production

# Install curl for health checks
RUN apk add --no-cache curl

# Create a non-root user for security
RUN addgroup -g 1001 -S nginx-user && \
    adduser -S -D -H -u 1001 -h /var/cache/nginx -s /sbin/nologin -G nginx-user -g nginx-user nginx-user

# Copy built application from builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Create nginx directories and set permissions
RUN mkdir -p /var/cache/nginx/client_temp && \
    mkdir -p /var/cache/nginx/proxy_temp && \
    mkdir -p /var/cache/nginx/fastcgi_temp && \
    mkdir -p /var/cache/nginx/uwsgi_temp && \
    mkdir -p /var/cache/nginx/scgi_temp && \
    chown -R nginx-user:nginx-user /var/cache/nginx && \
    chown -R nginx-user:nginx-user /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

# Expose port
EXPOSE 80

# Switch to non-root user
USER nginx-user

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
EOF
Subtask 2.3: Compare Image Sizes
Build both versions and compare their sizes:

# Build the basic version
docker build -t webapp-basic:v1.0 -f Dockerfile .

# Build the optimized version
docker build -t webapp-optimized:v1.0 -f Dockerfile.multistage .

# Compare image sizes
echo "=== Image Size Comparison ==="
docker images | grep webapp
You should see a significant difference in image sizes. The multi-stage build removes development dependencies and build tools from the final image.

Subtask 2.4: Test the Optimized Application
Test the optimized application:

# Run the optimized container
docker run -d --name webapp-optimized -p 8081:80 webapp-optimized:v1.0

# Test the application
curl http://localhost:8081

# Test the health check endpoint
curl http://localhost:8081/health

# Clean up
docker stop webapp-optimized
docker rm webapp-optimized
Task 3: Set Up Private Container Registry and Configure Kubernetes
Subtask 3.1: Start Harbor Private Registry
Harbor is already installed in your environment. Let's start it:

# Navigate to Harbor directory
cd ~/harbor

# Start Harbor
sudo docker-compose up -d

# Wait for Harbor to be ready (this may take a few minutes)
echo "Waiting for Harbor to start..."
sleep 60

# Check Harbor status
sudo docker-compose ps
Subtask 3.2: Configure Harbor Access
Access Harbor web interface and create a project:

# Get Harbor admin password
echo "Harbor admin password:"
sudo cat ~/harbor/harbor.yml | grep harbor_admin_password

# Harbor will be available at: https://localhost
# Default username: admin
# Use the password from above
Since we're in a lab environment, let's configure Docker to work with Harbor's self-signed certificate:

# Create Docker daemon configuration for insecure registry
sudo mkdir -p /etc/docker

sudo cat > /etc/docker/daemon.json << 'EOF'
{
  "insecure-registries": ["localhost", "127.0.0.1", "harbor.local"]
}
EOF

# Restart Docker daemon
sudo systemctl restart docker

# Wait for Docker to restart
sleep 10
Subtask 3.3: Create Harbor Project via API
Let's create a project in Harbor using the API:

# Get the admin password
HARBOR_PASSWORD=$(sudo cat ~/harbor/harbor.yml | grep harbor_admin_password | awk '{print $2}')

# Create a project called 'container-lab'
curl -X POST "http://localhost/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -u "admin:${HARBOR_PASSWORD}" \
  -d '{
    "project_name": "container-lab",
    "public": false,
    "metadata": {
      "public": "false"
    }
  }'

echo "Project created successfully!"
Subtask 3.4: Tag and Push Images to Harbor
Login to Harbor and push our images:

# Login to Harbor
echo "${HARBOR_PASSWORD}" | docker login localhost -u admin --password-stdin

# Tag images for Harbor
docker tag webapp-optimized:v1.0 localhost/container-lab/webapp:v1.0
docker tag webapp-optimized:v1.0 localhost/container-lab/webapp:latest

# Push images to Harbor
docker push localhost/container-lab/webapp:v1.0
docker push localhost/container-lab/webapp:latest

# Verify images are in Harbor
curl -u "admin:${HARBOR_PASSWORD}" "http://localhost/api/v2.0/projects/container-lab/repositories"
Subtask 3.5: Start Minikube and Configure Registry Access
Start the Kubernetes cluster:

# Start minikube
minikube start --driver=docker

# Wait for minikube to be ready
kubectl wait --for=condition=Ready nodes --all --timeout=300s

# Check cluster status
kubectl cluster-info
Subtask 3.6: Create Kubernetes Secret for Private Registry
Create a secret to authenticate with Harbor:

# Create namespace for our application
kubectl create namespace container-lab

# Create Docker registry secret
kubectl create secret docker-registry harbor-secret \
  --docker-server=localhost \
  --docker-username=admin \
  --docker-password="${HARBOR_PASSWORD}" \
  --docker-email=admin@harbor.local \
  -n container-lab

# Verify secret creation
kubectl get secrets -n container-lab
Subtask 3.7: Create Kubernetes Deployment
Create a deployment manifest for our application:

cd ~/container-lab

cat > webapp-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  namespace: container-lab
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      imagePullSecrets:
      - name: harbor-secret
      containers:
      - name: webapp
        image: localhost/container-lab/webapp:v1.0
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
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  namespace: container-lab
spec:
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: container-lab
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
EOF
Subtask 3.8: Deploy Application to Kubernetes
Deploy the application:

# Apply the deployment
kubectl apply -f webapp-deployment.yaml

# Wait for deployment to be ready
kubectl wait --for=condition=available --timeout=300s deployment/webapp-deployment -n container-lab

# Check deployment status
kubectl get deployments -n container-lab
kubectl get pods -n container-lab
kubectl get services -n container-lab
Subtask 3.9: Test the Deployed Application
Test the application running in Kubernetes:

# Port forward to access the application
kubectl port-forward service/webapp-service 8082:80 -n container-lab &

# Wait a moment for port forwarding to establish
sleep 5

# Test the application
curl http://localhost:8082

# Test health endpoint
curl http://localhost:8082/health

# Stop port forwarding
pkill -f "kubectl port-forward"
Subtask 3.10: Verify Image Pull from Private Registry
Check that Kubernetes successfully pulled the image from Harbor:

# Check events to see image pull
kubectl get events -n container-lab --sort-by='.lastTimestamp'

# Describe a pod to see image details
POD_NAME=$(kubectl get pods -n container-lab -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD_NAME -n container-lab

# Check image information
kubectl get pods -n container-lab -o jsonpath='{.items[*].spec.containers[*].image}'
Task 4: Advanced Container Management
Subtask 4.1: Implement Image Scanning and Security
Let's scan our image for vulnerabilities using Harbor's built-in scanner:

# Trigger vulnerability scan via API
curl -X POST "http://localhost/api/v2.0/projects/container-lab/repositories/webapp/artifacts/v1.0/scan" \
  -u "admin:${HARBOR_PASSWORD}"

# Wait for scan to complete
sleep 30

# Get scan results
curl -u "admin:${HARBOR_PASSWORD}" \
  "http://localhost/api/v2.0/projects/container-lab/repositories/webapp/artifacts/v1.0" | \
  python3 -m json.tool
Subtask 4.2: Create Image Update Strategy
Create a script to update and redeploy the application:

cat > update-app.sh << 'EOF'
#!/bin/bash

# Update application script
VERSION=${1:-v1.1}
echo "Updating application to version: $VERSION"

# Build new version
docker build -t webapp-optimized:$VERSION -f Dockerfile.multistage .

# Tag for Harbor
docker tag webapp-optimized:$VERSION localhost/container-lab/webapp:$VERSION

# Push to Harbor
docker push localhost/container-lab/webapp:$VERSION

# Update Kubernetes deployment
kubectl set image deployment/webapp-deployment webapp=localhost/container-lab/webapp:$VERSION -n container-lab

# Wait for rollout to complete
kubectl rollout status deployment/webapp-deployment -n container-lab

echo "Application updated successfully to version: $VERSION"
EOF

chmod +x update-app.sh
Subtask 4.3: Monitor Application Performance
Create monitoring commands:

# Monitor pod resource usage
kubectl top pods -n container-lab

# Watch deployment status
kubectl get pods -n container-lab -w &
WATCH_PID=$!

# Generate some load to test scaling
kubectl run load-generator --image=busybox --restart=Never -n container-lab -- \
  /bin/sh -c "while true; do wget -q -O- http://webapp-service/; sleep 1; done"

# Wait a bit to see the load
sleep 30

# Stop monitoring
kill $WATCH_PID

# Clean up load generator
kubectl delete pod load-generator -n container-lab
Troubleshooting Common Issues
Issue 1: Harbor Not Starting
If Harbor fails to start:

# Check Harbor logs
sudo docker-compose -f ~/harbor/docker-compose.yml logs

# Restart Harbor
cd ~/harbor
sudo docker-compose down
sudo docker-compose up -d
Issue 2: Image Pull Errors
If Kubernetes can't pull images:

# Check secret configuration
kubectl get secret harbor-secret -n container-lab -o yaml

# Recreate secret if needed
kubectl delete secret harbor-secret -n container-lab
kubectl create secret docker-registry harbor-secret \
  --docker-server=localhost \
  --docker-username=admin \
  --docker-password="${HARBOR_PASSWORD}" \
  --docker-email=admin@harbor.local \
  -n container-lab
Issue 3: Pod Not Starting
If pods fail to start:

# Check pod events
kubectl describe pod $POD_NAME -n container-lab

# Check logs
kubectl logs $POD_NAME -n container-lab

# Check resource constraints
kubectl get pods -n container-lab -o wide
Cleanup
When you're finished with the lab, clean up resources:

# Delete Kubernetes resources
kubectl delete namespace container-lab

# Stop Harbor
cd ~/harbor
sudo docker-compose down

# Remove Docker images
docker rmi webapp-basic:v1.0 webapp-optimized:v1.0
docker rmi localhost/container-lab/webapp:v1.0 localhost/container-lab/webapp:latest

# Stop minikube
minikube stop

# Clean up lab directory
rm -rf ~/container-lab
Conclusion
Congratulations! You have successfully completed Lab 1: Building and Managing Container Images. Here's what you accomplished:

Key Achievements:

• Created Professional Dockerfiles: You built both basic and optimized Dockerfiles, learning the difference between single-stage and multi-stage builds

• Implemented Security Best Practices: Your containers run as non-root users and include health checks for better security and reliability

• Optimized Image Size: Through multi-stage builds, you reduced image size by removing unnecessary build dependencies and tools

• Set Up Private Registry: You configured Harbor as a private container registry, understanding how organizations manage their container images securely

• Configured Kubernetes Authentication: You learned how to create and use Docker registry secrets for private image access

• Deployed Production-Ready Applications: Your Kubernetes deployment includes proper resource limits, health checks, and scaling capabilities

• Implemented Monitoring and Updates: You created scripts for application updates and learned to monitor container performance

Why This Matters:

In real-world scenarios, these skills are essential for:

DevOps Engineers who need to build and manage container pipelines
Application Developers preparing for cloud-native deployments
System Administrators managing containerized infrastructure
Security Teams ensuring container images meet compliance requirements
Next Steps:

This lab provides a solid foundation for the Certified Kubernetes Application Developer (CKAD) certification, specifically covering:

Container image creation and management
Application deployment and configuration
Services and networking
Security contexts and best practices
You're now ready to tackle more advanced Kubernetes topics like persistent storage, advanced networking, and application scaling strategies!
