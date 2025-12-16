Lab 12: Deploying a Multi-Tier Application in Kubernetes
Objectives
By the end of this lab, you will be able to:

• Deploy a complete multi-tier application architecture in Kubernetes consisting of frontend, backend, and database components • Create and configure separate Pods for each application tier • Implement Kubernetes Services to enable secure communication between application tiers • Utilize ConfigMaps to manage application configuration data externally • Understand the principles of microservices architecture in containerized environments • Apply best practices for service discovery and inter-pod communication in Kubernetes • Troubleshoot common connectivity issues in multi-tier Kubernetes deployments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with YAML syntax and file structure • Basic knowledge of containerization concepts • Understanding of web application architecture (frontend, backend, database) • Experience with command-line interface operations • Basic networking concepts (ports, IP addresses, DNS)

Note: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed. Simply click "Start Lab" to access your environment - no need to build your own VM or install Kubernetes manually.

Lab Environment Setup
Task 1: Verify Kubernetes Cluster Status
Subtask 1.1: Check Cluster Information
First, let's verify that your Kubernetes cluster is running properly.

# Check cluster information
kubectl cluster-info

# Verify node status
kubectl get nodes

# Check if all system pods are running
kubectl get pods -n kube-system
Subtask 1.2: Create Lab Namespace
Create a dedicated namespace for this lab to keep resources organized.

# Create namespace for the lab
kubectl create namespace multi-tier-app

# Set the namespace as default for this session
kubectl config set-context --current --namespace=multi-tier-app

# Verify namespace creation
kubectl get namespaces
Task 2: Deploy the Database Tier
Subtask 2.1: Create Database ConfigMap
Create a ConfigMap to store database configuration parameters.

# Create database configuration file
cat > database-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
  namespace: multi-tier-app
data:
  MYSQL_DATABASE: "webapp_db"
  MYSQL_USER: "webapp_user"
  DB_HOST: "database-service"
  DB_PORT: "3306"
EOF

# Apply the ConfigMap
kubectl apply -f database-config.yaml

# Verify ConfigMap creation
kubectl get configmaps
kubectl describe configmap database-config
Subtask 2.2: Create Database Secret
Create a Secret to store sensitive database credentials.

# Create database secret
kubectl create secret generic database-secret \
  --from-literal=MYSQL_ROOT_PASSWORD=rootpassword123 \
  --from-literal=MYSQL_PASSWORD=userpassword123

# Verify secret creation
kubectl get secrets
kubectl describe secret database-secret
Subtask 2.3: Deploy Database Pod
Create a MySQL database deployment.

# Create database deployment file
cat > database-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-deployment
  namespace: multi-tier-app
  labels:
    app: database
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: database-config
              key: MYSQL_DATABASE
        - name: MYSQL_USER
          valueFrom:
            configMapKeyRef:
              name: database-config
              key: MYSQL_USER
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: MYSQL_PASSWORD
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        emptyDir: {}
EOF

# Apply the database deployment
kubectl apply -f database-deployment.yaml

# Check deployment status
kubectl get deployments
kubectl get pods -l tier=database
Subtask 2.4: Create Database Service
Create a Service to expose the database within the cluster.

# Create database service file
cat > database-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: multi-tier-app
  labels:
    app: database
    tier: database
spec:
  selector:
    app: database
    tier: database
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
  type: ClusterIP
EOF

# Apply the database service
kubectl apply -f database-service.yaml

# Verify service creation
kubectl get services
kubectl describe service database-service
Task 3: Deploy the Backend Tier
Subtask 3.1: Create Backend ConfigMap
Create configuration for the backend application.

# Create backend configuration file
cat > backend-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: multi-tier-app
data:
  DB_HOST: "database-service"
  DB_PORT: "3306"
  DB_NAME: "webapp_db"
  DB_USER: "webapp_user"
  APP_PORT: "5000"
  APP_ENV: "production"
EOF

# Apply the backend ConfigMap
kubectl apply -f backend-config.yaml

# Verify ConfigMap creation
kubectl describe configmap backend-config
Subtask 3.2: Deploy Backend Application
Create a simple backend application deployment.

# Create backend deployment file
cat > backend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: multi-tier-app
  labels:
    app: backend
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: backend-app
        image: python:3.9-slim
        ports:
        - containerPort: 5000
          name: http
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: MYSQL_PASSWORD
        - name: APP_PORT
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: APP_PORT
        command: ["/bin/sh"]
        args: ["-c", "pip install flask mysql-connector-python && python -c \"
from flask import Flask, jsonify
import mysql.connector
import os
import time

app = Flask(__name__)

def get_db_connection():
    max_retries = 5
    for i in range(max_retries):
        try:
            connection = mysql.connector.connect(
                host=os.environ['DB_HOST'],
                port=int(os.environ['DB_PORT']),
                database=os.environ['DB_NAME'],
                user=os.environ['DB_USER'],
                password=os.environ['DB_PASSWORD']
            )
            return connection
        except Exception as e:
            print(f'Database connection attempt {i+1} failed: {e}')
            time.sleep(5)
    return None

@app.route('/api/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'backend'})

@app.route('/api/data')
def get_data():
    try:
        conn = get_db_connection()
        if conn:
            cursor = conn.cursor()
            cursor.execute('SELECT VERSION()')
            version = cursor.fetchone()
            conn.close()
            return jsonify({'message': 'Backend connected to database', 'db_version': version[0]})
        else:
            return jsonify({'error': 'Database connection failed'}), 500
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ['APP_PORT']))
\""]
        readinessProbe:
          httpGet:
            path: /api/health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/health
            port: 5000
          initialDelaySeconds: 60
          periodSeconds: 30
EOF

# Apply the backend deployment
kubectl apply -f backend-deployment.yaml

# Check deployment status
kubectl get deployments
kubectl get pods -l tier=backend
Subtask 3.3: Create Backend Service
Create a Service to expose the backend application.

# Create backend service file
cat > backend-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: multi-tier-app
  labels:
    app: backend
    tier: backend
spec:
  selector:
    app: backend
    tier: backend
  ports:
  - port: 5000
    targetPort: 5000
    protocol: TCP
    name: http
  type: ClusterIP
EOF

# Apply the backend service
kubectl apply -f backend-service.yaml

# Verify service creation
kubectl get services
kubectl describe service backend-service
Task 4: Deploy the Frontend Tier
Subtask 4.1: Create Frontend ConfigMap
Create configuration for the frontend application.

# Create frontend configuration file
cat > frontend-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: multi-tier-app
data:
  BACKEND_URL: "http://backend-service:5000"
  APP_TITLE: "Multi-Tier Kubernetes Application"
  APP_PORT: "80"
EOF

# Apply the frontend ConfigMap
kubectl apply -f frontend-config.yaml

# Verify ConfigMap creation
kubectl describe configmap frontend-config
Subtask 4.2: Deploy Frontend Application
Create a frontend application deployment.

# Create frontend deployment file
cat > frontend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: multi-tier-app
  labels:
    app: frontend
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: frontend-app
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: http
        env:
        - name: BACKEND_URL
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: BACKEND_URL
        - name: APP_TITLE
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: APP_TITLE
        volumeMounts:
        - name: frontend-content
          mountPath: /usr/share/nginx/html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: frontend-content
        configMap:
          name: frontend-html
      - name: nginx-config
        configMap:
          name: nginx-config
EOF

# Create HTML content ConfigMap
cat > frontend-html-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: multi-tier-app
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Multi-Tier Kubernetes App</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; background-color: #f5f5f5; }
            .container { max-width: 800px; margin: 0 auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
            .header { text-align: center; color: #333; margin-bottom: 30px; }
            .tier { margin: 20px 0; padding: 15px; border-left: 4px solid #007acc; background: #f9f9f9; }
            .button { background: #007acc; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; margin: 5px; }
            .button:hover { background: #005a99; }
            .result { margin: 10px 0; padding: 10px; background: #e8f4f8; border-radius: 4px; }
            .error { background: #ffe6e6; color: #cc0000; }
            .success { background: #e6ffe6; color: #006600; }
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <h1>Multi-Tier Kubernetes Application</h1>
                <p>Frontend → Backend → Database Communication Demo</p>
            </div>
            
            <div class="tier">
                <h3>🌐 Frontend Tier (Nginx)</h3>
                <p>This HTML page is served by an Nginx container running in the frontend pod.</p>
                <p><strong>Status:</strong> <span style="color: green;">Active</span></p>
            </div>
            
            <div class="tier">
                <h3>⚙️ Backend Tier (Python Flask)</h3>
                <p>Click the buttons below to test communication with the backend service.</p>
                <button class="button" onclick="testBackendHealth()">Test Backend Health</button>
                <button class="button" onclick="testBackendData()">Test Database Connection</button>
                <div id="backend-result" class="result" style="display: none;"></div>
            </div>
            
            <div class="tier">
                <h3>🗄️ Database Tier (MySQL)</h3>
                <p>MySQL database running in a separate pod, accessible via backend service.</p>
                <p><strong>Connection:</strong> Via backend service only (not directly accessible)</p>
            </div>
            
            <div class="tier">
                <h3>📋 Architecture Overview</h3>
                <ul>
                    <li><strong>Frontend:</strong> Nginx serving static content (this page)</li>
                    <li><strong>Backend:</strong> Python Flask API with health and data endpoints</li>
                    <li><strong>Database:</strong> MySQL database with persistent storage</li>
                    <li><strong>Communication:</strong> Services enable inter-pod communication</li>
                    <li><strong>Configuration:</strong> ConfigMaps store application settings</li>
                </ul>
            </div>
        </div>
        
        <script>
            async function testBackendHealth() {
                const resultDiv = document.getElementById('backend-result');
                resultDiv.style.display = 'block';
                resultDiv.innerHTML = 'Testing backend health...';
                resultDiv.className = 'result';
                
                try {
                    const response = await fetch('/api/health');
                    const data = await response.json();
                    resultDiv.innerHTML = `✅ Backend Health: ${JSON.stringify(data, null, 2)}`;
                    resultDiv.className = 'result success';
                } catch (error) {
                    resultDiv.innerHTML = `❌ Backend Health Check Failed: ${error.message}`;
                    resultDiv.className = 'result error';
                }
            }
            
            async function testBackendData() {
                const resultDiv = document.getElementById('backend-result');
                resultDiv.style.display = 'block';
                resultDiv.innerHTML = 'Testing database connection...';
                resultDiv.className = 'result';
                
                try {
                    const response = await fetch('/api/data');
                    const data = await response.json();
                    resultDiv.innerHTML = `✅ Database Connection: ${JSON.stringify(data, null, 2)}`;
                    resultDiv.className = 'result success';
                } catch (error) {
                    resultDiv.innerHTML = `❌ Database Connection Failed: ${error.message}`;
                    resultDiv.className = 'result error';
                }
            }
        </script>
    </body>
    </html>
EOF

# Create Nginx configuration ConfigMap
cat > nginx-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: multi-tier-app
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /api/ {
            proxy_pass http://backend-service:5000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
EOF

# Apply all frontend configurations
kubectl apply -f frontend-html-config.yaml
kubectl apply -f nginx-config.yaml
kubectl apply -f frontend-deployment.yaml

# Check deployment status
kubectl get deployments
kubectl get pods -l tier=frontend
Subtask 4.3: Create Frontend Service
Create a Service to expose the frontend application.

# Create frontend service file
cat > frontend-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: multi-tier-app
  labels:
    app: frontend
    tier: frontend
spec:
  selector:
    app: frontend
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  type: NodePort
EOF

# Apply the frontend service
kubectl apply -f frontend-service.yaml

# Get service details including NodePort
kubectl get services
kubectl describe service frontend-service
Task 5: Verify Multi-Tier Application Communication
Subtask 5.1: Check All Deployments and Services
Verify that all components are running correctly.

# Check all deployments
kubectl get deployments

# Check all pods
kubectl get pods

# Check all services
kubectl get services

# Check all configmaps
kubectl get configmaps

# Check all secrets
kubectl get secrets
Subtask 5.2: Test Inter-Service Communication
Test communication between the tiers.

# Get a frontend pod name
FRONTEND_POD=$(kubectl get pods -l tier=frontend -o jsonpath='{.items[0].metadata.name}')

# Test frontend to backend communication
kubectl exec -it $FRONTEND_POD -- curl -s http://backend-service:5000/api/health

# Test backend to database communication
BACKEND_POD=$(kubectl get pods -l tier=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $BACKEND_POD -- curl -s http://localhost:5000/api/data

# Check database connectivity from backend
kubectl exec -it $BACKEND_POD -- curl -s http://localhost:5000/api/health
Subtask 5.3: Access the Application
Get the external access information for your application.

# Get node IP and frontend service port
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
if [ -z "$NODE_IP" ]; then
    NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
fi

FRONTEND_PORT=$(kubectl get service frontend-service -o jsonpath='{.spec.ports[0].nodePort}')

echo "Application URL: http://$NODE_IP:$FRONTEND_PORT"
echo "Backend Health Check: http://$NODE_IP:$FRONTEND_PORT/api/health"
echo "Backend Data Endpoint: http://$NODE_IP:$FRONTEND_PORT/api/data"
Task 6: Monitor and Troubleshoot
Subtask 6.1: Monitor Application Logs
Check logs from each tier to ensure proper operation.

# Check frontend logs
kubectl logs -l tier=frontend --tail=20

# Check backend logs
kubectl logs -l tier=backend --tail=20

# Check database logs
kubectl logs -l tier=database --tail=20
Subtask 6.2: Verify ConfigMap Usage
Confirm that ConfigMaps are being used correctly by the applications.

# Check environment variables in backend pod
BACKEND_POD=$(kubectl get pods -l tier=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $BACKEND_POD -- env | grep -E "(DB_|APP_)"

# Check ConfigMap data
kubectl get configmap database-config -o yaml
kubectl get configmap backend-config -o yaml
kubectl get configmap frontend-config -o yaml
Subtask 6.3: Test Application Scaling
Test the scalability of your multi-tier application.

# Scale frontend deployment
kubectl scale deployment frontend-deployment --replicas=5

# Scale backend deployment
kubectl scale deployment backend-deployment --replicas=3

# Check scaling results
kubectl get deployments
kubectl get pods

# Scale back to original size
kubectl scale deployment frontend-deployment --replicas=3
kubectl scale deployment backend-deployment --replicas=2
Troubleshooting Common Issues
Database Connection Issues
If the backend cannot connect to the database:

# Check database pod status
kubectl describe pod -l tier=database

# Verify database service endpoints
kubectl get endpoints database-service

# Test database connectivity from backend pod
BACKEND_POD=$(kubectl get pods -l tier=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $BACKEND_POD -- nc -zv database-service 3306
Service Discovery Problems
If services cannot find each other:

# Check DNS resolution
kubectl exec -it $BACKEND_POD -- nslookup database-service
kubectl exec -it $FRONTEND_POD -- nslookup backend-service

# Verify service selectors match pod labels
kubectl describe service database-service
kubectl describe service backend-service
ConfigMap Issues
If configuration is not being applied:

# Verify ConfigMap mounting
kubectl describe pod $BACKEND_POD | grep -A 10 "Mounts:"

# Check environment variables
kubectl exec -it $BACKEND_POD -- printenv | grep DB_
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Delete all resources in the namespace
kubectl delete namespace multi-tier-app

# Verify cleanup
kubectl get namespaces
Conclusion
Congratulations! You have successfully deployed a complete multi-tier application in Kubernetes. In this lab, you accomplished the following:

Key Achievements:

• Multi-Tier Architecture: Deployed a three-tier application with separate frontend (Nginx), backend (Python Flask), and database (MySQL) components, demonstrating proper separation of concerns in microservices architecture.

• Pod Management: Created and managed multiple Pods across different tiers, understanding how containerized applications run in Kubernetes environments.

• Service Communication: Implemented Kubernetes Services to enable secure and reliable communication between application tiers, showcasing service discovery and internal networking.

• Configuration Management: Utilized ConfigMaps to externalize application configuration, making your applications more flexible and environment-agnostic.

• Security Best Practices: Implemented Secrets for sensitive data like database passwords, separating configuration from sensitive information.

• Application Scaling: Demonstrated horizontal scaling capabilities by running multiple replicas of frontend and backend services.

Why This Matters:

This lab represents real-world application deployment patterns used by organizations worldwide. Multi-tier architectures are fundamental to modern cloud-native applications because they provide:

Scalability: Each tier can be scaled independently based on demand
Maintainability: Changes to one tier don't affect others
Reliability: Failure in one component doesn't bring down the entire application
Security: Database access is restricted and controlled through the backend tier
The skills you've developed here are directly applicable to the Kubernetes and Cloud Native Associate (KCNA) certification and are essential for anyone working with containerized applications in production environments. You now understand how to deploy, configure, and manage complex applications in Kubernetes, which is a critical skill for modern DevOps and cloud engineering roles.

Next Steps:

Consider exploring advanced topics like persistent volumes for database storage, ingress controllers for external access, and monitoring solutions to further enhance your Kubernetes expertise.
