Lab 12: Exploring StatefulSets
Objectives
By the end of this lab, you will be able to:

• Understand the fundamental concepts and use cases of Kubernetes StatefulSets • Deploy a StatefulSet for a sample database application with persistent storage • Configure and attach PersistentVolumeClaims to ensure data persistence • Scale StatefulSets and observe their unique scaling behavior • Compare StatefulSets with Deployments and understand when to use each • Troubleshoot common StatefulSet deployment issues

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with YAML configuration files • Basic knowledge of persistent storage concepts • Understanding of command-line interface operations • Completion of previous Kubernetes labs covering Pods and Deployments

Note: Al Nafi provides ready-to-use Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to begin - no need to build your own VM or install Kubernetes manually.

Lab Environment Setup
Your cloud machine comes pre-configured with: • Kubernetes cluster (single-node for learning purposes) • kubectl command-line tool • Text editor (nano/vim) • All necessary permissions configured

Task 1: Understanding StatefulSets Fundamentals
Subtask 1.1: Explore StatefulSet Concepts
StatefulSets are designed for applications that require: • Stable network identities: Each pod gets a predictable hostname • Ordered deployment and scaling: Pods are created and terminated in order • Persistent storage: Each pod can have its own persistent volume

Let's start by examining the differences between StatefulSets and Deployments:

# First, let's check our cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes
Subtask 1.2: Create a Namespace for Our Lab
# Create a dedicated namespace for this lab
kubectl create namespace statefulset-lab

# Set the namespace as default for this session
kubectl config set-context --current --namespace=statefulset-lab

# Verify the namespace was created
kubectl get namespaces
Task 2: Deploy a StatefulSet for a Sample Database Application
Subtask 2.1: Create a Headless Service
StatefulSets require a headless service to provide stable network identities. Create the service configuration:

# Create the headless service configuration file
cat > mysql-headless-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: statefulset-lab
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None
  selector:
    app: mysql
EOF
Apply the headless service:

# Deploy the headless service
kubectl apply -f mysql-headless-service.yaml

# Verify the service was created
kubectl get services
Subtask 2.2: Create Storage Class for Dynamic Provisioning
# Create a storage class for our persistent volumes
cat > mysql-storage-class.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: kubernetes.io/host-path
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
Apply the storage class:

# Deploy the storage class
kubectl apply -f mysql-storage-class.yaml

# Verify the storage class
kubectl get storageclass
Subtask 2.3: Create the StatefulSet Configuration
Now, let's create our MySQL StatefulSet with persistent storage:

# Create the StatefulSet configuration
cat > mysql-statefulset.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
  namespace: statefulset-lab
spec:
  serviceName: mysql-headless
  replicas: 3
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
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword123"
        - name: MYSQL_DATABASE
          value: "testdb"
        - name: MYSQL_USER
          value: "testuser"
        - name: MYSQL_PASSWORD
          value: "testpass123"
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -u
            - root
            - -prootpassword123
            - -e
            - "SELECT 1"
          initialDelaySeconds: 5
          periodSeconds: 2
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: mysql-storage
      resources:
        requests:
          storage: 1Gi
EOF
Subtask 2.4: Deploy the StatefulSet
# Deploy the StatefulSet
kubectl apply -f mysql-statefulset.yaml

# Watch the pods being created in order
kubectl get pods -w
Note: Press Ctrl+C to stop watching after all pods are running.

Observe how StatefulSet pods are created: • Pods are created one at a time, in order (mysql-statefulset-0, then mysql-statefulset-1, then mysql-statefulset-2) • Each pod waits for the previous one to be ready before starting • Each pod gets a predictable name with an ordinal index

Task 3: Attach PersistentVolumeClaims and Verify Data Persistence
Subtask 3.1: Examine PersistentVolumeClaims
# Check the automatically created PVCs
kubectl get pvc

# Get detailed information about the PVCs
kubectl describe pvc

# Check the persistent volumes
kubectl get pv
Subtask 3.2: Test Data Persistence
Let's connect to one of the MySQL pods and create some test data:

# Connect to the first MySQL pod
kubectl exec -it mysql-statefulset-0 -- mysql -u root -prootpassword123

# Inside the MySQL shell, run these commands:
# CREATE TABLE test_table (id INT PRIMARY KEY, name VARCHAR(50));
# INSERT INTO test_table VALUES (1, 'StatefulSet Test Data');
# SELECT * FROM test_table;
# EXIT;
Alternative method using a single command:

# Create test data using a single command
kubectl exec -it mysql-statefulset-0 -- mysql -u root -prootpassword123 -e "
CREATE DATABASE IF NOT EXISTS testdb;
USE testdb;
CREATE TABLE IF NOT EXISTS test_table (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO test_table VALUES (1, 'StatefulSet Test Data');
SELECT * FROM test_table;
"
Subtask 3.3: Verify Data Persistence by Pod Restart
# Delete the pod to simulate a failure
kubectl delete pod mysql-statefulset-0

# Watch the pod being recreated
kubectl get pods -w
After the pod is recreated, verify the data still exists:

# Check if our test data persisted
kubectl exec -it mysql-statefulset-0 -- mysql -u root -prootpassword123 -e "
USE testdb;
SELECT * FROM test_table;
"
Task 4: Scale the StatefulSet and Observe Its Behavior
Subtask 4.1: Scale Up the StatefulSet
# Scale the StatefulSet from 3 to 5 replicas
kubectl scale statefulset mysql-statefulset --replicas=5

# Watch the scaling process
kubectl get pods -w
Observe the scaling behavior: • New pods are created one at a time, in order • mysql-statefulset-3 is created first, then mysql-statefulset-4 • Each new pod gets its own PVC automatically

Subtask 4.2: Examine the New PVCs
# Check the new PVCs created during scaling
kubectl get pvc

# Verify all pods are running
kubectl get pods
Subtask 4.3: Scale Down the StatefulSet
# Scale down from 5 to 2 replicas
kubectl scale statefulset mysql-statefulset --replicas=2

# Watch the scaling down process
kubectl get pods -w
Observe the scale-down behavior: • Pods are terminated in reverse order (highest ordinal first) • mysql-statefulset-4 is deleted first, then mysql-statefulset-3, then mysql-statefulset-2 • Important: PVCs are NOT automatically deleted during scale-down

Subtask 4.4: Verify PVC Retention
# Check that PVCs from deleted pods still exist
kubectl get pvc

# This shows StatefulSet's data safety feature
# PVCs are retained even when pods are deleted
Subtask 4.5: Scale Back Up to Verify Data Reattachment
# Scale back up to 4 replicas
kubectl scale statefulset mysql-statefulset --replicas=4

# Watch pods being recreated
kubectl get pods -w

# Verify that the recreated pod reattaches to its original PVC
kubectl get pvc
Task 5: Advanced StatefulSet Operations
Subtask 5.1: Update StatefulSet Configuration
Let's update the StatefulSet to change resource limits:

# Update the StatefulSet with new resource limits
kubectl patch statefulset mysql-statefulset -p='{"spec":{"template":{"spec":{"containers":[{"name":"mysql","resources":{"limits":{"memory":"1Gi","cpu":"1000m"}}}]}}}}'

# Check the rollout status
kubectl rollout status statefulset/mysql-statefulset

# Verify the update
kubectl describe statefulset mysql-statefulset
Subtask 5.2: Examine StatefulSet Network Identity
# Check the DNS names of StatefulSet pods
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mysql-statefulset-0.mysql-headless.statefulset-lab.svc.cluster.local

# Test connectivity to specific pods
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mysql-statefulset-1.mysql-headless.statefulset-lab.svc.cluster.local
Subtask 5.3: Create a Regular Service for External Access
# Create a regular service for external access to the StatefulSet
cat > mysql-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: statefulset-lab
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
EOF

# Apply the service
kubectl apply -f mysql-service.yaml

# Verify the service
kubectl get services
Task 6: Monitoring and Troubleshooting
Subtask 6.1: Monitor StatefulSet Status
# Get detailed StatefulSet information
kubectl describe statefulset mysql-statefulset

# Check StatefulSet events
kubectl get events --sort-by=.metadata.creationTimestamp

# Monitor pod logs
kubectl logs mysql-statefulset-0 --tail=20
Subtask 6.2: Common Troubleshooting Commands
# Check pod status and ready conditions
kubectl get pods -o wide

# Describe a specific pod for troubleshooting
kubectl describe pod mysql-statefulset-0

# Check PVC binding status
kubectl get pvc -o wide

# Verify storage class
kubectl describe storageclass mysql-storage
Task 7: Cleanup and Resource Management
Subtask 7.1: Controlled Cleanup
# Scale down to 0 replicas first
kubectl scale statefulset mysql-statefulset --replicas=0

# Wait for all pods to terminate
kubectl get pods

# Delete the StatefulSet
kubectl delete statefulset mysql-statefulset

# Delete services
kubectl delete service mysql-headless mysql-service

# Note: PVCs are still retained for data safety
kubectl get pvc
Subtask 7.2: Complete Cleanup (Optional)
Warning: This will permanently delete all data.

# Delete PVCs (this will delete all data permanently)
kubectl delete pvc --all

# Delete storage class
kubectl delete storageclass mysql-storage

# Delete the namespace
kubectl delete namespace statefulset-lab
Troubleshooting Common Issues
Issue 1: Pods Stuck in Pending State
Symptoms: Pods remain in Pending state Causes: • Insufficient resources • PVC binding issues • Storage class problems

Solutions:

# Check node resources
kubectl describe nodes

# Check PVC status
kubectl describe pvc

# Verify storage class
kubectl get storageclass
Issue 2: Pods Not Starting in Order
Symptoms: Multiple pods starting simultaneously Causes: • Incorrect service configuration • Missing headless service

Solutions:

# Verify headless service exists
kubectl get service mysql-headless

# Check service selector matches StatefulSet labels
kubectl describe service mysql-headless
Issue 3: Data Not Persisting
Symptoms: Data lost after pod restart Causes: • PVC not properly mounted • Incorrect volume mount path

Solutions:

# Check PVC mounting
kubectl describe pod mysql-statefulset-0

# Verify volume mounts
kubectl exec mysql-statefulset-0 -- df -h
Key Concepts Summary
StatefulSet vs Deployment
Feature	StatefulSet	Deployment
Pod Names	Predictable (app-0, app-1)	Random
Scaling	Ordered (sequential)	Parallel
Storage	Persistent per pod	Shared or ephemeral
Network Identity	Stable DNS names	Load-balanced
Use Cases	Databases, distributed systems	Stateless applications
Important StatefulSet Characteristics
• Ordered Deployment: Pods are created sequentially • Stable Network Identity: Each pod gets a predictable hostname • Persistent Storage: Each pod can have its own persistent volume • Ordered Termination: Pods are deleted in reverse order during scale-down • PVC Retention: PVCs are not deleted when StatefulSet is scaled down or deleted

Conclusion
In this lab, you have successfully:

• Deployed a StatefulSet for a MySQL database application with proper configuration • Implemented persistent storage using PersistentVolumeClaims and StorageClasses • Observed StatefulSet scaling behavior including ordered creation and termination • Verified data persistence across pod restarts and scaling operations • Learned troubleshooting techniques for common StatefulSet issues

Why This Matters: StatefulSets are crucial for running stateful applications in Kubernetes, such as databases, message queues, and distributed systems. Understanding StatefulSets enables you to:

• Deploy and manage stateful applications reliably in Kubernetes • Ensure data persistence and consistency in containerized environments • Handle scaling operations safely for applications that require stable identities • Prepare for real-world scenarios involving database deployments and distributed systems

This knowledge is essential for the Certified Kubernetes Application Developer (CKAD) certification and for working with production Kubernetes environments where stateful applications are common.

Next Steps: Practice deploying other stateful applications like PostgreSQL, MongoDB, or Apache Kafka using StatefulSets to reinforce these concepts and explore more advanced scenarios
