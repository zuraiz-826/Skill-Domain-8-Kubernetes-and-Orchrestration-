Lab 6: Hardening the Kubernetes Cluster Components
Objectives
By the end of this lab, students will be able to:

• Configure and harden the Kubernetes Controller Manager with restricted access and permissions • Implement security hardening for the Kubernetes Scheduler component • Enable and configure authentication mechanisms for Kubelet operations • Implement authorization controls for Kubelet access • Configure etcd data encryption at rest • Perform secure etcd backup and restore operations • Validate security configurations and test hardened cluster functionality • Apply security best practices for Kubernetes cluster components

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes architecture and components • Familiarity with Linux command line operations • Knowledge of YAML configuration files • Understanding of PKI (Public Key Infrastructure) concepts • Basic knowledge of systemd service management • Completion of previous Kubernetes security labs or equivalent experience

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 22.04 LTS with Kubernetes cluster pre-installed • All necessary tools and utilities pre-configured • Root access for system-level configurations • Network connectivity for testing

Task 1: Hardening the Controller Manager
Subtask 1.1: Analyze Current Controller Manager Configuration
First, let's examine the current Controller Manager configuration to understand what needs to be hardened.

Check the current Controller Manager status:
sudo systemctl status kube-controller-manager
Examine the Controller Manager configuration file:
sudo cat /etc/kubernetes/manifests/kube-controller-manager.yaml
View current Controller Manager logs:
sudo journalctl -u kube-controller-manager -n 50
Subtask 1.2: Configure Secure Communication
Create a dedicated directory for Controller Manager certificates:
sudo mkdir -p /etc/kubernetes/pki/controller-manager
sudo chmod 700 /etc/kubernetes/pki/controller-manager
Generate Controller Manager client certificate:
# Create private key
sudo openssl genrsa -out /etc/kubernetes/pki/controller-manager/controller-manager.key 2048

# Create certificate signing request
sudo openssl req -new -key /etc/kubernetes/pki/controller-manager/controller-manager.key \
  -out /etc/kubernetes/pki/controller-manager/controller-manager.csr \
  -subj "/CN=system:kube-controller-manager"

# Sign the certificate
sudo openssl x509 -req -in /etc/kubernetes/pki/controller-manager/controller-manager.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out /etc/kubernetes/pki/controller-manager/controller-manager.crt \
  -days 365
Set proper permissions on certificates:
sudo chmod 600 /etc/kubernetes/pki/controller-manager/controller-manager.key
sudo chmod 644 /etc/kubernetes/pki/controller-manager/controller-manager.crt
sudo chown root:root /etc/kubernetes/pki/controller-manager/*
Subtask 1.3: Update Controller Manager Configuration
Create a backup of the current configuration:
sudo cp /etc/kubernetes/manifests/kube-controller-manager.yaml \
  /etc/kubernetes/manifests/kube-controller-manager.yaml.backup
Create hardened Controller Manager configuration:
sudo tee /etc/kubernetes/manifests/kube-controller-manager.yaml > /dev/null <<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --use-service-account-credentials=true
    - --secure-port=10257
    - --port=0
    - --profiling=false
    - --terminated-pod-gc-threshold=10
    image: registry.k8s.io/kube-controller-manager:v1.28.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10257
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-controller-manager
    resources:
      requests:
        cpu: 200m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10257
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /etc/kubernetes/controller-manager.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
  hostNetwork: true
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /etc/kubernetes/controller-manager.conf
      type: FileOrCreate
    name: kubeconfig
  - hostPath:
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
status: {}
EOF
Wait for the Controller Manager to restart and verify:
# Wait for pod to restart
sleep 30

# Check if Controller Manager is running
kubectl get pods -n kube-system | grep controller-manager

# Verify the hardened configuration
kubectl logs -n kube-system kube-controller-manager-$(hostname) | tail -20
Task 2: Hardening the Scheduler
Subtask 2.1: Analyze Current Scheduler Configuration
Check current Scheduler status:
sudo systemctl status kube-scheduler
Examine current Scheduler configuration:
sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml
Subtask 2.2: Create Hardened Scheduler Configuration
Create backup of current Scheduler configuration:
sudo cp /etc/kubernetes/manifests/kube-scheduler.yaml \
  /etc/kubernetes/manifests/kube-scheduler.yaml.backup
Create hardened Scheduler configuration:
sudo tee /etc/kubernetes/manifests/kube-scheduler.yaml > /dev/null <<EOF
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --secure-port=10259
    - --port=0
    - --profiling=false
    - --logging-format=json
    - --v=2
    image: registry.k8s.io/kube-scheduler:v1.28.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
  hostNetwork: true
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
status: {}
EOF
Verify Scheduler restart and functionality:
# Wait for restart
sleep 30

# Check Scheduler pod status
kubectl get pods -n kube-system | grep scheduler

# Verify Scheduler logs
kubectl logs -n kube-system kube-scheduler-$(hostname) | tail -10
Task 3: Enable Authentication and Authorization for Kubelet
Subtask 3.1: Configure Kubelet Authentication
Check current Kubelet configuration:
sudo cat /var/lib/kubelet/config.yaml
Create enhanced Kubelet configuration with authentication:
sudo tee /var/lib/kubelet/config.yaml > /dev/null <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 2m0s
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 10s
evictionPressureTransitionPeriod: 5m0s
fileCheckFrequency: 20s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageMinimumGCAge: 2m0s
logging:
  flushFrequency: 5s
  options:
    json:
      infoBufferSize: "0"
  verbosity: 2
memorySwap: {}
nodeStatusReportFrequency: 10s
nodeStatusUpdateFrequency: 10s
rotateCertificates: true
runtimeRequestTimeout: 2m0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
serverTLSBootstrap: true
tlsCipherSuites:
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
- TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
protectKernelDefaults: true
readOnlyPort: 0
EOF
Subtask 3.2: Configure Kubelet Authorization
Create RBAC configuration for Kubelet:
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubelet-api-admin
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/status", "nodes/metrics", "nodes/proxy", "nodes/stats"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/status"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-api-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubelet-api-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kubelet-api
EOF
Restart Kubelet service:
sudo systemctl restart kubelet
Verify Kubelet authentication configuration:
# Check Kubelet status
sudo systemctl status kubelet

# Verify authentication is working
curl -k https://localhost:10250/metrics
Subtask 3.3: Test Kubelet Security
Test anonymous access (should be denied):
# This should fail with authentication error
curl -k https://localhost:10250/pods
Test with proper authentication:
# Create a test certificate for authentication
sudo openssl genrsa -out /tmp/kubelet-client.key 2048
sudo openssl req -new -key /tmp/kubelet-client.key \
  -out /tmp/kubelet-client.csr \
  -subj "/CN=kubelet-api"

sudo openssl x509 -req -in /tmp/kubelet-client.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out /tmp/kubelet-client.crt \
  -days 365

# Test with client certificate
curl -k --cert /tmp/kubelet-client.crt --key /tmp/kubelet-client.key \
  https://localhost:10250/metrics | head -10
Task 4: Encrypt etcd Data and Test Backup/Restore
Subtask 4.1: Configure etcd Encryption at Rest
Create encryption configuration:
# Generate encryption key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# Create encryption configuration file
sudo tee /etc/kubernetes/pki/encryption-config.yaml > /dev/null <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: ${ENCRYPTION_KEY}
  - identity: {}
EOF
Set proper permissions on encryption config:
sudo chmod 600 /etc/kubernetes/pki/encryption-config.yaml
sudo chown root:root /etc/kubernetes/pki/encryption-config.yaml
Update API server to use encryption:
# Backup current API server configuration
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml \
  /etc/kubernetes/manifests/kube-apiserver.yaml.backup

# Add encryption configuration to API server
sudo sed -i '/- --tls-private-key-file/a\    - --encryption-provider-config=/etc/kubernetes/pki/encryption-config.yaml' \
  /etc/kubernetes/manifests/kube-apiserver.yaml

# Add volume mount for encryption config
sudo sed -i '/name: k8s-certs/a\    - mountPath: /etc/kubernetes/pki/encryption-config.yaml\n      name: encryption-config\n      readOnly: true' \
  /etc/kubernetes/manifests/kube-apiserver.yaml

# Add volume for encryption config
sudo sed -i '/type: DirectoryOrCreate/a\  - hostPath:\n      path: /etc/kubernetes/pki/encryption-config.yaml\n      type: FileOrCreate\n    name: encryption-config' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
Wait for API server to restart and verify:
# Wait for API server restart
sleep 60

# Check API server status
kubectl get pods -n kube-system | grep apiserver

# Test encryption by creating a secret
kubectl create secret generic test-secret --from-literal=key=value

# Verify secret is encrypted in etcd
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C
Subtask 4.2: Configure etcd Backup
Create backup script:
sudo tee /usr/local/bin/etcd-backup.sh > /dev/null <<'EOF'
#!/bin/bash

# Set variables
BACKUP_DIR="/var/backups/etcd"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="etcd-backup-${DATE}.db"

# Create backup directory
mkdir -p ${BACKUP_DIR}

# Perform backup
ETCDCTL_API=3 etcdctl snapshot save ${BACKUP_DIR}/${BACKUP_FILE} \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status ${BACKUP_DIR}/${BACKUP_FILE} \
  --write-out=table

# Set permissions
chmod 600 ${BACKUP_DIR}/${BACKUP_FILE}

echo "Backup completed: ${BACKUP_DIR}/${BACKUP_FILE}"

# Clean up old backups (keep last 7 days)
find ${BACKUP_DIR} -name "etcd-backup-*.db" -mtime +7 -delete
EOF

sudo chmod +x /usr/local/bin/etcd-backup.sh
Create backup directory and perform initial backup:
sudo mkdir -p /var/backups/etcd
sudo /usr/local/bin/etcd-backup.sh
Verify backup was created:
sudo ls -la /var/backups/etcd/
Subtask 4.3: Test etcd Restore Process
Create test data before restore:
# Create some test resources
kubectl create namespace test-restore
kubectl create secret generic restore-test --from-literal=data=original -n test-restore
kubectl create configmap restore-test --from-literal=config=original -n test-restore
Perform backup with test data:
sudo /usr/local/bin/etcd-backup.sh
BACKUP_FILE=$(sudo ls -t /var/backups/etcd/ | head -1)
echo "Latest backup: $BACKUP_FILE"
Simulate data corruption by modifying data:
# Modify the test data
kubectl patch secret restore-test -n test-restore -p '{"data":{"data":"'$(echo -n "modified" | base64)'"}}'
kubectl patch configmap restore-test -n test-restore -p '{"data":{"config":"modified"}}'

# Verify changes
kubectl get secret restore-test -n test-restore -o yaml | grep data:
kubectl get configmap restore-test -n test-restore -o yaml | grep config:
Perform restore process:
# Stop etcd
sudo systemctl stop etcd

# Remove current etcd data
sudo rm -rf /var/lib/etcd

# Restore from backup
sudo ETCDCTL_API=3 etcdctl snapshot restore /var/backups/etcd/$BACKUP_FILE \
  --data-dir=/var/lib/etcd \
  --name=master \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# Set proper ownership
sudo chown -R etcd:etcd /var/lib/etcd

# Start etcd
sudo systemctl start etcd

# Wait for cluster to be ready
sleep 30
Verify restore was successful:
# Check if original data is restored
kubectl get secret restore-test -n test-restore -o yaml | grep data:
kubectl get configmap restore-test -n test-restore -o yaml | grep config:

# Verify cluster health
kubectl get nodes
kubectl get pods -n kube-system
Task 5: Validation and Testing
Subtask 5.1: Validate Controller Manager Hardening
Test Controller Manager security:
# Verify Controller Manager is not accessible on insecure port
curl -k http://localhost:10252/metrics 2>/dev/null || echo "Insecure port properly disabled"

# Verify secure port is working
curl -k https://localhost:10257/healthz
Check Controller Manager logs for security events:
kubectl logs -n kube-system kube-controller-manager-$(hostname) | grep -i "auth\|security\|tls"
Subtask 5.2: Validate Scheduler Hardening
Test Scheduler security:
# Verify Scheduler insecure port is disabled
curl -k http://localhost:10251/metrics 2>/dev/null || echo "Insecure port properly disabled"

# Verify secure port is working
curl -k https://localhost:10259/healthz
Subtask 5.3: Validate Kubelet Security
Test Kubelet authentication:
# Test anonymous access (should fail)
curl -k https://localhost:10250/pods 2>&1 | grep -i "unauthorized\|forbidden" && echo "Anonymous access properly blocked"

# Test with valid certificate
curl -k --cert /tmp/kubelet-client.crt --key /tmp/kubelet-client.key \
  https://localhost:10250/healthz
Subtask 5.4: Validate etcd Encryption
Test data encryption:
# Create a new secret
kubectl create secret generic encryption-test --from-literal=password=supersecret

# Verify it's encrypted in etcd
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/encryption-test | strings | grep -q "supersecret" && echo "Data NOT encrypted" || echo "Data properly encrypted"
Test backup and restore functionality:
# Perform final backup test
sudo /usr/local/bin/etcd-backup.sh

# Verify backup integrity
LATEST_BACKUP=$(sudo ls -t /var/backups/etcd/ | head -1)
sudo ETCDCTL_API=3 etcdctl snapshot status /var/backups/etcd/$LATEST_BACKUP --write-out=table
Troubleshooting Tips
Common Issues and Solutions
Issue: Controller Manager or Scheduler fails to start after hardening Solution:

# Check pod logs
kubectl logs -n kube-system kube-controller-manager-$(hostname)
kubectl logs -n kube-system kube-scheduler-$(hostname)

# Verify certificate permissions
sudo ls -la /etc/kubernetes/pki/
Issue: Kubelet authentication not working Solution:

# Check Kubelet logs
sudo journalctl -u kubelet -f

# Verify CA certificate
sudo openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout
Issue: etcd backup fails Solution:

# Check etcd status
sudo systemctl status etcd

# Verify etcd certificates
sudo ls -la /etc/kubernetes/pki/etcd/
Issue: API server fails to start with encryption Solution:

# Check API server logs
sudo journalctl -u kubelet | grep apiserver

# Verify encryption config syntax
sudo cat /etc/kubernetes/pki/encryption-config.yaml
Conclusion
In this lab, you have successfully implemented comprehensive security hardening for Kubernetes cluster components. Here's what you accomplished:

Controller Manager Hardening: You configured secure communication, disabled insecure ports, and implemented proper authentication and authorization mechanisms for the Controller Manager component.

Scheduler Hardening: You applied security configurations to the Scheduler, including disabling insecure ports, enabling secure communication, and implementing proper logging.

Kubelet Security: You enabled authentication and authorization for Kubelet operations, preventing anonymous access and ensuring only authorized users can interact with the Kubelet API.

etcd Encryption and Backup: You implemented encryption at rest for etcd data, ensuring sensitive information like secrets and configmaps are encrypted when stored. You also established a robust backup and restore process for disaster recovery.

Security Validation: You tested all security configurations to ensure they work as expected and provide the intended protection.

These hardening measures significantly improve your Kubernetes cluster's security posture by:

Preventing unauthorized access to cluster components
Encrypting sensitive data at rest
Ensuring secure communication between components
Providing reliable backup and recovery capabilities
Following security best practices for production environments
This knowledge is essential for the Kubernetes and Cloud Native Security Associate (KCSA) certification and for maintaining secure Kubernetes clusters in production environments. The skills you've learned will help you implement defense-in-depth strategies and protect your containerized applications from various security threats.
