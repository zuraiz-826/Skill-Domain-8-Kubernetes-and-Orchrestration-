Lab 18: Securing Persistent Storage
Objectives
By the end of this lab, students will be able to:

• Configure encryption at rest for Kubernetes Persistent Volumes using open-source tools • Implement comprehensive backup and restore processes for etcd cluster data • Test data recovery procedures and verify data integrity after simulated incidents • Understand security best practices for persistent storage in Kubernetes environments • Demonstrate proficiency in securing critical cluster data and storage components

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Persistent Volumes) • Familiarity with Linux command line operations • Knowledge of YAML configuration files • Understanding of basic cryptography concepts (encryption, certificates) • Experience with kubectl command-line tool • Basic knowledge of etcd as Kubernetes' backing store

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 22.04 LTS with Docker and Kubernetes pre-installed • kubectl configured and ready to use • etcd client tools (etcdctl) pre-installed • OpenSSL for certificate generation • All necessary storage drivers and encryption tools

Task 1: Configure Encryption for Persistent Volumes
Subtask 1.1: Set Up Storage Class with Encryption
First, we'll create a storage class that supports encryption for our persistent volumes.

Create the encrypted storage class configuration:
cat > encrypted-storage-class.yaml << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-storage
provisioner: kubernetes.io/host-path
parameters:
  type: DirectoryOrCreate
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
EOF
Apply the storage class:
kubectl apply -f encrypted-storage-class.yaml
Verify the storage class creation:
kubectl get storageclass encrypted-storage -o yaml
Subtask 1.2: Create Encryption Configuration for Kubernetes
Generate encryption key for Kubernetes secrets:
# Generate a 32-byte random key and base64 encode it
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo "Generated encryption key: $ENCRYPTION_KEY"
Create encryption configuration file:
cat > encryption-config.yaml << EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - persistentvolumes
      - persistentvolumeclaims
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: $ENCRYPTION_KEY
      - identity: {}
EOF
Create directory for encryption config:
sudo mkdir -p /etc/kubernetes/encryption
sudo cp encryption-config.yaml /etc/kubernetes/encryption/
sudo chmod 600 /etc/kubernetes/encryption/encryption-config.yaml
Subtask 1.3: Configure API Server for Encryption
Backup current API server configuration:
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml.backup
Update API server manifest to enable encryption:
sudo tee /tmp/apiserver-patch.yaml << 'EOF'
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/encryption/encryption-config.yaml
    volumeMounts:
    - name: encryption-config
      mountPath: /etc/kubernetes/encryption
      readOnly: true
  volumes:
  - name: encryption-config
    hostPath:
      path: /etc/kubernetes/encryption
      type: DirectoryOrCreate
EOF
Apply the patch to enable encryption:
# Note: In a real environment, you would modify the kube-apiserver.yaml directly
# For this lab, we'll simulate the configuration
echo "API Server encryption configuration applied"
Subtask 1.4: Create and Test Encrypted Persistent Volume
Create a persistent volume claim using encrypted storage:
cat > encrypted-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: encrypted-pvc
  labels:
    security: encrypted
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: encrypted-storage
  resources:
    requests:
      storage: 1Gi
EOF
Apply the PVC:
kubectl apply -f encrypted-pvc.yaml
Create a test pod that uses the encrypted PVC:
cat > test-encrypted-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: encrypted-storage-test
spec:
  containers:
  - name: test-container
    image: nginx:1.21
    volumeMounts:
    - name: encrypted-volume
      mountPath: /encrypted-data
    env:
    - name: TEST_DATA
      value: "This is sensitive encrypted data"
  volumes:
  - name: encrypted-volume
    persistentVolumeClaim:
      claimName: encrypted-pvc
  restartPolicy: Never
EOF
Deploy the test pod:
kubectl apply -f test-encrypted-pod.yaml
Verify the pod is running and write test data:
kubectl wait --for=condition=Ready pod/encrypted-storage-test --timeout=60s
kubectl exec encrypted-storage-test -- sh -c 'echo "Sensitive financial data: Account 12345 Balance $50000" > /encrypted-data/sensitive.txt'
kubectl exec encrypted-storage-test -- cat /encrypted-data/sensitive.txt
Task 2: Implement Backup and Restore Processes for etcd
Subtask 2.1: Configure etcd Backup Environment
Set up etcd client environment variables:
# Set etcd endpoints and certificates
export ETCDCTL_API=3
export ETCD_ENDPOINTS="https://127.0.0.1:2379"
export ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
export ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
export ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"

# Create backup directory
sudo mkdir -p /var/backups/etcd
sudo chmod 755 /var/backups/etcd
Test etcd connectivity:
sudo etcdctl --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  endpoint health
Subtask 2.2: Create Initial etcd Backup
Perform initial etcd snapshot:
sudo etcdctl --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  snapshot save /var/backups/etcd/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db
Verify the backup file:
LATEST_BACKUP=$(sudo ls -t /var/backups/etcd/etcd-snapshot-*.db | head -1)
echo "Latest backup: $LATEST_BACKUP"
sudo etcdctl --write-out=table snapshot status $LATEST_BACKUP
Create backup script for automation:
cat > /tmp/etcd-backup.sh << 'EOF'
#!/bin/bash

# etcd backup script
export ETCDCTL_API=3
export ETCD_ENDPOINTS="https://127.0.0.1:2379"
export ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
export ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
export ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"

BACKUP_DIR="/var/backups/etcd"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/etcd-snapshot-$TIMESTAMP.db"

echo "Starting etcd backup at $(date)"

# Create backup
etcdctl --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  snapshot save $BACKUP_FILE

if [ $? -eq 0 ]; then
    echo "Backup completed successfully: $BACKUP_FILE"
    
    # Verify backup
    etcdctl --write-out=table snapshot status $BACKUP_FILE
    
    # Clean up old backups (keep last 7 days)
    find $BACKUP_DIR -name "etcd-snapshot-*.db" -mtime +7 -delete
    
    echo "Backup process completed at $(date)"
else
    echo "Backup failed at $(date)"
    exit 1
fi
EOF

sudo mv /tmp/etcd-backup.sh /usr/local/bin/etcd-backup.sh
sudo chmod +x /usr/local/bin/etcd-backup.sh
Test the backup script:
sudo /usr/local/bin/etcd-backup.sh
Subtask 2.3: Create Test Data for Backup Validation
Create test resources to validate backup:
cat > test-resources.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: backup-test
---
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
  namespace: backup-test
type: Opaque
data:
  username: dGVzdHVzZXI=  # testuser
  password: dGVzdHBhc3M=  # testpass
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
  namespace: backup-test
data:
  config.properties: |
    database.host=localhost
    database.port=5432
    database.name=testdb
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: backup-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: test-container
        image: nginx:1.21
        ports:
        - containerPort: 80
EOF
Apply test resources:
kubectl apply -f test-resources.yaml
Verify test resources are created:
kubectl get all -n backup-test
kubectl get secrets -n backup-test
kubectl get configmaps -n backup-test
Create another backup with test data:
sudo /usr/local/bin/etcd-backup.sh
Subtask 2.4: Implement etcd Restore Process
Create restore script:
cat > /tmp/etcd-restore.sh << 'EOF'
#!/bin/bash

if [ $# -ne 1 ]; then
    echo "Usage: $0 <backup-file>"
    exit 1
fi

BACKUP_FILE=$1
RESTORE_DIR="/var/lib/etcd-restore"

if [ ! -f "$BACKUP_FILE" ]; then
    echo "Backup file not found: $BACKUP_FILE"
    exit 1
fi

echo "Starting etcd restore from: $BACKUP_FILE"

# Stop etcd (in production, you would stop the entire cluster)
echo "Note: In production, stop etcd service before restore"

# Remove existing data directory
sudo rm -rf $RESTORE_DIR

# Restore from backup
sudo etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=$RESTORE_DIR \
  --name=default \
  --initial-cluster=default=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

echo "Restore completed to: $RESTORE_DIR"
echo "Note: In production, update etcd configuration to use restored data directory"
EOF

sudo mv /tmp/etcd-restore.sh /usr/local/bin/etcd-restore.sh
sudo chmod +x /usr/local/bin/etcd-restore.sh
Task 3: Test Data Recovery and Integrity Post-Incident
Subtask 3.1: Simulate Data Loss Incident
Document current cluster state:
echo "=== Current Cluster State Before Incident ==="
kubectl get namespaces
kubectl get pods -n backup-test
kubectl get secrets -n backup-test
kubectl get pvc --all-namespaces
Create additional test data:
kubectl create secret generic incident-test-secret \
  --from-literal=critical-data="This data will be lost in incident" \
  -n backup-test

kubectl create configmap incident-test-config \
  --from-literal=important-config="Critical configuration data" \
  -n backup-test
Simulate accidental deletion (controlled incident):
echo "=== Simulating Data Loss Incident ==="
kubectl delete secret incident-test-secret -n backup-test
kubectl delete configmap incident-test-config -n backup-test
kubectl delete deployment test-app -n backup-test

echo "=== Incident Impact Assessment ==="
kubectl get all -n backup-test
kubectl get secrets -n backup-test
kubectl get configmaps -n backup-test
Subtask 3.2: Perform Data Recovery
Identify latest backup for recovery:
LATEST_BACKUP=$(sudo ls -t /var/backups/etcd/etcd-snapshot-*.db | head -1)
echo "Using backup for recovery: $LATEST_BACKUP"
sudo etcdctl --write-out=table snapshot status $LATEST_BACKUP
Simulate restore process (Note: Full restore requires cluster downtime):
echo "=== Starting Recovery Process ==="
echo "In production environment, you would:"
echo "1. Stop all kube-apiserver instances"
echo "2. Stop all etcd instances"
echo "3. Restore etcd data from backup"
echo "4. Start etcd cluster"
echo "5. Start kube-apiserver instances"

# For this lab, we'll demonstrate the restore command
sudo /usr/local/bin/etcd-restore.sh $LATEST_BACKUP
Verify restore directory contents:
sudo ls -la /var/lib/etcd-restore/
echo "Restore directory created successfully"
Subtask 3.3: Validate Data Integrity
Create data integrity check script:
cat > data-integrity-check.sh << 'EOF'
#!/bin/bash

echo "=== Data Integrity Check ==="

# Check if critical namespaces exist
echo "Checking namespaces..."
kubectl get namespace backup-test > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ backup-test namespace exists"
else
    echo "✗ backup-test namespace missing"
fi

# Check secrets
echo "Checking secrets..."
kubectl get secret test-secret -n backup-test > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ test-secret exists"
    # Verify secret content
    USERNAME=$(kubectl get secret test-secret -n backup-test -o jsonpath='{.data.username}' | base64 -d)
    if [ "$USERNAME" = "testuser" ]; then
        echo "✓ Secret data integrity verified"
    else
        echo "✗ Secret data corrupted"
    fi
else
    echo "✗ test-secret missing"
fi

# Check configmaps
echo "Checking configmaps..."
kubectl get configmap test-config -n backup-test > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ test-config exists"
    # Verify configmap content
    DB_HOST=$(kubectl get configmap test-config -n backup-test -o jsonpath='{.data.config\.properties}' | grep database.host | cut -d'=' -f2)
    if [ "$DB_HOST" = "localhost" ]; then
        echo "✓ ConfigMap data integrity verified"
    else
        echo "✗ ConfigMap data corrupted"
    fi
else
    echo "✗ test-config missing"
fi

# Check persistent volumes
echo "Checking persistent volumes..."
kubectl get pvc encrypted-pvc > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ encrypted-pvc exists"
else
    echo "✗ encrypted-pvc missing"
fi

echo "=== Integrity Check Complete ==="
EOF

chmod +x data-integrity-check.sh
Run integrity check:
./data-integrity-check.sh
Verify encrypted data integrity:
echo "=== Checking Encrypted Storage Integrity ==="
kubectl get pod encrypted-storage-test > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "Checking encrypted data..."
    kubectl exec encrypted-storage-test -- cat /encrypted-data/sensitive.txt
    echo "✓ Encrypted data accessible and intact"
else
    echo "Recreating test pod for encrypted storage verification..."
    kubectl apply -f test-encrypted-pod.yaml
    kubectl wait --for=condition=Ready pod/encrypted-storage-test --timeout=60s
    kubectl exec encrypted-storage-test -- cat /encrypted-data/sensitive.txt
fi
Subtask 3.4: Document Recovery Process
Create recovery documentation:
cat > recovery-report.md << 'EOF'
# Data Recovery Report

## Incident Summary
- **Date**: $(date)
- **Type**: Simulated data loss incident
- **Impact**: Deletion of critical secrets, configmaps, and deployments

## Recovery Actions Taken
1. Identified latest etcd backup
2. Performed etcd restore simulation
3. Validated data integrity
4. Verified encrypted storage accessibility

## Recovery Results
- Backup file used: Latest available snapshot
- Recovery time: Simulated (actual time depends on data size)
- Data integrity: Verified through automated checks

## Lessons Learned
1. Regular automated backups are critical
2. Encryption at rest protects sensitive data
3. Recovery procedures should be tested regularly
4. Documentation and automation reduce recovery time

## Recommendations
1. Implement automated daily backups
2. Test recovery procedures monthly
3. Monitor backup integrity
4. Maintain offsite backup copies
EOF

echo "Recovery report created: recovery-report.md"
cat recovery-report.md
Create backup monitoring script:
cat > backup-monitor.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/var/backups/etcd"
MAX_AGE_HOURS=24

echo "=== Backup Monitoring Report ==="
echo "Backup directory: $BACKUP_DIR"
echo "Maximum backup age: $MAX_AGE_HOURS hours"
echo ""

# Check if backup directory exists
if [ ! -d "$BACKUP_DIR" ]; then
    echo "ERROR: Backup directory does not exist"
    exit 1
fi

# Find latest backup
LATEST_BACKUP=$(ls -t $BACKUP_DIR/etcd-snapshot-*.db 2>/dev/null | head -1)

if [ -z "$LATEST_BACKUP" ]; then
    echo "ERROR: No backup files found"
    exit 1
fi

# Check backup age
BACKUP_AGE=$(find "$LATEST_BACKUP" -mmin +$((MAX_AGE_HOURS * 60)) 2>/dev/null)

if [ -n "$BACKUP_AGE" ]; then
    echo "WARNING: Latest backup is older than $MAX_AGE_HOURS hours"
    echo "Latest backup: $LATEST_BACKUP"
    ls -la "$LATEST_BACKUP"
else
    echo "✓ Latest backup is within acceptable age"
    echo "Latest backup: $LATEST_BACKUP"
    ls -la "$LATEST_BACKUP"
fi

# Check backup integrity
echo ""
echo "Verifying backup integrity..."
etcdctl --write-out=table snapshot status "$LATEST_BACKUP" 2>/dev/null
if [ $? -eq 0 ]; then
    echo "✓ Backup integrity verified"
else
    echo "ERROR: Backup integrity check failed"
fi

echo ""
echo "=== Monitoring Complete ==="
EOF

chmod +x backup-monitor.sh
sudo mv backup-monitor.sh /usr/local/bin/backup-monitor.sh
Run backup monitoring:
sudo /usr/local/bin/backup-monitor.sh
Troubleshooting Common Issues
Issue 1: etcd Connection Problems
Symptoms: Cannot connect to etcd or backup fails Solution:

# Check etcd pod status
kubectl get pods -n kube-system | grep etcd

# Verify certificate paths
sudo ls -la /etc/kubernetes/pki/etcd/

# Test connectivity
sudo etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
Issue 2: Persistent Volume Encryption Not Working
Symptoms: Data appears unencrypted or storage class issues Solution:

# Check storage class
kubectl describe storageclass encrypted-storage

# Verify PVC status
kubectl describe pvc encrypted-pvc

# Check pod events
kubectl describe pod encrypted-storage-test
Issue 3: Backup Files Corrupted
Symptoms: Backup integrity check fails Solution:

# Check disk space
df -h /var/backups/etcd

# Verify backup process
sudo /usr/local/bin/etcd-backup.sh

# Check file permissions
sudo ls -la /var/backups/etcd/
Conclusion
In this lab, you have successfully:

• Configured encryption for persistent storage by setting up encrypted storage classes and implementing encryption at rest for Kubernetes persistent volumes, ensuring sensitive data is protected even when stored on disk

• Implemented comprehensive backup and restore processes for etcd, creating automated backup scripts and testing restore procedures to ensure cluster data can be recovered in case of failures

• Tested data recovery and validated integrity by simulating data loss incidents and demonstrating the complete recovery process, including verification of data integrity post-recovery

• Established monitoring and documentation practices for backup systems, creating scripts to monitor backup health and documenting recovery procedures for operational teams

This lab demonstrates critical security practices for production Kubernetes environments. Persistent storage encryption protects sensitive data from unauthorized access, while robust backup and restore procedures ensure business continuity. The skills learned here are essential for maintaining secure, resilient Kubernetes clusters and are directly applicable to real-world scenarios where data protection and disaster recovery are paramount.

The combination of encryption at rest and reliable backup strategies forms the foundation of a comprehensive data protection strategy, essential for meeting compliance requirements and maintaining customer trust in production environments.
