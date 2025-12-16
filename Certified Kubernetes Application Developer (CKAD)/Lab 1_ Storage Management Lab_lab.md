Lab 1: Storage Management Lab
Objectives
By the end of this lab, you will be able to:

• Understand Kubernetes storage concepts including StorageClasses, PersistentVolumes (PV), and PersistentVolumeClaims (PVC) • Create and configure a StorageClass for dynamic provisioning • Deploy and manage PersistentVolumes and PersistentVolumeClaims • Deploy a stateful application (MySQL) that uses persistent storage • Verify data persistence across pod restarts and deletions • Troubleshoot common storage-related issues in Kubernetes

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, deployments, services) • Familiarity with YAML configuration files • Basic knowledge of Linux command line operations • Understanding of database concepts (helpful but not required) • Experience with kubectl command-line tool

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 20.04 LTS with Kubernetes cluster (single-node) • kubectl pre-configured and ready to use • All necessary tools and dependencies installed • Root access to perform administrative tasks

Task 1: Understanding Storage Concepts and Creating a StorageClass
Subtask 1.1: Explore Current Storage Configuration
First, let's examine the current storage setup in your Kubernetes cluster.

Check existing StorageClasses:
kubectl get storageclass
View detailed information about default StorageClass:
kubectl describe storageclass
Check for existing PersistentVolumes:
kubectl get pv
Subtask 1.2: Create a Custom StorageClass
Now we'll create a custom StorageClass for dynamic provisioning.

Create the StorageClass configuration file:
cat > custom-storageclass.yaml << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/host-path
parameters:
  type: Directory
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Delete
EOF
Apply the StorageClass configuration:
kubectl apply -f custom-storageclass.yaml
Verify the StorageClass was created:
kubectl get storageclass fast-storage -o yaml
Subtask 1.3: Understand StorageClass Components
Let's break down what each component in our StorageClass does:

• provisioner: Determines which volume plugin is used for provisioning PVs • parameters: Passes parameters to the provisioner • allowVolumeExpansion: Allows PVCs to be expanded after creation • volumeBindingMode: Controls when volume binding occurs • reclaimPolicy: Defines what happens to PV when PVC is deleted

Task 2: Create PersistentVolume and PersistentVolumeClaim
Subtask 2.1: Create a Static PersistentVolume
We'll create a static PV first to understand the concept before moving to dynamic provisioning.

Create the PersistentVolume configuration:
cat > mysql-pv.yaml << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    app: mysql
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-storage
  hostPath:
    path: /var/lib/mysql-data
    type: DirectoryOrCreate
EOF
Apply the PersistentVolume configuration:
kubectl apply -f mysql-pv.yaml
Verify the PV was created and check its status:
kubectl get pv mysql-pv
kubectl describe pv mysql-pv
Subtask 2.2: Create a PersistentVolumeClaim
Now we'll create a PVC that will bind to our PV.

Create the PersistentVolumeClaim configuration:
cat > mysql-pvc.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: fast-storage
  selector:
    matchLabels:
      app: mysql
EOF
Apply the PVC configuration:
kubectl apply -f mysql-pvc.yaml
Verify the PVC was created and bound to the PV:
kubectl get pvc mysql-pvc
kubectl describe pvc mysql-pvc
Check that the PV is now bound:
kubectl get pv mysql-pv
The status should show Bound instead of Available.

Task 3: Deploy MySQL Stateful Application
Subtask 3.1: Create MySQL Secret for Database Credentials
Before deploying MySQL, we need to create a secret for the database password.

Create the MySQL secret:
kubectl create secret generic mysql-secret \
  --from-literal=mysql-root-password=MySecurePassword123 \
  --from-literal=mysql-database=testdb \
  --from-literal=mysql-user=testuser \
  --from-literal=mysql-password=TestUserPassword123
Verify the secret was created:
kubectl get secret mysql-secret
kubectl describe secret mysql-secret
Subtask 3.2: Deploy MySQL Application
Now we'll deploy MySQL using our PVC for persistent storage.

Create the MySQL deployment configuration:
cat > mysql-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
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
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-database
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
EOF
Deploy MySQL:
kubectl apply -f mysql-deployment.yaml
Check the deployment status:
kubectl get deployment mysql-deployment
kubectl get pods -l app=mysql
Wait for the pod to be ready (this may take a few minutes):
kubectl wait --for=condition=ready pod -l app=mysql --timeout=300s
Subtask 3.3: Create MySQL Service
Create a service to access MySQL from within the cluster.

Create the MySQL service configuration:
cat > mysql-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  labels:
    app: mysql
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
  type: ClusterIP
EOF
Apply the service configuration:
kubectl apply -f mysql-service.yaml
Verify the service was created:
kubectl get service mysql-service
kubectl describe service mysql-service
Task 4: Verify Data Persistence
Subtask 4.1: Connect to MySQL and Create Test Data
Let's connect to MySQL and create some test data to verify persistence.

Get the MySQL pod name:
MYSQL_POD=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')
echo "MySQL Pod: $MYSQL_POD"
Connect to MySQL and create test data:
kubectl exec -it $MYSQL_POD -- mysql -u root -pMySecurePassword123 -e "
CREATE DATABASE IF NOT EXISTS testdb;
USE testdb;
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO users (name, email) VALUES 
    ('John Doe', 'john@example.com'),
    ('Jane Smith', 'jane@example.com'),
    ('Bob Johnson', 'bob@example.com');
SELECT * FROM users;
"
Verify the data was inserted:
kubectl exec -it $MYSQL_POD -- mysql -u root -pMySecurePassword123 -e "
USE testdb;
SELECT COUNT(*) as 'Total Users' FROM users;
"
Subtask 4.2: Test Data Persistence by Restarting the Pod
Now we'll delete the pod and verify that data persists when a new pod is created.

Delete the MySQL pod:
kubectl delete pod $MYSQL_POD
Wait for the new pod to be ready:
kubectl wait --for=condition=ready pod -l app=mysql --timeout=300s
Get the new pod name:
NEW_MYSQL_POD=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')
echo "New MySQL Pod: $NEW_MYSQL_POD"
Verify that our data still exists:
kubectl exec -it $NEW_MYSQL_POD -- mysql -u root -pMySecurePassword123 -e "
USE testdb;
SELECT * FROM users;
SELECT COUNT(*) as 'Total Users' FROM users;
"
The data should still be there, proving that persistence is working correctly.

Subtask 4.3: Test Data Persistence by Scaling Down and Up
Let's also test persistence by scaling the deployment.

Scale down the deployment to 0 replicas:
kubectl scale deployment mysql-deployment --replicas=0
Verify no pods are running:
kubectl get pods -l app=mysql
Scale back up to 1 replica:
kubectl scale deployment mysql-deployment --replicas=1
Wait for the pod to be ready:
kubectl wait --for=condition=ready pod -l app=mysql --timeout=300s
Verify data persistence again:
SCALED_MYSQL_POD=$(kubectl get pods -l app=mysql -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $SCALED_MYSQL_POD -- mysql -u root -pMySecurePassword123 -e "
USE testdb;
SELECT * FROM users;
"
Task 5: Advanced Storage Operations
Subtask 5.1: Monitor Storage Usage
Let's check how much storage is being used by our MySQL application.

Check PV and PVC status:
kubectl get pv,pvc
Get detailed information about storage usage:
kubectl describe pv mysql-pv
kubectl describe pvc mysql-pvc
Check the actual disk usage on the host:
sudo du -sh /var/lib/mysql-data
sudo ls -la /var/lib/mysql-data
Subtask 5.2: Create a Backup of the Data
Let's create a simple backup of our MySQL data.

Create a backup directory:
sudo mkdir -p /backup/mysql
Create a database dump:
kubectl exec $SCALED_MYSQL_POD -- mysqldump -u root -pMySecurePassword123 --all-databases > /tmp/mysql-backup.sql
Copy the backup to our backup directory:
sudo cp /tmp/mysql-backup.sql /backup/mysql/mysql-backup-$(date +%Y%m%d-%H%M%S).sql
Verify the backup was created:
sudo ls -la /backup/mysql/
Subtask 5.3: Test Dynamic Provisioning
Now let's test dynamic provisioning by creating a PVC without a pre-existing PV.

Create a new PVC for dynamic provisioning:
cat > dynamic-pvc.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: fast-storage
EOF
Apply the dynamic PVC:
kubectl apply -f dynamic-pvc.yaml
Check if a PV was automatically created:
kubectl get pvc dynamic-pvc
kubectl get pv
Note: In a single-node cluster with hostPath provisioner, dynamic provisioning behavior may vary. This demonstrates the concept for learning purposes.

Troubleshooting Common Issues
Issue 1: Pod Stuck in Pending State
If your MySQL pod is stuck in Pending state:

Check pod events:
kubectl describe pod -l app=mysql
Check PVC status:
kubectl get pvc mysql-pvc
Verify PV availability:
kubectl get pv
Issue 2: PVC Not Binding to PV
If your PVC is not binding:

Check if StorageClass matches:
kubectl describe pvc mysql-pvc
kubectl describe pv mysql-pv
Verify access modes compatibility:
kubectl get pvc mysql-pvc -o yaml | grep -A 5 spec
kubectl get pv mysql-pv -o yaml | grep -A 10 spec
Issue 3: MySQL Connection Issues
If you cannot connect to MySQL:

Check pod logs:
kubectl logs -l app=mysql
Verify service endpoints:
kubectl get endpoints mysql-service
Test connectivity:
kubectl run mysql-client --image=mysql:8.0 -it --rm --restart=Never -- mysql -h mysql-service -u root -pMySecurePassword123
Cleanup
When you're finished with the lab, clean up the resources:

Delete the deployment and service:
kubectl delete deployment mysql-deployment
kubectl delete service mysql-service
kubectl delete secret mysql-secret
Delete PVC and PV:
kubectl delete pvc mysql-pvc dynamic-pvc
kubectl delete pv mysql-pv
Delete StorageClass:
kubectl delete storageclass fast-storage
Clean up local directories:
sudo rm -rf /var/lib/mysql-data
sudo rm -rf /backup/mysql
Conclusion
Congratulations! You have successfully completed the Storage Management Lab. Here's what you accomplished:

Key Achievements: • Created and configured a custom StorageClass for dynamic provisioning • Successfully deployed and managed PersistentVolumes and PersistentVolumeClaims • Deployed a stateful MySQL application with persistent storage • Verified data persistence across pod restarts, deletions, and scaling operations • Learned troubleshooting techniques for common storage issues • Performed backup operations and tested dynamic provisioning concepts

Why This Matters: Storage management is crucial for stateful applications in Kubernetes. The skills you've learned enable you to: • Deploy databases and other stateful applications reliably • Ensure data survives pod failures and cluster maintenance • Implement proper backup and recovery strategies • Optimize storage performance and costs in production environments

Real-World Applications: These storage concepts are essential for: • Running production databases (MySQL, PostgreSQL, MongoDB) • Implementing CI/CD systems with persistent data • Managing application logs and metrics storage • Building resilient microservices architectures

This lab provides the foundation for the Certified Kubernetes Administrator (CKA) certification, specifically covering storage-related objectives. The hands-on experience with PVs, PVCs, and StorageClasses prepares you for real-world Kubernetes storage challenges.
