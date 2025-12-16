Lab 11: Managing Kubernetes Volumes and Persistent Storage
Objectives
By the end of this lab, you will be able to:

• Understand the difference between Volumes, PersistentVolumes (PV), and PersistentVolumeClaims (PVC) • Create and configure PersistentVolumes with different storage classes • Create PersistentVolumeClaims to request storage resources • Deploy applications that utilize persistent storage • Verify data persistence across pod restarts and deletions • Troubleshoot common storage-related issues in Kubernetes • Implement best practices for managing persistent data in containerized applications

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with YAML syntax and Kubernetes manifest files • Basic Linux command-line knowledge • Understanding of file systems and storage concepts • Completed previous Kubernetes labs or equivalent experience

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. Your machine comes with:

• Kubernetes cluster (minikube) pre-installed and configured • kubectl command-line tool ready to use • All necessary permissions and tools configured • No need to build your own VM or install software

Lab Environment Setup
Task 1: Verify Kubernetes Cluster Status
Subtask 1.1: Check Cluster Information
First, let's verify that your Kubernetes cluster is running properly.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Check available storage classes
kubectl get storageclass
Expected Output:

NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  5m
Subtask 1.2: Create Lab Namespace
Create a dedicated namespace for this lab to keep resources organized.

# Create namespace for the lab
kubectl create namespace storage-lab

# Set the namespace as default for this session
kubectl config set-context --current --namespace=storage-lab

# Verify namespace creation
kubectl get namespaces
Task 2: Create PersistentVolume and PersistentVolumeClaim
Subtask 2.1: Create a PersistentVolume
A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes.

Create a file named persistent-volume.yaml:

cat > persistent-volume.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/lab-data"
  persistentVolumeReclaimPolicy: Retain
EOF
Key Configuration Explained: • storageClassName: manual - Custom storage class for this lab • capacity: 1Gi - Allocates 1 gigabyte of storage • accessModes: ReadWriteOnce - Can be mounted by one node at a time • hostPath - Uses local node storage (suitable for single-node clusters) • persistentVolumeReclaimPolicy: Retain - Data persists after PVC deletion

Apply the PersistentVolume:

# Create the PersistentVolume
kubectl apply -f persistent-volume.yaml

# Verify PV creation
kubectl get pv

# Get detailed information about the PV
kubectl describe pv lab-pv
Subtask 2.2: Create a PersistentVolumeClaim
A PersistentVolumeClaim (PVC) is a request for storage by a user. It's similar to a Pod requesting compute resources.

Create a file named persistent-volume-claim.yaml:

cat > persistent-volume-claim.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-pvc
  namespace: storage-lab
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
EOF
Key Configuration Explained: • storageClassName: manual - Must match the PV's storage class • accessModes: ReadWriteOnce - Must be compatible with PV access modes • requests.storage: 500Mi - Requests 500 megabytes (less than PV capacity)

Apply the PersistentVolumeClaim:

# Create the PVC
kubectl apply -f persistent-volume-claim.yaml

# Check PVC status
kubectl get pvc

# Verify PV is now bound to PVC
kubectl get pv

# Get detailed PVC information
kubectl describe pvc lab-pvc
Expected Status: The PVC should show Bound status, and the PV should show Bound to storage-lab/lab-pvc.

Task 3: Deploy Application with Persistent Storage
Subtask 3.1: Create Application Deployment
Now we'll deploy an application that writes data to the persistent volume to demonstrate data persistence.

Create a file named storage-app-deployment.yaml:

cat > storage-app-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-app
  namespace: storage-lab
  labels:
    app: storage-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage-app
  template:
    metadata:
      labels:
        app: storage-app
    spec:
      containers:
      - name: storage-container
        image: busybox:1.35
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo $(date) >> /data/timestamps.log; sleep 30; done"]
        volumeMounts:
        - name: storage-volume
          mountPath: /data
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      volumes:
      - name: storage-volume
        persistentVolumeClaim:
          claimName: lab-pvc
EOF
Application Behavior Explained: • busybox:1.35 - Lightweight Linux container • Command - Writes timestamp to /data/timestamps.log every 30 seconds • volumeMounts - Mounts PVC at /data path inside container • Resource limits - Prevents resource overconsumption

Deploy the application:

# Deploy the application
kubectl apply -f storage-app-deployment.yaml

# Check deployment status
kubectl get deployments

# Check pod status
kubectl get pods

# Wait for pod to be running
kubectl wait --for=condition=Ready pod -l app=storage-app --timeout=60s
Subtask 3.2: Verify Data Writing
Let's verify that the application is successfully writing data to the persistent volume.

# Get the pod name
POD_NAME=$(kubectl get pods -l app=storage-app -o jsonpath='{.items[0].metadata.name}')

# Check if data is being written
kubectl exec $POD_NAME -- ls -la /data

# View the content of the log file
kubectl exec $POD_NAME -- cat /data/timestamps.log

# Monitor real-time data writing (press Ctrl+C to stop)
kubectl exec $POD_NAME -- tail -f /data/timestamps.log
Expected Output: You should see timestamps being written to the file every 30 seconds.

Subtask 3.3: Create Service for Application Access
Create a service to access the application:

cat > storage-app-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: storage-app-service
  namespace: storage-lab
spec:
  selector:
    app: storage-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
EOF
# Apply the service
kubectl apply -f storage-app-service.yaml

# Verify service creation
kubectl get services
Task 4: Verify Data Persistence
Subtask 4.1: Delete and Recreate Application
Now we'll test the core functionality of persistent storage by deleting the application and verifying that data persists.

# First, let's see how much data we have
kubectl exec $POD_NAME -- wc -l /data/timestamps.log

# Note the current timestamp count
INITIAL_COUNT=$(kubectl exec $POD_NAME -- wc -l /data/timestamps.log | awk '{print $1}')
echo "Initial line count: $INITIAL_COUNT"

# Delete the deployment (this will delete the pod)
kubectl delete deployment storage-app

# Verify pod is deleted
kubectl get pods

# Wait a moment to ensure complete deletion
sleep 10
Subtask 4.2: Recreate Application and Verify Data
# Recreate the deployment
kubectl apply -f storage-app-deployment.yaml

# Wait for new pod to be ready
kubectl wait --for=condition=Ready pod -l app=storage-app --timeout=60s

# Get new pod name
NEW_POD_NAME=$(kubectl get pods -l app=storage-app -o jsonpath='{.items[0].metadata.name}')

# Check if our data still exists
kubectl exec $NEW_POD_NAME -- ls -la /data

# Verify the log file still contains our previous data
kubectl exec $NEW_POD_NAME -- head -5 /data/timestamps.log

# Check current line count
CURRENT_COUNT=$(kubectl exec $NEW_POD_NAME -- wc -l /data/timestamps.log | awk '{print $1}')
echo "Current line count: $CURRENT_COUNT"

# The count should be equal or greater than initial count
if [ $CURRENT_COUNT -ge $INITIAL_COUNT ]; then
    echo "SUCCESS: Data persisted across pod deletion and recreation!"
else
    echo "WARNING: Some data may have been lost"
fi
Subtask 4.3: Advanced Persistence Testing
Let's perform additional tests to thoroughly verify persistence:

# Create a test file with specific content
kubectl exec $NEW_POD_NAME -- sh -c 'echo "Persistence Test - $(date)" > /data/test-file.txt'

# Add some structured data
kubectl exec $NEW_POD_NAME -- sh -c 'echo "Lab: Kubernetes Persistent Storage" >> /data/test-file.txt'
kubectl exec $NEW_POD_NAME -- sh -c 'echo "Student: $(whoami)" >> /data/test-file.txt'
kubectl exec $NEW_POD_NAME -- sh -c 'echo "Node: $(hostname)" >> /data/test-file.txt'

# Verify file creation
kubectl exec $NEW_POD_NAME -- cat /data/test-file.txt

# Scale deployment to 0 replicas (another way to delete pods)
kubectl scale deployment storage-app --replicas=0

# Verify no pods are running
kubectl get pods

# Scale back to 1 replica
kubectl scale deployment storage-app --replicas=1

# Wait for pod to be ready
kubectl wait --for=condition=Ready pod -l app=storage-app --timeout=60s

# Get newest pod name
FINAL_POD_NAME=$(kubectl get pods -l app=storage-app -o jsonpath='{.items[0].metadata.name}')

# Verify our test file still exists
kubectl exec $FINAL_POD_NAME -- cat /data/test-file.txt

echo "Final persistence test completed successfully!"
Task 5: Storage Management and Monitoring
Subtask 5.1: Monitor Storage Usage
# Check storage usage inside the pod
kubectl exec $FINAL_POD_NAME -- df -h /data

# Check PV and PVC status
kubectl get pv,pvc

# Get detailed storage information
kubectl describe pv lab-pv
kubectl describe pvc lab-pvc

# Check events related to storage
kubectl get events --field-selector involvedObject.kind=PersistentVolume
kubectl get events --field-selector involvedObject.kind=PersistentVolumeClaim
Subtask 5.2: Create Storage Monitoring Script
Create a script to monitor storage usage:

cat > monitor-storage.sh << 'EOF'
#!/bin/bash

echo "=== Kubernetes Storage Monitoring ==="
echo "Date: $(date)"
echo

echo "=== PersistentVolumes ==="
kubectl get pv -o wide

echo
echo "=== PersistentVolumeClaims ==="
kubectl get pvc -o wide

echo
echo "=== Storage Usage in Pod ==="
POD_NAME=$(kubectl get pods -l app=storage-app -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ ! -z "$POD_NAME" ]; then
    echo "Pod: $POD_NAME"
    kubectl exec $POD_NAME -- df -h /data 2>/dev/null || echo "Pod not ready or not found"
    echo "Files in /data:"
    kubectl exec $POD_NAME -- ls -la /data 2>/dev/null || echo "Cannot access /data"
else
    echo "No storage-app pods found"
fi

echo
echo "=== Recent Storage Events ==="
kubectl get events --field-selector involvedObject.kind=PersistentVolume,involvedObject.kind=PersistentVolumeClaim --sort-by='.lastTimestamp' | tail -5
EOF

# Make script executable
chmod +x monitor-storage.sh

# Run the monitoring script
./monitor-storage.sh
Task 6: Advanced Storage Scenarios
Subtask 6.1: Create Multiple PVCs
Let's create additional PVCs to understand storage allocation:

# Create additional PV
cat > additional-pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-pv-2
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/lab-data-2"
  persistentVolumeReclaimPolicy: Retain
EOF

# Apply additional PV
kubectl apply -f additional-pv.yaml

# Create second PVC
cat > additional-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-pvc-2
  namespace: storage-lab
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Apply additional PVC
kubectl apply -f additional-pvc.yaml

# Check all PVs and PVCs
kubectl get pv,pvc
Subtask 6.2: Deploy Multi-Volume Application
Create an application that uses multiple volumes:

cat > multi-volume-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-volume-app
  namespace: storage-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multi-volume-app
  template:
    metadata:
      labels:
        app: multi-volume-app
    spec:
      containers:
      - name: multi-volume-container
        image: busybox:1.35
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo 'Volume 1: '$(date) >> /data1/log1.txt; echo 'Volume 2: '$(date) >> /data2/log2.txt; sleep 45; done"]
        volumeMounts:
        - name: volume-1
          mountPath: /data1
        - name: volume-2
          mountPath: /data2
      volumes:
      - name: volume-1
        persistentVolumeClaim:
          claimName: lab-pvc
      - name: volume-2
        persistentVolumeClaim:
          claimName: lab-pvc-2
EOF

# Deploy multi-volume application
kubectl apply -f multi-volume-app.yaml

# Wait for pod to be ready
kubectl wait --for=condition=Ready pod -l app=multi-volume-app --timeout=60s

# Get pod name
MULTI_POD=$(kubectl get pods -l app=multi-volume-app -o jsonpath='{.items[0].metadata.name}')

# Verify both volumes are mounted
kubectl exec $MULTI_POD -- df -h | grep data
kubectl exec $MULTI_POD -- ls -la /data1
kubectl exec $MULTI_POD -- ls -la /data2

# Check logs in both volumes
sleep 60
kubectl exec $MULTI_POD -- cat /data1/log1.txt
kubectl exec $MULTI_POD -- cat /data2/log2.txt
Task 7: Troubleshooting and Best Practices
Subtask 7.1: Common Issues and Solutions
Let's explore common storage issues and their solutions:

# Create a problematic PVC (requesting more storage than available)
cat > problematic-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: large-pvc
  namespace: storage-lab
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi  # This exceeds our available PV capacity
EOF

# Apply the problematic PVC
kubectl apply -f problematic-pvc.yaml

# Check PVC status (should be Pending)
kubectl get pvc large-pvc

# Describe to see the issue
kubectl describe pvc large-pvc

# Check events to understand the problem
kubectl get events --field-selector involvedObject.name=large-pvc

echo "This PVC will remain in Pending state because no PV can satisfy the 10Gi request"
Subtask 7.2: Storage Cleanup and Best Practices
# Create cleanup script
cat > cleanup-storage.sh << 'EOF'
#!/bin/bash

echo "=== Storage Cleanup Script ==="

# Delete deployments first
echo "Deleting deployments..."
kubectl delete deployment storage-app multi-volume-app --ignore-not-found=true

# Wait for pods to terminate
echo "Waiting for pods to terminate..."
kubectl wait --for=delete pod -l app=storage-app --timeout=60s 2>/dev/null || true
kubectl wait --for=delete pod -l app=multi-volume-app --timeout=60s 2>/dev/null || true

# Delete PVCs
echo "Deleting PVCs..."
kubectl delete pvc lab-pvc lab-pvc-2 large-pvc --ignore-not-found=true

# Delete PVs
echo "Deleting PVs..."
kubectl delete pv lab-pv lab-pv-2 --ignore-not-found=true

# Delete service
echo "Deleting service..."
kubectl delete service storage-app-service --ignore-not-found=true

echo "Cleanup completed!"

# Show remaining resources
echo "Remaining storage resources:"
kubectl get pv,pvc
EOF

# Make cleanup script executable
chmod +x cleanup-storage.sh
Subtask 7.3: Storage Best Practices Summary
Create a best practices documentation:

cat > storage-best-practices.md << 'EOF'
# Kubernetes Storage Best Practices

## 1. Storage Class Selection
- Use appropriate storage classes for different workloads
- Consider performance requirements (SSD vs HDD)
- Plan for backup and disaster recovery

## 2. Resource Management
- Set appropriate storage requests and limits
- Monitor storage usage regularly
- Implement storage quotas in namespaces

## 3. Data Persistence Strategy
- Use StatefulSets for stateful applications
- Implement proper backup strategies
- Consider data replication for critical applications

## 4. Security Considerations
- Use proper RBAC for storage resources
- Encrypt sensitive data at rest
- Implement network policies for storage access

## 5. Monitoring and Alerting
- Monitor PV/PVC status regularly
- Set up alerts for storage capacity
- Track storage performance metrics

## 6. Cleanup and Maintenance
- Regularly clean up unused PVCs
- Monitor orphaned PVs
- Implement retention policies
EOF

echo "Best practices documentation created: storage-best-practices.md"
Verification and Testing
Final Verification Steps
Let's perform a comprehensive verification of everything we've learned:

# Run comprehensive verification
cat > final-verification.sh << 'EOF'
#!/bin/bash

echo "=== Final Lab Verification ==="
echo "Date: $(date)"
echo

# Check if main components exist
echo "1. Checking PersistentVolumes..."
kubectl get pv | grep -E "(lab-pv|lab-pv-2)" && echo "✓ PVs found" || echo "✗ PVs missing"

echo
echo "2. Checking PersistentVolumeClaims..."
kubectl get pvc | grep -E "(lab-pvc|lab-pvc-2)" && echo "✓ PVCs found" || echo "✗ PVCs missing"

echo
echo "3. Checking Applications..."
kubectl get deployments | grep -E "(storage-app|multi-volume-app)" && echo "✓ Applications found" || echo "✗ Applications missing"

echo
echo "4. Checking Data Persistence..."
STORAGE_POD=$(kubectl get pods -l app=storage-app -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ ! -z "$STORAGE_POD" ]; then
    if kubectl exec $STORAGE_POD -- test -f /data/timestamps.log 2>/dev/null; then
        LINE_COUNT=$(kubectl exec $STORAGE_POD -- wc -l /data/timestamps.log | awk '{print $1}')
        echo "✓ Data persistence verified - $LINE_COUNT log entries found"
    else
        echo "✗ Data persistence failed - log file not found"
    fi
else
    echo "✗ Storage app pod not found"
fi

echo
echo "5. Checking Multi-Volume Setup..."
MULTI_POD=$(kubectl get pods -l app=multi-volume-app -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ ! -z "$MULTI_POD" ]; then
    if kubectl exec $MULTI_POD -- test -f /data1/log1.txt 2>/dev/null && kubectl exec $MULTI_POD -- test -f /data2/log2.txt 2>/dev/null; then
        echo "✓ Multi-volume setup verified"
    else
        echo "✗ Multi-volume setup failed"
    fi
else
    echo "✗ Multi-volume app pod not found"
fi

echo
echo "=== Verification Complete ==="
EOF

chmod +x final-verification.sh
./final-verification.sh
Troubleshooting Common Issues
Issue 1: PVC Stuck in Pending State
Symptoms: PVC shows "Pending" status indefinitely

Diagnosis:

kubectl describe pvc <pvc-name>
kubectl get events --field-selector involvedObject.name=<pvc-name>
Common Causes and Solutions:

No matching PV available - Create PV with matching storage class and capacity
Access mode mismatch - Ensure PV and PVC have compatible access modes
Storage class not found - Verify storage class exists and is spelled correctly
Issue 2: Pod Cannot Mount Volume
Symptoms: Pod stuck in "ContainerCreating" state

Diagnosis:

kubectl describe pod <pod-name>
kubectl get events --field-selector involvedObject.name=<pod-name>
Common Solutions:

Check if PVC is bound to a PV
Verify volume mount paths in pod specification
Ensure node has necessary permissions for hostPath volumes
Issue 3: Data Not Persisting
Symptoms: Data disappears after pod restart

Diagnosis:

kubectl get pv,pvc
kubectl describe pv <pv-name>
Common Causes:

Using emptyDir instead of PVC
PV reclaim policy set to "Delete"
Volume not properly mounted in container
Conclusion
Congratulations! You have successfully completed Lab 11: Managing Kubernetes Volumes and Persistent Storage.

What You Accomplished
In this comprehensive lab, you have:

Created and Configured Persistent Storage

Set up PersistentVolumes with proper specifications
Created PersistentVolumeClaims to request storage resources
Understood the relationship between PVs, PVCs, and storage classes
Deployed Applications with Persistent Data

Deployed applications that write data to persistent volumes
Configured volume mounts and storage paths
Implemented resource limits and best practices
Verified Data Persistence

Tested data persistence across pod deletions and recreations
Verified that data survives application restarts
Demonstrated the core value of persistent storage in Kubernetes
Explored Advanced Storage Scenarios

Worked with multiple volumes in a single application
Understood storage allocation and management
Implemented monitoring and troubleshooting procedures
Applied Best Practices

Learned storage security considerations
Implemented proper cleanup procedures
Understood performance and scalability implications
Why This Matters
Persistent storage is crucial for real-world applications because:

Data Durability: Ensures critical application data survives container restarts and failures
Stateful Applications: Enables deployment of databases, file systems, and other stateful workloads
Business Continuity: Provides foundation for backup, disaster recovery, and data migration strategies
Scalability: Allows applications to scale while maintaining data consistency
Compliance: Meets regulatory requirements for data retention and security
Next Steps
To continue your Kubernetes journey:

Explore Dynamic Provisioning: Learn about StorageClasses and automatic PV provisioning
Study StatefulSets: Understand how to deploy stateful applications with ordered deployment
Implement Backup Strategies: Learn about volume snapshots and backup solutions
Security Hardening: Explore encryption at rest and access control for storage
Performance Optimization: Study different storage types and their performance characteristics
Real-World Applications
The skills you've learned apply directly to:

Database Deployments: PostgreSQL, MySQL, MongoDB in Kubernetes
Content Management: WordPress, Drupal, and other CMS platforms
Data Analytics: Persistent storage for data processing pipelines
File Sharing: Network-attached storage solutions
Backup Systems: Implementing enterprise backup and recovery solutions
You now have the foundational knowledge to manage persistent storage in production Kubernetes environments, making you well-prepared for the Kubernetes and Cloud Native Associate (KCNA) certification and real-world cloud-native application development.

Remember to clean up your lab resources when finished:

# Run the cleanup script if you want to remove all lab resources
./cleanup-storage.sh

# Or keep them for further experimentation and learning
Great job completing this hands-on lab! The persistent storage concepts you've mastered are essential for any serious Kubernetes deployment.
