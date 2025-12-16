Lab 13: Using ConfigMaps and Secrets for Application Configuration
Objectives
By the end of this lab, you will be able to:

• Create and manage ConfigMaps to store non-sensitive configuration data • Create and manage Secrets to store sensitive information securely • Pass environment variables to Pods using ConfigMaps and Secrets • Mount ConfigMaps and Secrets as files in Pod containers • Understand the differences between ConfigMaps and Secrets • Apply best practices for application configuration management in Kubernetes

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments) • Familiarity with YAML syntax • Basic command-line interface experience • Understanding of environment variables and file systems

Ready-to-Use Cloud Machines
Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install additional software.

Your lab environment includes: • Kubernetes cluster (minikube or similar) • kubectl command-line tool • Text editor (nano/vim) • All necessary permissions to create resources

Lab Tasks
Task 1: Create a ConfigMap to Pass Environment Variables to a Pod
Subtask 1.1: Create a ConfigMap Using kubectl
First, let's create a ConfigMap that contains application configuration data.

Create a ConfigMap with literal values:
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=mysql-service \
  --from-literal=DATABASE_PORT=3306 \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info
Verify the ConfigMap was created:
kubectl get configmaps
View the ConfigMap details:
kubectl describe configmap app-config
Subtask 1.2: Create a ConfigMap from a File
Create a configuration file:
cat > app.properties << EOF
database.host=mysql-service
database.port=3306
app.environment=production
log.level=info
cache.enabled=true
cache.ttl=300
EOF
Create a ConfigMap from the file:
kubectl create configmap app-properties --from-file=app.properties
Verify the file-based ConfigMap:
kubectl get configmap app-properties -o yaml
Subtask 1.3: Create a Pod Using ConfigMap as Environment Variables
Create a Pod manifest that uses the ConfigMap:
cat > pod-with-configmap.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-pod-env
  labels:
    app: demo-app
spec:
  containers:
  - name: app-container
    image: nginx:1.21
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: CUSTOM_MESSAGE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
  restartPolicy: Never
EOF
Apply the Pod manifest:
kubectl apply -f pod-with-configmap.yaml
Verify the Pod is running:
kubectl get pods
Check the environment variables inside the Pod:
kubectl exec app-pod-env -- env | grep -E "(DATABASE|APP|LOG)"
Task 2: Create a Secret to Store Sensitive Information
Subtask 2.1: Create a Secret for Database Credentials
Create a Secret using kubectl:
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret123 \
  --from-literal=root-password=rootsecret456
Verify the Secret was created:
kubectl get secrets
View the Secret details (note that values are base64 encoded):
kubectl describe secret db-credentials
View the Secret in YAML format:
kubectl get secret db-credentials -o yaml
Subtask 2.2: Create a Secret from Files
Create credential files:
echo -n 'admin' > username.txt
echo -n 'supersecret123' > password.txt
Create a Secret from files:
kubectl create secret generic file-credentials --from-file=username.txt --from-file=password.txt
Clean up the credential files:
rm username.txt password.txt
Subtask 2.3: Create a Pod Using Secrets as Environment Variables
Create a Pod manifest that uses the Secret:
cat > pod-with-secret.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-pod-secret
  labels:
    app: demo-app
spec:
  containers:
  - name: app-container
    image: nginx:1.21
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: DB_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: root-password
  restartPolicy: Never
EOF
Apply the Pod manifest:
kubectl apply -f pod-with-secret.yaml
Verify the environment variables (be careful with sensitive data):
kubectl exec app-pod-secret -- env | grep DB_
Task 3: Mount Both ConfigMaps and Secrets into Pods as Files
Subtask 3.1: Create a Pod with ConfigMap and Secret as Volume Mounts
Create a comprehensive Pod manifest:
cat > pod-with-volumes.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-pod-volumes
  labels:
    app: demo-app
spec:
  containers:
  - name: app-container
    image: nginx:1.21
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
    - name: properties-volume
      mountPath: /etc/properties
      readOnly: true
    env:
    - name: CONFIG_PATH
      value: "/etc/config"
    - name: SECRETS_PATH
      value: "/etc/secrets"
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: db-credentials
      defaultMode: 0400
  - name: properties-volume
    configMap:
      name: app-properties
  restartPolicy: Never
EOF
Apply the Pod manifest:
kubectl apply -f pod-with-volumes.yaml
Wait for the Pod to be ready:
kubectl wait --for=condition=Ready pod/app-pod-volumes --timeout=60s
Subtask 3.2: Verify Mounted Files
Check the ConfigMap files:
kubectl exec app-pod-volumes -- ls -la /etc/config/
View ConfigMap content:
kubectl exec app-pod-volumes -- cat /etc/config/DATABASE_HOST
kubectl exec app-pod-volumes -- cat /etc/config/APP_ENV
Check the Secret files (note the restricted permissions):
kubectl exec app-pod-volumes -- ls -la /etc/secrets/
View Secret content:
kubectl exec app-pod-volumes -- cat /etc/secrets/username
Check the properties file:
kubectl exec app-pod-volumes -- ls -la /etc/properties/
kubectl exec app-pod-volumes -- cat /etc/properties/app.properties
Subtask 3.3: Create a Deployment Using ConfigMaps and Secrets
Create a Deployment manifest:
cat > deployment-with-config.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-deployment
  labels:
    app: web-app
spec:
  replicas: 2
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
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/nginx/html/config
          readOnly: true
        - name: secret-volume
          mountPath: /usr/share/nginx/html/secrets
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: secret-volume
        secret:
          secretName: db-credentials
          defaultMode: 0400
EOF
Apply the Deployment:
kubectl apply -f deployment-with-config.yaml
Verify the Deployment:
kubectl get deployments
kubectl get pods -l app=web-app
Test configuration in one of the Deployment pods:
POD_NAME=$(kubectl get pods -l app=web-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- env | grep -E "(DATABASE|APP|LOG|DB_)"
Task 4: Update ConfigMaps and Secrets
Subtask 4.1: Update a ConfigMap
Update the ConfigMap:
kubectl patch configmap app-config --patch '{"data":{"LOG_LEVEL":"debug","NEW_FEATURE":"enabled"}}'
Verify the update:
kubectl describe configmap app-config
Check if mounted files are updated (may take a few moments):
kubectl exec app-pod-volumes -- cat /etc/config/LOG_LEVEL
kubectl exec app-pod-volumes -- cat /etc/config/NEW_FEATURE
Subtask 4.2: Update a Secret
Update the Secret:
kubectl patch secret db-credentials --patch '{"data":{"api-key":"'$(echo -n 'new-api-key-123' | base64)'"}}'
Verify the Secret update:
kubectl describe secret db-credentials
Task 5: Clean Up and Best Practices
Subtask 5.1: View All Created Resources
List all ConfigMaps:
kubectl get configmaps
List all Secrets:
kubectl get secrets
List all Pods:
kubectl get pods
List all Deployments:
kubectl get deployments
Subtask 5.2: Clean Up Resources
Delete the Pods:
kubectl delete pod app-pod-env app-pod-secret app-pod-volumes
Delete the Deployment:
kubectl delete deployment web-app-deployment
Delete ConfigMaps:
kubectl delete configmap app-config app-properties
Delete Secrets:
kubectl delete secret db-credentials file-credentials
Clean up local files:
rm -f pod-with-configmap.yaml pod-with-secret.yaml pod-with-volumes.yaml deployment-with-config.yaml app.properties
Key Concepts and Best Practices
ConfigMaps vs Secrets
ConfigMaps are used for: • Non-sensitive configuration data • Application settings • Configuration files • Environment-specific values

Secrets are used for: • Sensitive information (passwords, tokens, keys) • TLS certificates • Docker registry credentials • API keys

Security Considerations
• Secrets are base64 encoded, not encrypted by default • Use RBAC to control access to Secrets • Consider using external secret management systems for production • Set appropriate file permissions when mounting Secrets • Avoid logging Secret values

Performance Tips
• ConfigMaps and Secrets have size limits (1MB) • Volume mounts are eventually consistent • Environment variables are set at container startup • Use subPath for mounting specific files

Troubleshooting Common Issues
Issue 1: Pod Not Starting
Problem: Pod stuck in Pending or Error state

Solution:

kubectl describe pod <pod-name>
kubectl logs <pod-name>
Issue 2: ConfigMap/Secret Not Found
Problem: Error mounting ConfigMap or Secret

Solution:

kubectl get configmaps
kubectl get secrets
# Ensure the referenced ConfigMap/Secret exists
Issue 3: File Not Updated After ConfigMap Change
Problem: Mounted files don't reflect ConfigMap updates

Solution: • Wait up to 60 seconds for kubelet sync • Restart the Pod if immediate update is needed • Environment variables require Pod restart

Conclusion
In this lab, you have successfully:

• Created ConfigMaps using both literal values and files to store non-sensitive configuration data • Created Secrets to securely store sensitive information like database credentials • Used ConfigMaps and Secrets as environment variables in Pods to configure applications • Mounted ConfigMaps and Secrets as files in containers for file-based configuration • Applied configuration to Deployments for scalable applications • Updated ConfigMaps and Secrets and observed how changes propagate to running containers

Why This Matters:

Configuration management is crucial for modern applications because it: • Separates configuration from code, making applications more portable • Enables environment-specific deployments without code changes • Improves security by keeping sensitive data separate from application code • Facilitates easier updates and configuration changes without rebuilding images • Supports the twelve-factor app methodology for cloud-native applications

Understanding ConfigMaps and Secrets is essential for the Kubernetes and Cloud Native Associate (KCNA) certification and real-world Kubernetes deployments. These skills enable you to build secure, configurable, and maintainable applications in Kubernetes environments.
