Lab 7: Secure Application Configuration
Objectives
By the end of this lab, you will be able to:

• Create and configure ConfigMaps for storing non-sensitive application configuration data • Create and manage Secrets for handling sensitive information like API keys and passwords • Mount ConfigMaps as volumes in Kubernetes pods • Inject Secrets as environment variables into applications • Verify that applications can correctly access both ConfigMaps and Secrets • Understand the security implications and best practices for application configuration in Kubernetes • Demonstrate proficiency in secure configuration management for containerized applications

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, deployments, services) • Familiarity with YAML syntax and structure • Knowledge of Linux command-line operations • Understanding of environment variables and file systems • Basic Docker and containerization concepts • Access to kubectl command-line tool

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes cluster already set up. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Kubernetes cluster (single-node or multi-node) • kubectl command-line tool pre-installed and configured • Docker runtime environment • Text editor (nano, vim, or code editor of choice)

Task 1: Create a ConfigMap for Non-Sensitive Application Settings
Subtask 1.1: Understanding ConfigMaps
ConfigMaps are Kubernetes objects that store configuration data in key-value pairs. They are designed for non-sensitive information that your applications need to function properly.

Common use cases for ConfigMaps: • Application configuration files • Command-line arguments • Environment variables for non-sensitive data • Configuration parameters

Subtask 1.2: Create a ConfigMap Using kubectl
First, let's create a simple ConfigMap with application settings:

# Create a ConfigMap with literal values
kubectl create configmap app-config \
  --from-literal=database_host=mysql-service \
  --from-literal=database_port=3306 \
  --from-literal=app_mode=production \
  --from-literal=log_level=info \
  --from-literal=max_connections=100
Verify the ConfigMap was created successfully:

# List all ConfigMaps
kubectl get configmaps

# View detailed information about our ConfigMap
kubectl describe configmap app-config

# View the ConfigMap in YAML format
kubectl get configmap app-config -o yaml
Subtask 1.3: Create a ConfigMap from a Configuration File
Create a configuration file that our application will use:

# Create a directory for our lab files
mkdir -p ~/lab7-secure-config
cd ~/lab7-secure-config

# Create an application configuration file
cat > app.properties << EOF
# Database Configuration
database.host=mysql-service
database.port=3306
database.name=myapp

# Application Settings
app.name=SecureApp
app.version=1.0.0
app.environment=production

# Logging Configuration
logging.level=INFO
logging.file=/var/log/app.log

# Performance Settings
max.connections=100
timeout.seconds=30
EOF
Create a ConfigMap from this file:

# Create ConfigMap from file
kubectl create configmap app-properties --from-file=app.properties

# Verify the file-based ConfigMap
kubectl describe configmap app-properties
Subtask 1.4: Create a ConfigMap Using YAML Manifest
Create a YAML manifest for more complex configuration:

cat > configmap-manifest.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-yaml
  labels:
    app: secure-demo
    component: configuration
data:
  # Simple key-value pairs
  database_host: "mysql-service"
  database_port: "3306"
  app_mode: "production"
  
  # Multi-line configuration file
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /api {
            proxy_pass http://backend-service:8080;
        }
    }
  
  # JSON configuration
  app-config.json: |
    {
      "database": {
        "host": "mysql-service",
        "port": 3306,
        "ssl": true
      },
      "cache": {
        "enabled": true,
        "ttl": 300
      },
      "features": {
        "authentication": true,
        "logging": true
      }
    }
EOF
Apply the ConfigMap manifest:

# Apply the ConfigMap
kubectl apply -f configmap-manifest.yaml

# Verify all ConfigMaps
kubectl get configmaps
Task 2: Create a Secret for Sensitive Data
Subtask 2.1: Understanding Kubernetes Secrets
Secrets are Kubernetes objects designed to store sensitive information such as: • Passwords and API keys • OAuth tokens • SSH keys • TLS certificates

Secrets are base64 encoded (not encrypted) and should be used with additional security measures in production.

Subtask 2.2: Create a Secret Using kubectl
Create a Secret for sensitive application data:

# Create a Secret with literal values
kubectl create secret generic app-secrets \
  --from-literal=database_password=MySecurePassword123 \
  --from-literal=api_key=sk-1234567890abcdef \
  --from-literal=jwt_secret=super-secret-jwt-key-2024 \
  --from-literal=redis_password=RedisPass456
Verify the Secret was created:

# List all Secrets
kubectl get secrets

# View Secret details (note that values are not shown)
kubectl describe secret app-secrets

# View Secret in YAML format (values are base64 encoded)
kubectl get secret app-secrets -o yaml
Subtask 2.3: Create a Secret from Files
Create files containing sensitive data:

# Create sensitive configuration files
echo -n "MySecurePassword123" > db-password.txt
echo -n "sk-1234567890abcdef" > api-key.txt

# Create a certificate file (simulated)
cat > tls.crt << EOF
-----BEGIN CERTIFICATE-----
MIICljCCAX4CCQDAOxqozVrCFDANBgkqhkiG9w0BAQsFADCBjDELMAkGA1UEBhMC
VVMxCzAJBgNVBAgMAkNBMRYwFAYDVQQHDA1TYW4gRnJhbmNpc2NvMRMwEQYDVQQK
DApNeUNvbXBhbnkxEzARBgNVBAsMCk15RGl2aXNpb24xEjAQBgNVBAMMCWxvY2Fs
aG9zdDEaMBgGCSqGSIb3DQEJARYLdGVzdEB0ZXN0LmNvbTAeFw0yNDAxMDEwMDAw
MDBaFw0yNTAxMDEwMDAwMDBaMIGMMQswCQYDVQQGEwJVUzELMAkGA1UECAwCQ0Ex
FjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xEzARBgNVBAoMCk15Q29tcGFueTETMBEG
A1UECwwKTXlEaXZpc2lvbjESMBAGA1UEAwwJbG9jYWxob3N0MRowGAYJKoZIhvcN
AQkBFgt0ZXN0QHRlc3QuY29tMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC7
-----END CERTIFICATE-----
EOF

# Create Secret from files
kubectl create secret generic file-secrets \
  --from-file=database-password=db-password.txt \
  --from-file=api-key=api-key.txt \
  --from-file=tls.crt=tls.crt

# Clean up sensitive files
rm db-password.txt api-key.txt
Subtask 2.4: Create a Secret Using YAML Manifest
Create a YAML manifest for Secrets (values must be base64 encoded):

# Encode values to base64
echo -n "admin123" | base64  # Output: YWRtaW4xMjM=
echo -n "secret-api-key-xyz" | base64  # Output: c2VjcmV0LWFwaS1rZXkteHl6

# Create Secret manifest
cat > secret-manifest.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets-yaml
  labels:
    app: secure-demo
    component: secrets
type: Opaque
data:
  # Base64 encoded values
  admin_password: YWRtaW4xMjM=
  api_key: c2VjcmV0LWFwaS1rZXkteHl6
  database_url: bXlzcWw6Ly91c2VyOnBhc3NAZGItc2VydmVyOjMzMDYvbXlkYg==
stringData:
  # Plain text values (automatically base64 encoded)
  smtp_password: "email-password-123"
  oauth_client_secret: "oauth-secret-456"
EOF
Apply the Secret manifest:

# Apply the Secret
kubectl apply -f secret-manifest.yaml

# Verify all Secrets
kubectl get secrets
Task 3: Create an Application Pod with ConfigMap and Secret
Subtask 3.1: Create a Deployment with ConfigMap Volume Mount
Create a deployment that mounts the ConfigMap as a volume:

cat > app-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  labels:
    app: secure-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      containers:
      - name: app-container
        image: nginx:1.21
        ports:
        - containerPort: 80
        
        # Environment variables from Secret
        env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database_password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api_key
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt_secret
        
        # Environment variables from ConfigMap
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_host
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_port
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app_mode
        
        # Volume mounts
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
        - name: properties-volume
          mountPath: /etc/app
          readOnly: true
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
      
      # Volumes
      volumes:
      - name: config-volume
        configMap:
          name: app-config-yaml
      - name: properties-volume
        configMap:
          name: app-properties
      - name: secret-volume
        secret:
          secretName: file-secrets
          defaultMode: 0400  # Read-only for owner only
EOF
Deploy the application:

# Apply the deployment
kubectl apply -f app-deployment.yaml

# Check deployment status
kubectl get deployments
kubectl get pods

# Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=secure-app --timeout=60s
Subtask 3.2: Create a Debug Pod for Testing
Create a simple debug pod to test our configuration:

cat > debug-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  labels:
    app: debug
spec:
  containers:
  - name: debug-container
    image: busybox:1.35
    command: ['sleep', '3600']
    
    # Environment variables from ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
    
    # Environment variables from Secret
    env:
    - name: SECRET_API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api_key
    - name: SECRET_DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database_password
    
    # Volume mounts
    volumeMounts:
    - name: config-files
      mountPath: /config
    - name: secret-files
      mountPath: /secrets
      readOnly: true
  
  volumes:
  - name: config-files
    configMap:
      name: app-config-yaml
  - name: secret-files
    secret:
      secretName: app-secrets-yaml
EOF
Deploy the debug pod:

# Apply the debug pod
kubectl apply -f debug-pod.yaml

# Wait for pod to be ready
kubectl wait --for=condition=ready pod debug-pod --timeout=60s
Task 4: Verify Application Access to ConfigMap and Secret
Subtask 4.1: Verify ConfigMap Access
Test that the application can access ConfigMap data:

# Get pod name
POD_NAME=$(kubectl get pods -l app=secure-app -o jsonpath='{.items[0].metadata.name}')

# Check environment variables from ConfigMap
kubectl exec $POD_NAME -- env | grep -E "(DATABASE_HOST|DATABASE_PORT|APP_MODE)"

# Check mounted ConfigMap files
kubectl exec $POD_NAME -- ls -la /etc/config/
kubectl exec $POD_NAME -- cat /etc/config/nginx.conf
kubectl exec $POD_NAME -- cat /etc/config/app-config.json

# Check properties file
kubectl exec $POD_NAME -- ls -la /etc/app/
kubectl exec $POD_NAME -- cat /etc/app/app.properties
Subtask 4.2: Verify Secret Access
Test that the application can access Secret data:

# Check environment variables from Secret (be careful with sensitive data)
kubectl exec $POD_NAME -- sh -c 'echo "API_KEY length: ${#API_KEY}"'
kubectl exec $POD_NAME -- sh -c 'echo "DATABASE_PASSWORD is set: $([ -n "$DATABASE_PASSWORD" ] && echo "YES" || echo "NO")"'

# Check mounted Secret files
kubectl exec $POD_NAME -- ls -la /etc/secrets/
kubectl exec $POD_NAME -- wc -c /etc/secrets/api-key
kubectl exec $POD_NAME -- file /etc/secrets/tls.crt
Subtask 4.3: Comprehensive Testing with Debug Pod
Use the debug pod for more detailed testing:

# Execute interactive shell in debug pod
kubectl exec -it debug-pod -- sh

# Inside the pod, run these commands:
Once inside the debug pod, execute these commands:

# Check all environment variables
env | sort

# Check ConfigMap environment variables
env | grep -E "(database_|app_|log_|max_)"

# Check Secret environment variables
echo "API Key length: ${#SECRET_API_KEY}"
echo "DB Password is set: $([ -n "$SECRET_DB_PASSWORD" ] && echo "YES" || echo "NO")"

# Check mounted ConfigMap files
ls -la /config/
cat /config/app-config.json | head -10

# Check mounted Secret files
ls -la /secrets/
wc -c /secrets/*

# Exit the pod
exit
Subtask 4.4: Verify Security and Permissions
Check the security aspects of our configuration:

# Check Secret permissions in the pod
kubectl exec $POD_NAME -- ls -la /etc/secrets/

# Verify that Secrets are not visible in pod description
kubectl describe pod $POD_NAME | grep -A 10 -B 10 -i secret

# Check ConfigMap visibility (should be visible)
kubectl describe pod $POD_NAME | grep -A 5 -B 5 -i configmap

# Verify base64 encoding of Secrets
kubectl get secret app-secrets -o jsonpath='{.data.api_key}' | base64 -d
echo  # Add newline
Task 5: Advanced Configuration Scenarios
Subtask 5.1: Update ConfigMap and Observe Changes
Demonstrate how to update configuration:

# Update the ConfigMap
kubectl patch configmap app-config --patch '{"data":{"log_level":"debug","max_connections":"200"}}'

# Check the updated ConfigMap
kubectl get configmap app-config -o yaml

# Note: Environment variables won't update automatically
# But mounted volumes will update (may take up to 1 minute)

# Check if mounted files are updated
kubectl exec $POD_NAME -- cat /etc/config/app-config.json
Subtask 5.2: Create a Rolling Update with New Configuration
Create a new version of the deployment with updated configuration:

# Create a new ConfigMap version
kubectl create configmap app-config-v2 \
  --from-literal=database_host=mysql-service-v2 \
  --from-literal=database_port=3306 \
  --from-literal=app_mode=staging \
  --from-literal=log_level=debug \
  --from-literal=max_connections=150 \
  --from-literal=app_version=2.0.0

# Update deployment to use new ConfigMap
kubectl patch deployment secure-app --patch '
spec:
  template:
    spec:
      containers:
      - name: app-container
        env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config-v2
              key: database_host
        - name: APP_VERSION
          valueFrom:
            configMapKeyRef:
              name: app-config-v2
              key: app_version'

# Watch the rolling update
kubectl rollout status deployment/secure-app

# Verify new configuration
NEW_POD_NAME=$(kubectl get pods -l app=secure-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD_NAME -- env | grep -E "(DATABASE_HOST|APP_VERSION)"
Subtask 5.3: Implement Configuration Validation
Create a pod with init container for configuration validation:

cat > validated-app.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: validated-app
  labels:
    app: validated-app
spec:
  initContainers:
  - name: config-validator
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      echo "Validating configuration..."
      
      # Check required environment variables
      if [ -z "$DATABASE_HOST" ]; then
        echo "ERROR: DATABASE_HOST not set"
        exit 1
      fi
      
      if [ -z "$API_KEY" ]; then
        echo "ERROR: API_KEY not set"
        exit 1
      fi
      
      # Validate configuration files
      if [ ! -f /config/app-config.json ]; then
        echo "ERROR: app-config.json not found"
        exit 1
      fi
      
      # Simple JSON validation
      if ! grep -q "database" /config/app-config.json; then
        echo "ERROR: Invalid configuration format"
        exit 1
      fi
      
      echo "Configuration validation passed!"
    
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api_key
    
    volumeMounts:
    - name: config-volume
      mountPath: /config
  
  containers:
  - name: main-app
    image: nginx:1.21
    ports:
    - containerPort: 80
    
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api_key
    
    volumeMounts:
    - name: config-volume
      mountPath: /config
  
  volumes:
  - name: config-volume
    configMap:
      name: app-config-yaml
EOF
Deploy and test the validated application:

# Apply the validated app
kubectl apply -f validated-app.yaml

# Check init container logs
kubectl logs validated-app -c config-validator

# Check if main container started successfully
kubectl get pod validated-app
kubectl logs validated-app -c main-app
Task 6: Troubleshooting and Best Practices
Subtask 6.1: Common Issues and Solutions
Test common configuration issues:

# Create a pod with missing Secret reference
cat > broken-app.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ['sleep', '3600']
    env:
    - name: MISSING_SECRET
      valueFrom:
        secretKeyRef:
          name: non-existent-secret
          key: some-key
EOF

# Apply and observe the error
kubectl apply -f broken-app.yaml
kubectl describe pod broken-app

# Check events for error details
kubectl get events --sort-by=.metadata.creationTimestamp | tail -5

# Clean up broken pod
kubectl delete pod broken-app
Subtask 6.2: Security Best Practices Demonstration
Implement security best practices:

# Create a Secret with proper RBAC
cat > secure-secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: production-secrets
  annotations:
    kubernetes.io/description: "Production secrets - restricted access"
type: Opaque
stringData:
  database_password: "ProductionPassword123!"
  api_key: "prod-api-key-xyz789"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-app-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["production-secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
subjects:
- kind: ServiceAccount
  name: secure-app-sa
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Apply security configuration
kubectl apply -f secure-secret.yaml

# Create a pod using the service account
cat > secure-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: secure-app-sa
  containers:
  - name: app
    image: busybox:1.35
    command: ['sleep', '3600']
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: production-secrets
          key: database_password
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
EOF

kubectl apply -f secure-pod.yaml
kubectl get pod secure-pod
Task 7: Cleanup and Resource Management
Subtask 7.1: Clean Up Lab Resources
Remove all resources created during the lab:

# Delete pods
kubectl delete pod debug-pod validated-app secure-pod broken-app --ignore-not-found=true

# Delete deployments
kubectl delete deployment secure-app --ignore-not-found=true

# Delete ConfigMaps
kubectl delete configmap app-config app-properties app-config-yaml app-config-v2 --ignore-not-found=true

# Delete Secrets
kubectl delete secret app-secrets file-secrets app-secrets-yaml production-secrets --ignore-not-found=true

# Delete RBAC resources
kubectl delete rolebinding secret-reader-binding --ignore-not-found=true
kubectl delete role secret-reader --ignore-not-found=true
kubectl delete serviceaccount secure-app-sa --ignore-not-found=true

# Clean up local files
cd ~
rm -rf ~/lab7-secure-config

# Verify cleanup
kubectl get configmaps,secrets,pods,deployments
Subtask 7.2: Verify Complete Cleanup
Ensure all lab resources have been removed:

# Check for any remaining lab resources
kubectl get all -l app=secure-app
kubectl get all -l app=debug
kubectl get all -l app=validated-app

# Check ConfigMaps and Secrets
kubectl get configmaps | grep -E "(app-config|app-properties)"
kubectl get secrets | grep -E "(app-secrets|file-secrets|production-secrets)"

echo "Lab cleanup completed successfully!"
Troubleshooting Guide
Common Issues and Solutions
Issue 1: Pod fails to start with Secret reference error

# Solution: Check if Secret exists and has correct key names
kubectl get secrets
kubectl describe secret <secret-name>
Issue 2: ConfigMap changes not reflected in pod

# Solution: ConfigMap environment variables require pod restart
kubectl rollout restart deployment/<deployment-name>
Issue 3: Permission denied accessing mounted secrets

# Solution: Check file permissions and security context
kubectl exec <pod-name> -- ls -la /path/to/secrets
Issue 4: Base64 encoding issues

# Solution: Ensure proper encoding without newlines
echo -n "your-secret" | base64
Conclusion
Congratulations! You have successfully completed Lab 7: Secure Application Configuration. In this comprehensive lab, you have accomplished the following:

Key Achievements
• ConfigMap Mastery: You learned to create and manage ConfigMaps using multiple methods (kubectl commands, file imports, and YAML manifests) for storing non-sensitive application configuration data

• Secret Management: You gained hands-on experience creating and securing Secrets for sensitive information like API keys, passwords, and certificates using various approaches

• Volume Mounting: You successfully mounted ConfigMaps as volumes in pods, allowing applications to access configuration files directly from the filesystem

• Environment Variable Injection: You demonstrated how to inject both ConfigMap and Secret data as environment variables, providing flexible configuration options for applications

• Security Best Practices: You implemented proper security measures including RBAC, service accounts, and appropriate file permissions for sensitive data

• Configuration Validation: You created init containers to validate configuration before application startup, ensuring robust deployment practices

• Troubleshooting Skills: You learned to identify and resolve common configuration issues, preparing you for real-world scenarios

Why This Matters
Secure application configuration is crucial in modern containerized environments because:

• Security: Proper separation of sensitive and non-sensitive data protects your applications from security breaches • Flexibility: ConfigMaps and Secrets allow you to modify application behavior without rebuilding container images • Compliance: Following security best practices helps meet regulatory requirements and industry standards • Maintainability: Centralized configuration management simplifies application updates and environment-specific deployments • Scalability: These patterns support multi-environment deployments (development, staging, production) with different configurations

Next Steps
To further enhance your Kubernetes security and configuration management skills:

• Explore Helm charts for templated configuration management • Learn about External Secrets Operator for integration with external secret management systems • Study Pod Security Standards and Network Policies for comprehensive security • Practice with Kustomize for configuration customization across environments • Investigate Vault or other external secret management solutions for enterprise environments

This lab has provided you with essential skills for the Certified Kubernetes Application Developer (CKAD) certification, particularly in the areas of application configuration, security, and resource management. The hands-on experience you've gained will be invaluable in real-world Kubernetes deployments where secure configuration management is critical for application succes
