Lab 15: Hardening API Server Security
Objectives
By the end of this lab, students will be able to:

• Configure and enable comprehensive API server audit logging to monitor and track all requests • Implement network policies to restrict unauthorized access to the Kubernetes API server • Simulate and test security measures by attempting unauthorized access scenarios • Analyze audit logs to identify potential security threats and access patterns • Apply security best practices for hardening Kubernetes API server configurations • Understand the importance of API server security in maintaining cluster integrity

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes architecture and components • Familiarity with kubectl command-line tool • Knowledge of YAML configuration files • Understanding of Linux command-line operations • Basic networking concepts (IP addresses, ports, firewalls) • Previous experience with Kubernetes pods, services, and deployments

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed. Simply click Start Lab to access your environment - no need to build your own VM or install software.

Your lab environment includes: • Ubuntu 20.04 LTS with Kubernetes cluster (1 master, 2 worker nodes) • kubectl configured and ready to use • All necessary tools pre-installed • Root access for system-level configurations

Task 1: Enable API Server Audit Logging
Subtask 1.1: Understanding Audit Logging
Audit logging in Kubernetes records all requests made to the API server, providing a security trail of who did what and when. This is crucial for: • Security monitoring and compliance • Troubleshooting access issues • Forensic analysis after security incidents

Subtask 1.2: Create Audit Policy Configuration
First, let's create a comprehensive audit policy that defines what events should be logged.

# Create directory for audit configurations
sudo mkdir -p /etc/kubernetes/audit

# Create the audit policy file
sudo tee /etc/kubernetes/audit/audit-policy.yaml > /dev/null << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all requests at RequestResponse level for secrets
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log all authentication and authorization failures
- level: Request
  namespaces: [""]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["pods", "services", "deployments"]
  omitStages:
  - RequestReceived

# Log all requests to system:masters group
- level: RequestResponse
  userGroups: ["system:masters"]

# Log all requests from service accounts in kube-system namespace
- level: Request
  namespaces: ["kube-system"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]

# Log metadata for all other requests
- level: Metadata
  omitStages:
  - RequestReceived
EOF
Subtask 1.3: Configure API Server for Audit Logging
Now we need to modify the API server configuration to enable audit logging.

# Backup the original API server manifest
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml.backup

# Create the modified API server configuration
sudo tee /etc/kubernetes/manifests/kube-apiserver.yaml > /dev/null << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.0.0.10:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.0.0.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    # Audit logging configuration
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --audit-policy-file=/etc/kubernetes/audit/audit-policy.yaml
    image: registry.k8s.io/kube-apiserver:v1.28.2
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 10.0.0.10
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-apiserver
    readinessProbe:
      failureThreshold: 3
      httpGet:
        host: 10.0.0.10
        path: /readyz
        port: 6443
        scheme: HTTPS
      periodSeconds: 1
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 250m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 10.0.0.10
        path: /livez
        port: 6443
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
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
    # Audit logging volume mounts
    - mountPath: /var/log/kubernetes
      name: audit-log
    - mountPath: /etc/kubernetes/audit
      name: audit-policy
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
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
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
  # Audit logging volumes
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log
  - hostPath:
      path: /etc/kubernetes/audit
      type: DirectoryOrCreate
    name: audit-policy
status: {}
EOF
Subtask 1.4: Create Log Directory and Restart API Server
# Create the audit log directory
sudo mkdir -p /var/log/kubernetes

# Set proper permissions
sudo chmod 755 /var/log/kubernetes

# Wait for the API server to restart (this happens automatically when the manifest changes)
echo "Waiting for API server to restart with audit logging enabled..."
sleep 30

# Verify the API server is running
kubectl get nodes
Subtask 1.5: Verify Audit Logging is Working
# Check if audit log file is created
ls -la /var/log/kubernetes/

# Generate some API activity to create audit entries
kubectl get pods --all-namespaces
kubectl create namespace audit-test
kubectl delete namespace audit-test

# Check the audit log content
sudo tail -10 /var/log/kubernetes/audit.log
Task 2: Restrict Access Using Network Policies
Subtask 2.1: Install and Configure Network Policy Controller
First, we need to install a network policy controller. We'll use Calico for this lab.

# Install Calico network policy controller
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# Wait for Calico pods to be ready
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=300s
kubectl wait --for=condition=ready pod -l k8s-app=calico-kube-controllers -n kube-system --timeout=300s

# Verify Calico installation
kubectl get pods -n kube-system | grep calico
Subtask 2.2: Create Test Applications
Let's create some test applications to demonstrate network policy restrictions.

# Create a test namespace
kubectl create namespace security-test

# Create a web application
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: security-test
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
        role: frontend
    spec:
      containers:
      - name: web
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: security-test
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Create a database application
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: security-test
  labels:
    app: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        role: backend
    spec:
      containers:
      - name: db
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: "testpassword"
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: security-test
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
EOF

# Wait for deployments to be ready
kubectl wait --for=condition=available deployment/web-app -n security-test --timeout=300s
kubectl wait --for=condition=available deployment/database -n security-test --timeout=300s
Subtask 2.3: Test Initial Connectivity
Before applying network policies, let's test that pods can communicate freely.

# Get pod information
kubectl get pods -n security-test -o wide

# Test connectivity from web-app to database
WEB_POD=$(kubectl get pods -n security-test -l app=web-app -o jsonpath='{.items[0].metadata.name}')
DB_SERVICE_IP=$(kubectl get service database-service -n security-test -o jsonpath='{.spec.clusterIP}')

echo "Testing connectivity from web-app to database..."
kubectl exec -n security-test $WEB_POD -- nc -zv $DB_SERVICE_IP 5432

# This should succeed, showing that connectivity is currently open
Subtask 2.4: Create Restrictive Network Policies
Now let's create network policies to restrict access.

# Create a default deny-all network policy
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: security-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Create a policy to allow web-app to access database
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
  namespace: security-test
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 5432
EOF

# Create a policy to allow web-app egress to database
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-egress
  namespace: security-test
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
EOF

# Allow external access to web-app (for testing)
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress
  namespace: security-test
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
EOF
Subtask 2.5: Verify Network Policy Enforcement
# Wait for network policies to take effect
sleep 10

# Test that web-app can still access database (should work)
echo "Testing allowed connectivity from web-app to database..."
kubectl exec -n security-test $WEB_POD -- nc -zv $DB_SERVICE_IP 5432

# Create a test pod that should NOT be able to access the database
kubectl run test-pod --image=busybox -n security-test --rm -it --restart=Never -- /bin/sh

# Inside the test pod, try to connect to the database (should fail)
# nc -zv database-service.security-test.svc.cluster.local 5432
# exit

# The connection should be blocked by the network policy
Subtask 2.6: Create API Server Access Restrictions
Now let's create network policies to restrict access to the API server itself.

# Create a namespace for privileged operations
kubectl create namespace privileged-access

# Create a network policy to restrict API server access
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-api-access
  namespace: privileged-access
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
  # Block API server access (port 6443) except for specific pods
  - to: []
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
EOF

# Create an allowed pod with special label
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: privileged-access
  labels:
    access: privileged
spec:
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["/bin/sleep", "3600"]
  serviceAccountName: default
EOF

# Create a restricted pod
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  namespace: privileged-access
  labels:
    access: restricted
spec:
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["/bin/sleep", "3600"]
  serviceAccountName: default
EOF
Task 3: Test Security by Simulating Unauthorized Access
Subtask 3.1: Create Test Service Accounts and RBAC
Let's create different service accounts with varying levels of access to test our security measures.

# Create service accounts for testing
kubectl create serviceaccount limited-user -n security-test
kubectl create serviceaccount admin-user -n security-test

# Create a limited role that can only read pods
kubectl apply -f - << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-test
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

# Create an admin role with broader permissions
kubectl apply -f - << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-test
  name: namespace-admin
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["*"]
  verbs: ["*"]
EOF

# Bind the roles to service accounts
kubectl create rolebinding limited-user-binding \
  --role=pod-reader \
  --serviceaccount=security-test:limited-user \
  -n security-test

kubectl create rolebinding admin-user-binding \
  --role=namespace-admin \
  --serviceaccount=security-test:admin-user \
  -n security-test
Subtask 3.2: Test Unauthorized API Access Attempts
# Get service account tokens for testing
LIMITED_TOKEN=$(kubectl create token limited-user -n security-test --duration=1h)
ADMIN_TOKEN=$(kubectl create token admin-user -n security-test --duration=1h)

# Test limited user access (should succeed for reading pods)
echo "Testing limited user access to read pods..."
kubectl get pods -n security-test --token=$LIMITED_TOKEN

# Test limited user trying to create resources (should fail)
echo "Testing limited user trying to create deployment (should fail)..."
kubectl create deployment test-deploy --image=nginx -n security-test --token=$LIMITED_TOKEN || echo "Access denied as expected"

# Test admin user access (should succeed)
echo "Testing admin user access..."
kubectl get pods -n security-test --token=$ADMIN_TOKEN
kubectl create deployment admin-test --image=nginx -n security-test --token=$ADMIN_TOKEN
kubectl delete deployment admin-test -n security-test --token=$ADMIN_TOKEN
Subtask 3.3: Simulate Network-Based Attacks
# Create a malicious pod that tries to access restricted resources
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: malicious-pod
  namespace: security-test
  labels:
    app: malicious
spec:
  containers:
  - name: attacker
    image: busybox
    command: ["/bin/sleep", "3600"]
EOF

# Wait for the pod to be ready
kubectl wait --for=condition=ready pod/malicious-pod -n security-test --timeout=60s

# Try to access the database from the malicious pod (should be blocked)
echo "Testing malicious pod trying to access database (should be blocked)..."
kubectl exec -n security-test malicious-pod -- nc -zv $DB_SERVICE_IP 5432 || echo "Connection blocked by network policy"

# Try to scan for other services (should be limited)
echo "Testing port scanning from malicious pod..."
kubectl exec -n security-test malicious-pod -- nc -zv web-service.security-test.svc.cluster.local 80 || echo "Connection blocked"
Subtask 3.4: Test API Server Audit Logging
# Generate various API activities to test audit logging
echo "Generating API activities for audit testing..."

# Successful operations
kubectl get secrets -n kube-system
kubectl create secret generic test-secret --from-literal=key=value -n security-test
kubectl get secret test-secret -n security-test

# Failed operations (using invalid tokens)
kubectl get pods --token="invalid-token" || echo "Authentication failed as expected"

# Operations with different service accounts
kubectl get pods -n security-test --token=$LIMITED_TOKEN
kubectl get secrets -n security-test --token=$LIMITED_TOKEN || echo "Authorization failed as expected"

# Check the audit logs for these activities
echo "Checking audit logs for recent activities..."
sudo tail -20 /var/log/kubernetes/audit.log | jq -r '.verb + " " + .objectRef.resource + " " + .user.username + " " + .responseStatus.code'
Subtask 3.5: Analyze Security Events
# Create a script to analyze audit logs
cat > analyze_audit.sh << 'EOF'
#!/bin/bash

echo "=== API Server Security Analysis ==="
echo

echo "1. Failed Authentication Attempts:"
sudo grep '"code":401' /var/log/kubernetes/audit.log | jq -r '.timestamp + " " + .user.username + " " + .verb + " " + .requestURI' | tail -5

echo
echo "2. Failed Authorization Attempts:"
sudo grep '"code":403' /var/log/kubernetes/audit.log | jq -r '.timestamp + " " + .user.username + " " + .verb + " " + .requestURI' | tail -5

echo
echo "3. Secret Access Attempts:"
sudo grep '"resource":"secrets"' /var/log/kubernetes/audit.log | jq -r '.timestamp + " " + .user.username + " " + .verb + " " + .objectRef.name' | tail -5

echo
echo "4. High-Privilege Operations:"
sudo grep 'system:masters' /var/log/kubernetes/audit.log | jq -r '.timestamp + " " + .verb + " " + .requestURI' | tail -5

echo
echo "5. Recent API Activity Summary:"
sudo tail -50 /var/log/kubernetes/audit.log | jq -r '.verb' | sort | uniq -c | sort -nr
EOF

chmod +x analyze_audit.sh
./analyze_audit.sh
Task 4: Advanced Security Hardening
Subtask 4.1: Implement Pod Security Standards
# Create a namespace with pod security standards
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
EOF

# Try to create a privileged pod (should fail)
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: secure-namespace
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF

# Create a compliant pod
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secure-namespace
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: test
    image: nginx:1.21
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}
EOF
Subtask 4.2: Configure API Server Rate Limiting
# Create an API priority and fairness configuration
kubectl apply -f - << 'EOF'
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: limited-priority
spec:
  matchingPrecedence: 1000
  priorityLevelConfiguration:
    name: limited-priority
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: limited-user
        namespace: security-test
    resourceRules:
    - verbs: ["*"]
      resources: ["*"]
      apiGroups: ["*"]
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: limited-priority
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 5
    limitResponse:
      type: Queue
      queuing:
        queues: 16
        handSize: 4
        queueLengthLimit: 50
EOF
Verification and Testing
Final Security Test
# Run a comprehensive security test
echo "=== Final Security Verification ==="

# 1. Test audit logging
echo "1. Verifying audit logging..."
kubectl get pods --all-namespaces > /dev/null
if sudo test -f /var/log/kubernetes/audit.log; then
    echo "✓ Audit logging is working"
else
    echo "✗ Audit logging failed"
fi

# 2. Test network policies
echo "2. Testing network policies..."
if kubectl exec -n security-test $WEB_POD -- nc -zv $DB_SERVICE_IP 5432 2>/dev/null; then
    echo "✓ Allowed network traffic works"
else
    echo "✗ Network policy blocking allowed traffic"
fi

# 3. Test RBAC
echo "3. Testing RBAC restrictions..."
if kubectl get pods -n security-test --token=$LIMITED_TOKEN > /dev/null 2>&1; then
    echo "✓ RBAC allows permitted operations"
else
    echo "✗ RBAC blocking permitted operations"
fi

if kubectl create deployment rbac-test --image=nginx -n security-test --token=$LIMITED_TOKEN 2>/dev/null; then
    echo "✗ RBAC not properly restricting operations"
    kubectl delete deployment rbac-test -n security-test
else
    echo "✓ RBAC properly restricting unauthorized operations"
fi

# 4. Test pod security standards
echo "4. Testing pod security standards..."
if kubectl get pod secure-pod -n secure-namespace > /dev/null 2>&1; then
    echo "✓ Pod security standards allow compliant pods"
else
    echo "✗ Pod security standards blocking compliant pods"
fi

echo
echo "Security hardening verification complete!"
Troubleshooting Common Issues
Issue 1: Audit Logs Not Generated
# Check if API server is running with audit configuration
kubectl get pods -n kube-system | grep kube-apiserver

# Check API server logs
sudo journalctl -u kubelet | grep apiserver

# Verify audit policy file exists and is readable
sudo ls -la /etc/kubernetes/audit/audit-policy.yaml
sudo cat /etc/kubernetes/audit/audit-policy.yaml
Issue 2: Network Policies Not Working
# Verify network policy controller is running
kubectl get pods -n kube-system | grep calico

# Check network policy status
kubectl get networkpolicies -n security-test

# Describe network policies for details
kubectl describe networkpolicy default-deny-all -n security-test
Issue 3: RBAC Issues
# Check service account tokens
kubectl get serviceaccounts -n security-test

# Verify role bindings
kubectl get rolebindings -n security-test

# Check permissions
kubectl auth can-i get pods --as=system:serviceaccount:security-test:limited-user -n security-test
Cleanup
# Clean up test resources
kubectl delete namespace security-test
kubectl delete namespace privileged-access
kubectl delete namespace secure-namespace

# Remove audit configuration (optional - keep for production use)
# sudo rm -rf /etc/kubernetes/audit/
# sudo rm -rf /var/log/kubernetes/

echo "Lab cleanup completed!"
Conclusion
In this comprehensive lab, you have successfully:

• Implemented comprehensive audit logging for the Kubernetes API server, enabling you to monitor and track all API requests for security and compliance purposes

• Applied network policies to restrict network traffic between pods and services, demonstrating how to implement microsegmentation in Kubernetes environments

• Tested security measures through simulated unauthorized access attempts, validating that your security controls are working effectively

• Configured RBAC policies to limit user and service account permissions, following the principle of least privilege

• Implemented Pod Security Standards to enforce security policies at the pod level, preventing the deployment of insecure workloads

• Analyzed audit logs to identify potential security threats and understand access patterns in your cluster

Why This Matters:

API server security is critical because the API server is the central control plane component that manages all cluster operations. A compromised API server can lead to complete cluster compromise, data breaches, and service disruptions. The security measures you've implemented in this lab represent industry best practices for:

Compliance requirements in regulated industries
Zero-trust security models in cloud-native environments
Incident response and forensics capabilities
Defense in depth security strategies
These skills are essential for the Kubernetes and Cloud Native Security Associate (KCSA) certification and are directly applicable to real-world production environments where security is paramount.

The combination of audit logging, network policies, RBAC, and pod security standards creates multiple layers of security that work together to protect your Kubernetes infrastructure from both external threats and insider risks.
