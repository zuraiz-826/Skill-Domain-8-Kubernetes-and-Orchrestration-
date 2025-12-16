Lab 15: Managing Persistent and Ephemeral Volumes
Objectives
By the end of this lab, you will be able to:

• Understand the difference between Persistent and Ephemeral volumes in Kubernetes • Create and configure PersistentVolumes (PV) and PersistentVolumeClaims (PVC) • Deploy applications using persistent storage for data retention • Implement ephemeral volumes for temporary storage needs • Validate data persistence behavior across Pod restarts and deletions • Troubleshoot common volume-related issues in Kubernetes

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with YAML configuration files • Knowledge of basic Linux commands • Understanding of container storage concepts • Experience with kubectl command-line tool

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Kubernetes cluster (single-node or multi-node) • kubectl configured and ready to use • Text editor (nano/vim) • All necessary permissions for volume operations

Task 1: Understanding Volume Types and Creating Persistent Storage
Subtask 1.1: Explore Current Storage Classes
First, let's examine the available storage classes in your cluster.

Check available storage classes:
kubectl get storageclass
Get detailed information about the default storage class:
kubectl describe storageclass
Create a directory for lab files:
mkdir ~/volume-lab
cd ~/volume-lab
Subtask 1.2: Create a PersistentVolume
Create a PersistentVolume that will serve as the underlying storage for our database application.

Create the PersistentVolume configuration:
cat > pv-database.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: database-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/database"
  persistentVolumeReclaimPolicy: Retain
EOF
Apply the PersistentVolume:
kubectl apply -f pv-database.yaml
Verify the PV creation:
kubectl get pv
kubectl describe pv database-pv
Subtask 1.3: Create a PersistentVolumeClaim
Now create a PersistentVolumeClaim that will bind to our PersistentVolume.

Create the PVC configuration:
cat > pvc-database.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
Apply the PVC:
kubectl apply -f pvc-database.yaml
Verify the PVC binding:
kubectl get pvc
kubectl get pv
Expected Output: You should see the PVC status as Bound and the PV status as Bound as well.

Task 2: Deploy Database Application with Persistent Storage
Subtask 2.1: Create a MySQL Database with Persistent Volume
Deploy a MySQL database that uses the persistent volume for data storage.

Create the MySQL deployment configuration:
cat > mysql-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-database
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword123"
        - name: MYSQL_DATABASE
          value: "testdb"
        - name: MYSQL_USER
          value: "testuser"
        - name: MYSQL_PASSWORD
          value: "testpass123"
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: database-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
EOF
Deploy the MySQL application:
kubectl apply -f mysql-deployment.yaml
Wait for the deployment to be ready:
kubectl rollout status deployment/mysql-database
Verify the deployment:
kubectl get pods -l app=mysql
kubectl get svc mysql-service
Subtask 2.2: Test Database Connectivity and Create Test Data
Connect to the MySQL database and create test data:
kubectl exec -it deployment/mysql-database -- mysql -u testuser -ptestpass123 testdb
Inside the MySQL prompt, create a test table and insert data:
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, email) VALUES 
('John Doe', 'john@example.com'),
('Jane Smith', 'jane@example.com'),
('Bob Johnson', 'bob@example.com');

SELECT * FROM users;
Exit the MySQL prompt:
EXIT;
Task 3: Deploy Application with Ephemeral Volume
Subtask 3.1: Create Application with EmptyDir Volume
Create an application that uses ephemeral storage for temporary files.

Create the ephemeral volume application configuration:
cat > app-ephemeral.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: temp-storage-app
  labels:
    app: temp-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: temp-app
  template:
    metadata:
      labels:
        app: temp-app
    spec:
      containers:
      - name: app-container
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: temp-storage
          mountPath: /tmp/app-data
        - name: cache-storage
          mountPath: /var/cache/app
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo 'Temporary data: '$(date) >> /tmp/app-data/temp-file.txt; echo 'Cache data: '$(date) >> /var/cache/app/cache-file.txt; sleep 30; done"]
      - name: sidecar-container
        image: busybox
        volumeMounts:
        - name: temp-storage
          mountPath: /shared-temp
        command: ["/bin/sh"]
        args: ["-c", "while true; do if [ -f /shared-temp/temp-file.txt ]; then echo 'Sidecar reading: '; tail -n 1 /shared-temp/temp-file.txt; fi; sleep 60; done"]
      volumes:
      - name: temp-storage
        emptyDir: {}
      - name: cache-storage
        emptyDir:
          sizeLimit: 100Mi
EOF
Deploy the ephemeral volume application:
kubectl apply -f app-ephemeral.yaml
Verify the deployment:
kubectl get pods -l app=temp-app
kubectl rollout status deployment/temp-storage-app
Subtask 3.2: Examine Ephemeral Volume Behavior
Check the contents of ephemeral volumes:
# Get one of the pod names
POD_NAME=$(kubectl get pods -l app=temp-app -o jsonpath='{.items[0].metadata.name}')
echo "Using pod: $POD_NAME"

# Check temporary file contents
kubectl exec $POD_NAME -c app-container -- cat /tmp/app-data/temp-file.txt

# Check cache file contents
kubectl exec $POD_NAME -c app-container -- cat /var/cache/app/cache-file.txt
Verify volume sharing between containers:
kubectl logs $POD_NAME -c sidecar-container --tail=5
Task 4: Validate Data Persistence Behavior
Subtask 4.1: Test Persistent Volume Data Retention
Delete the MySQL pod to simulate a restart:
kubectl delete pod -l app=mysql
Wait for the new pod to be ready:
kubectl rollout status deployment/mysql-database
Verify data persistence by connecting to the new pod:
kubectl exec -it deployment/mysql-database -- mysql -u testuser -ptestpass123 testdb -e "SELECT * FROM users;"
Expected Result: You should see the same data that was inserted earlier, proving that data persisted across pod restarts.

Subtask 4.2: Test Ephemeral Volume Data Loss
Record current ephemeral data:
POD_NAME=$(kubectl get pods -l app=temp-app -o jsonpath='{.items[0].metadata.name}')
echo "Current temp file contents:"
kubectl exec $POD_NAME -c app-container -- cat /tmp/app-data/temp-file.txt | tail -3
Delete the ephemeral volume pod:
kubectl delete pod $POD_NAME
Wait for replacement pod and check data:
# Wait for new pod
kubectl rollout status deployment/temp-storage-app

# Get new pod name
NEW_POD_NAME=$(kubectl get pods -l app=temp-app -o jsonpath='{.items[0].metadata.name}')
echo "New pod: $NEW_POD_NAME"

# Wait a moment for the container to create some data
sleep 60

# Check if old data exists (it shouldn't)
kubectl exec $NEW_POD_NAME -c app-container -- ls -la /tmp/app-data/
kubectl exec $NEW_POD_NAME -c app-container -- cat /tmp/app-data/temp-file.txt
Expected Result: The ephemeral volume should be empty or contain only new data, demonstrating that ephemeral volumes don't persist across pod restarts.

Task 5: Advanced Volume Management
Subtask 5.1: Create a ConfigMap Volume
Create a ConfigMap for application configuration:
cat > app-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    database.host=mysql-service
    database.port=3306
    database.name=testdb
    cache.enabled=true
    cache.ttl=300
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /config {
            alias /etc/config;
            autoindex on;
        }
    }
EOF
Apply the ConfigMap:
kubectl apply -f app-config.yaml
Create a deployment using ConfigMap as volume:
cat > app-with-config.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-app
  template:
    metadata:
      labels:
        app: config-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: nginx-config
        configMap:
          name: app-config
EOF
Deploy the application:
kubectl apply -f app-with-config.yaml
Verify ConfigMap volume mounting:
kubectl exec deployment/app-with-config -- ls -la /etc/config/
kubectl exec deployment/app-with-config -- cat /etc/config/app.properties
Subtask 5.2: Monitor Volume Usage
Check volume usage and status:
# Check PV and PVC status
kubectl get pv,pvc

# Describe PVC for detailed information
kubectl describe pvc database-pvc

# Check which pods are using volumes
kubectl get pods -o wide
Create a script to monitor volume usage:
cat > monitor-volumes.sh << 'EOF'
#!/bin/bash
echo "=== Persistent Volumes ==="
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,ACCESS:.spec.accessModes,STATUS:.status.phase,CLAIM:.spec.claimRef.name

echo -e "\n=== Persistent Volume Claims ==="
kubectl get pvc -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,VOLUME:.spec.volumeName,CAPACITY:.status.capacity.storage,ACCESS:.status.accessModes

echo -e "\n=== Pods with Volumes ==="
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,VOLUMES:.spec.volumes[*].name
EOF

chmod +x monitor-volumes.sh
./monitor-volumes.sh
Task 6: Cleanup and Volume Management
Subtask 6.1: Proper Volume Cleanup
Delete applications first:
kubectl delete deployment mysql-database
kubectl delete deployment temp-storage-app
kubectl delete deployment app-with-config
kubectl delete service mysql-service
Delete ConfigMap:
kubectl delete configmap app-config
Delete PVC (this will also affect PV based on reclaim policy):
kubectl delete pvc database-pvc
Check PV status after PVC deletion:
kubectl get pv database-pv
Manually delete PV if needed:
kubectl delete pv database-pv
Subtask 6.2: Verify Complete Cleanup
Verify all resources are cleaned up:
kubectl get pv,pvc
kubectl get deployments
kubectl get pods
kubectl get configmaps
Clean up local files:
cd ~
rm -rf ~/volume-lab
Troubleshooting Common Issues
Issue 1: PVC Stuck in Pending State
Symptoms: PVC remains in Pending status Causes:

No suitable PV available
Storage class issues
Insufficient permissions
Solutions:

# Check PVC events
kubectl describe pvc <pvc-name>

# Check available PVs
kubectl get pv

# Check storage classes
kubectl get storageclass
Issue 2: Pod Cannot Mount Volume
Symptoms: Pod fails to start with volume mount errors Causes:

PVC not bound
Incorrect mount path
Permission issues
Solutions:

# Check pod events
kubectl describe pod <pod-name>

# Verify PVC status
kubectl get pvc

# Check volume mounts in pod spec
kubectl get pod <pod-name> -o yaml | grep -A 10 volumeMounts
Issue 3: Data Not Persisting
Symptoms: Data disappears after pod restart Causes:

Using ephemeral volumes instead of persistent
Incorrect volume configuration
PV reclaim policy issues
Solutions:

# Verify volume type
kubectl describe pod <pod-name> | grep -A 5 Volumes

# Check PV reclaim policy
kubectl get pv -o custom-columns=NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy
Key Concepts Summary
Persistent Volumes (PV)
Cluster-level resources that provide storage
Independent lifecycle from pods
Reclaim policies: Retain, Delete, Recycle
Persistent Volume Claims (PVC)
Namespace-scoped requests for storage
Bind to suitable PVs based on requirements
Used by pods to access persistent storage
Ephemeral Volumes
Pod-scoped storage that exists only during pod lifetime
Types: emptyDir, configMap, secret, downwardAPI
Automatically deleted when pod is removed
Volume Types Comparison
Volume Type	Persistence	Scope	Use Case
PersistentVolume	Yes	Cluster	Database storage, file storage
emptyDir	No	Pod	Temporary files, cache
configMap	No	Namespace	Configuration files
secret	No	Namespace	Sensitive data
Conclusion
In this lab, you have successfully:

• Created and managed Persistent Volumes for long-term data storage • Deployed a MySQL database with persistent storage that survives pod restarts • Implemented ephemeral volumes for temporary storage needs • Validated data persistence behavior across different volume types • Explored advanced volume types including ConfigMap volumes • Learned troubleshooting techniques for common volume issues

Why This Matters: Understanding volume management is crucial for production Kubernetes deployments. Persistent volumes ensure data durability for stateful applications like databases, while ephemeral volumes provide efficient temporary storage. This knowledge is essential for the CKAD certification and real-world Kubernetes operations.

Next Steps: Practice with different storage classes, explore dynamic provisioning, and learn about StatefulSets for more complex stateful application deployments.
