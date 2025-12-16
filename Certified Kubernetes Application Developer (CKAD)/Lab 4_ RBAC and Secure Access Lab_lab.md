Lab 4: RBAC and Secure Access Lab
Objectives
By the end of this lab, you will be able to:

• Understand the fundamentals of Role-Based Access Control (RBAC) in Kubernetes • Create and configure Roles and ClusterRoles to define permissions • Implement RoleBindings and ClusterRoleBindings to assign permissions to users and service accounts • Configure service accounts for Pods to securely access cluster resources • Test and validate access permissions with different roles • Apply security best practices for namespace-level access control • Troubleshoot common RBAC configuration issues

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Namespaces) • Familiarity with kubectl command-line tool • Knowledge of YAML file structure and syntax • Understanding of Linux command-line operations • Completion of previous Kubernetes labs or equivalent experience

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-installed • Kubernetes cluster (single-node) ready for use • All necessary tools and utilities • Root access for administrative tasks

Lab Tasks Overview
This lab is divided into the following main tasks:

Setting Up the Lab Environment
Creating Namespaces and Basic Resources
Implementing Role-Based Access Control
Configuring Service Accounts
Testing Access Permissions
Advanced RBAC Scenarios
Task 1: Setting Up the Lab Environment
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure your Kubernetes cluster is running properly.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
Subtask 1.2: Create Working Directory
Create a dedicated directory for this lab's files.

# Create lab directory
mkdir ~/rbac-lab
cd ~/rbac-lab

# Create subdirectories for organization
mkdir manifests scripts
Task 2: Creating Namespaces and Basic Resources
Subtask 2.1: Create Multiple Namespaces
We'll create separate namespaces to demonstrate namespace-level access control.

# Create development namespace
kubectl create namespace development

# Create production namespace
kubectl create namespace production

# Create testing namespace
kubectl create namespace testing

# Verify namespaces
kubectl get namespaces
Subtask 2.2: Deploy Sample Applications
Create sample applications in different namespaces for testing purposes.

Create the development application manifest:

cat > manifests/dev-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-app
  namespace: development
  labels:
    app: dev-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dev-app
  template:
    metadata:
      labels:
        app: dev-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: dev-app-service
  namespace: development
spec:
  selector:
    app: dev-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Create the production application manifest:

cat > manifests/prod-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  namespace: production
  labels:
    app: prod-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: prod-app-service
  namespace: production
spec:
  selector:
    app: prod-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Deploy the applications:

# Deploy development application
kubectl apply -f manifests/dev-app.yaml

# Deploy production application
kubectl apply -f manifests/prod-app.yaml

# Verify deployments
kubectl get deployments -n development
kubectl get deployments -n production
Task 3: Implementing Role-Based Access Control
Subtask 3.1: Create Service Accounts
Service accounts provide an identity for processes running in Pods.

# Create service account for development team
kubectl create serviceaccount dev-team -n development

# Create service account for production team
kubectl create serviceaccount prod-team -n production

# Create service account for read-only access
kubectl create serviceaccount readonly-user -n testing

# Verify service accounts
kubectl get serviceaccounts -n development
kubectl get serviceaccounts -n production
kubectl get serviceaccounts -n testing
Subtask 3.2: Create Roles for Namespace-Level Permissions
Roles define what actions can be performed within a specific namespace.

Create a role for development team with full access to development namespace:

cat > manifests/dev-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: dev-full-access
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF
Create a role for production team with limited access:

cat > manifests/prod-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: prod-limited-access
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
EOF
Create a read-only role:

cat > manifests/readonly-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: testing
  name: readonly-access
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
EOF
Apply the roles:

# Apply all roles
kubectl apply -f manifests/dev-role.yaml
kubectl apply -f manifests/prod-role.yaml
kubectl apply -f manifests/readonly-role.yaml

# Verify roles
kubectl get roles -n development
kubectl get roles -n production
kubectl get roles -n testing
Subtask 3.3: Create RoleBindings
RoleBindings grant the permissions defined in roles to users or service accounts.

Create RoleBinding for development team:

cat > manifests/dev-rolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: dev-team
  namespace: development
roleRef:
  kind: Role
  name: dev-full-access
  apiGroup: rbac.authorization.k8s.io
EOF
Create RoleBinding for production team:

cat > manifests/prod-rolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prod-team-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: prod-team
  namespace: production
roleRef:
  kind: Role
  name: prod-limited-access
  apiGroup: rbac.authorization.k8s.io
EOF
Create RoleBinding for read-only user:

cat > manifests/readonly-rolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-binding
  namespace: testing
subjects:
- kind: ServiceAccount
  name: readonly-user
  namespace: testing
roleRef:
  kind: Role
  name: readonly-access
  apiGroup: rbac.authorization.k8s.io
EOF
Apply the RoleBindings:

# Apply all RoleBindings
kubectl apply -f manifests/dev-rolebinding.yaml
kubectl apply -f manifests/prod-rolebinding.yaml
kubectl apply -f manifests/readonly-rolebinding.yaml

# Verify RoleBindings
kubectl get rolebindings -n development
kubectl get rolebindings -n production
kubectl get rolebindings -n testing
Subtask 3.4: Create ClusterRole for Cross-Namespace Access
ClusterRoles define permissions across the entire cluster or multiple namespaces.

cat > manifests/cluster-viewer-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
- apiGroups: [""]
  resources: ["namespaces", "nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
  resourceNames: []
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EOF
Create ClusterRoleBinding:

cat > manifests/cluster-viewer-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-viewer-binding
subjects:
- kind: ServiceAccount
  name: readonly-user
  namespace: testing
roleRef:
  kind: ClusterRole
  name: cluster-viewer
  apiGroup: rbac.authorization.k8s.io
EOF
Apply the ClusterRole and ClusterRoleBinding:

# Apply ClusterRole and ClusterRoleBinding
kubectl apply -f manifests/cluster-viewer-role.yaml
kubectl apply -f manifests/cluster-viewer-binding.yaml

# Verify ClusterRole and ClusterRoleBinding
kubectl get clusterroles | grep cluster-viewer
kubectl get clusterrolebindings | grep cluster-viewer
Task 4: Configuring Service Accounts for Pods
Subtask 4.1: Create Pods with Specific Service Accounts
Create a Pod that uses the development service account:

cat > manifests/dev-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dev-pod
  namespace: development
spec:
  serviceAccountName: dev-team
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
  restartPolicy: Never
EOF
Create a Pod that uses the production service account:

cat > manifests/prod-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: prod-pod
  namespace: production
spec:
  serviceAccountName: prod-team
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
  restartPolicy: Never
EOF
Create a Pod that uses the read-only service account:

cat > manifests/readonly-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: readonly-pod
  namespace: testing
spec:
  serviceAccountName: readonly-user
  containers:
  - name: kubectl-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
  restartPolicy: Never
EOF
Deploy the Pods:

# Deploy all Pods
kubectl apply -f manifests/dev-pod.yaml
kubectl apply -f manifests/prod-pod.yaml
kubectl apply -f manifests/readonly-pod.yaml

# Wait for Pods to be ready
kubectl wait --for=condition=Ready pod/dev-pod -n development --timeout=60s
kubectl wait --for=condition=Ready pod/prod-pod -n production --timeout=60s
kubectl wait --for=condition=Ready pod/readonly-pod -n testing --timeout=60s

# Verify Pods are running
kubectl get pods -n development
kubectl get pods -n production
kubectl get pods -n testing
Task 5: Testing Access Permissions
Subtask 5.1: Test Development Team Permissions
Test the development team's full access to the development namespace:

# Test creating a ConfigMap (should succeed)
kubectl exec -it dev-pod -n development -- kubectl create configmap test-config --from-literal=key=value -n development

# Test listing pods (should succeed)
kubectl exec -it dev-pod -n development -- kubectl get pods -n development

# Test creating a deployment (should succeed)
kubectl exec -it dev-pod -n development -- kubectl create deployment test-deploy --image=nginx -n development

# Test accessing production namespace (should fail)
kubectl exec -it dev-pod -n development -- kubectl get pods -n production

# Clean up test resources
kubectl delete configmap test-config -n development
kubectl delete deployment test-deploy -n development
Subtask 5.2: Test Production Team Permissions
Test the production team's limited access:

# Test listing pods (should succeed)
kubectl exec -it prod-pod -n production -- kubectl get pods -n production

# Test viewing deployments (should succeed)
kubectl exec -it prod-pod -n production -- kubectl get deployments -n production

# Test creating a secret (should fail - not in role)
kubectl exec -it prod-pod -n production -- kubectl create secret generic test-secret --from-literal=password=secret123 -n production

# Test deleting a deployment (should fail - delete not allowed)
kubectl exec -it prod-pod -n production -- kubectl delete deployment prod-app -n production

# Test accessing development namespace (should fail)
kubectl exec -it prod-pod -n production -- kubectl get pods -n development
Subtask 5.3: Test Read-Only Permissions
Test the read-only user's permissions:

# Test listing pods in testing namespace (should succeed)
kubectl exec -it readonly-pod -n testing -- kubectl get pods -n testing

# Test listing namespaces (should succeed due to ClusterRole)
kubectl exec -it readonly-pod -n testing -- kubectl get namespaces

# Test creating a pod (should fail)
kubectl exec -it readonly-pod -n testing -- kubectl run test-pod --image=nginx -n testing

# Test listing nodes (should succeed due to ClusterRole)
kubectl exec -it readonly-pod -n testing -- kubectl get nodes
Subtask 5.4: Create Testing Script
Create a comprehensive testing script to validate all permissions:

cat > scripts/test-permissions.sh << 'EOF'
#!/bin/bash

echo "=== RBAC Permission Testing ==="
echo

# Test Development Team Permissions
echo "Testing Development Team Permissions:"
echo "1. Creating ConfigMap in development namespace..."
kubectl exec -it dev-pod -n development -- kubectl create configmap rbac-test --from-literal=test=value -n development
if [ $? -eq 0 ]; then
    echo "   ✓ SUCCESS: Can create ConfigMap"
    kubectl delete configmap rbac-test -n development
else
    echo "   ✗ FAILED: Cannot create ConfigMap"
fi

echo "2. Trying to access production namespace..."
kubectl exec -it dev-pod -n development -- kubectl get pods -n production 2>/dev/null
if [ $? -eq 0 ]; then
    echo "   ✗ SECURITY ISSUE: Can access production namespace"
else
    echo "   ✓ SUCCESS: Cannot access production namespace"
fi

echo

# Test Production Team Permissions
echo "Testing Production Team Permissions:"
echo "1. Listing pods in production namespace..."
kubectl exec -it prod-pod -n production -- kubectl get pods -n production >/dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "   ✓ SUCCESS: Can list pods"
else
    echo "   ✗ FAILED: Cannot list pods"
fi

echo "2. Trying to create secret..."
kubectl exec -it prod-pod -n production -- kubectl create secret generic test-secret --from-literal=key=value -n production 2>/dev/null
if [ $? -eq 0 ]; then
    echo "   ✗ SECURITY ISSUE: Can create secrets (should not be allowed)"
    kubectl delete secret test-secret -n production
else
    echo "   ✓ SUCCESS: Cannot create secrets"
fi

echo

# Test Read-Only Permissions
echo "Testing Read-Only Permissions:"
echo "1. Listing namespaces..."
kubectl exec -it readonly-pod -n testing -- kubectl get namespaces >/dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "   ✓ SUCCESS: Can list namespaces"
else
    echo "   ✗ FAILED: Cannot list namespaces"
fi

echo "2. Trying to create pod..."
kubectl exec -it readonly-pod -n testing -- kubectl run unauthorized-pod --image=nginx -n testing 2>/dev/null
if [ $? -eq 0 ]; then
    echo "   ✗ SECURITY ISSUE: Can create pods (should not be allowed)"
    kubectl delete pod unauthorized-pod -n testing
else
    echo "   ✓ SUCCESS: Cannot create pods"
fi

echo
echo "=== Testing Complete ==="
EOF

# Make script executable
chmod +x scripts/test-permissions.sh

# Run the testing script
./scripts/test-permissions.sh
Task 6: Advanced RBAC Scenarios
Subtask 6.1: Create Resource-Specific Permissions
Create a role that only allows access to specific resources by name:

cat > manifests/specific-resource-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: specific-resource-access
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
  resourceNames: ["dev-app-*"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
  resourceNames: ["app-config", "database-config"]
EOF
Subtask 6.2: Create Time-Based Access Control
Create a script that demonstrates temporary access:

cat > scripts/temporary-access.sh << 'EOF'
#!/bin/bash

echo "Creating temporary admin access..."

# Create temporary service account
kubectl create serviceaccount temp-admin -n development

# Create temporary RoleBinding with admin privileges
cat << 'YAML' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: temp-admin-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: temp-admin
  namespace: development
roleRef:
  kind: Role
  name: dev-full-access
  apiGroup: rbac.authorization.k8s.io
YAML

echo "Temporary access granted for 60 seconds..."
sleep 60

echo "Removing temporary access..."
kubectl delete rolebinding temp-admin-binding -n development
kubectl delete serviceaccount temp-admin -n development

echo "Temporary access revoked."
EOF

chmod +x scripts/temporary-access.sh
Subtask 6.3: Implement Network Policy Integration
Create a NetworkPolicy that works with RBAC:

cat > manifests/network-policy.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: development-network-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: dev-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: development
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: development
    ports:
    - protocol: TCP
      port: 80
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
EOF

# Apply network policy
kubectl apply -f manifests/network-policy.yaml
Task 7: Monitoring and Auditing RBAC
Subtask 7.1: Check Current Permissions
Create a script to audit current permissions:

cat > scripts/audit-permissions.sh << 'EOF'
#!/bin/bash

echo "=== RBAC Audit Report ==="
echo

echo "Service Accounts:"
kubectl get serviceaccounts --all-namespaces

echo
echo "Roles:"
kubectl get roles --all-namespaces

echo
echo "RoleBindings:"
kubectl get rolebindings --all-namespaces

echo
echo "ClusterRoles (custom):"
kubectl get clusterroles | grep -v "system:"

echo
echo "ClusterRoleBindings (custom):"
kubectl get clusterrolebindings | grep -v "system:"

echo
echo "=== Detailed Permission Analysis ==="

# Check what the dev-team service account can do
echo
echo "Development Team Permissions:"
kubectl auth can-i --list --as=system:serviceaccount:development:dev-team -n development

echo
echo "Production Team Permissions:"
kubectl auth can-i --list --as=system:serviceaccount:production:prod-team -n production

echo
echo "Read-Only User Permissions:"
kubectl auth can-i --list --as=system:serviceaccount:testing:readonly-user -n testing
EOF

chmod +x scripts/audit-permissions.sh
./scripts/audit-permissions.sh
Subtask 7.2: Test Specific Permissions
Use kubectl auth can-i to test specific permissions:

# Test if dev-team can create pods in development namespace
kubectl auth can-i create pods --as=system:serviceaccount:development:dev-team -n development

# Test if prod-team can delete deployments in production namespace
kubectl auth can-i delete deployments --as=system:serviceaccount:production:prod-team -n production

# Test if readonly-user can create secrets in testing namespace
kubectl auth can-i create secrets --as=system:serviceaccount:testing:readonly-user -n testing

# Test cross-namespace access
kubectl auth can-i get pods --as=system:serviceaccount:development:dev-team -n production
Troubleshooting Common Issues
Issue 1: Permission Denied Errors
If you encounter permission denied errors:

# Check if the service account exists
kubectl get serviceaccount <service-account-name> -n <namespace>

# Check if the role exists
kubectl get role <role-name> -n <namespace>

# Check if the rolebinding exists and is correct
kubectl describe rolebinding <rolebinding-name> -n <namespace>

# Verify the pod is using the correct service account
kubectl get pod <pod-name> -n <namespace> -o yaml | grep serviceAccount
Issue 2: ClusterRole Not Working
If ClusterRole permissions are not working:

# Check if ClusterRole exists
kubectl get clusterrole <clusterrole-name>

# Check ClusterRoleBinding
kubectl describe clusterrolebinding <clusterrolebinding-name>

# Verify the subject in ClusterRoleBinding matches your service account
kubectl get clusterrolebinding <clusterrolebinding-name> -o yaml
Issue 3: Pod Cannot Access API Server
If pods cannot access the Kubernetes API:

# Check if service account token is mounted
kubectl exec -it <pod-name> -n <namespace> -- ls -la /var/run/secrets/kubernetes.io/serviceaccount/

# Check if API server is accessible from pod
kubectl exec -it <pod-name> -n <namespace> -- curl -k https://kubernetes.default.svc.cluster.local/api/v1/namespaces
Cleanup
Remove All Lab Resources
Create a cleanup script:

cat > scripts/cleanup.sh << 'EOF'
#!/bin/bash

echo "Cleaning up RBAC lab resources..."

# Delete pods
kubectl delete pod dev-pod -n development --ignore-not-found=true
kubectl delete pod prod-pod -n production --ignore-not-found=true
kubectl delete pod readonly-pod -n testing --ignore-not-found=true

# Delete applications
kubectl delete -f manifests/dev-app.yaml --ignore-not-found=true
kubectl delete -f manifests/prod-app.yaml --ignore-not-found=true

# Delete RBAC resources
kubectl delete rolebinding dev-team-binding -n development --ignore-not-found=true
kubectl delete rolebinding prod-team-binding -n production --ignore-not-found=true
kubectl delete rolebinding readonly-binding -n testing --ignore-not-found=true
kubectl delete clusterrolebinding cluster-viewer-binding --ignore-not-found=true

kubectl delete role dev-full-access -n development --ignore-not-found=true
kubectl delete role prod-limited-access -n production --ignore-not-found=true
kubectl delete role readonly-access -n testing --ignore-not-found=true
kubectl delete clusterrole cluster-viewer --ignore-not-found=true

# Delete service accounts
kubectl delete serviceaccount dev-team -n development --ignore-not-found=true
kubectl delete serviceaccount prod-team -n production --ignore-not-found=true
kubectl delete serviceaccount readonly-user -n testing --ignore-not-found=true

# Delete network policy
kubectl delete networkpolicy development-network-policy -n development --ignore-not-found=true

# Delete namespaces (this will delete everything in them)
kubectl delete namespace development --ignore-not-found=true
kubectl delete namespace production --ignore-not-found=true
kubectl delete namespace testing --ignore-not-found=true

echo "Cleanup complete!"
EOF

chmod +x scripts/cleanup.sh
Run cleanup when ready:

./scripts/cleanup.sh
Conclusion
Congratulations! You have successfully completed the RBAC and Secure Access Lab. In this comprehensive lab, you have:

Key Accomplishments:

• Implemented Role-Based Access Control: Created Roles and ClusterRoles to define granular permissions for different user groups and service accounts

• Configured Namespace-Level Security: Set up isolated environments with specific access controls for development, production, and testing namespaces

• Created Service Accounts: Established secure identities for applications running in Pods with appropriate permissions

• Applied Security Best Practices: Implemented the principle of least privilege by granting only necessary permissions to each role

• Tested Access Controls: Validated that RBAC policies work correctly and prevent unauthorized access across namespaces

• Monitored and Audited Permissions: Used kubectl commands to verify and audit current permission assignments

Why This Matters:

RBAC is crucial for production Kubernetes environments because it:

Enhances Security: Prevents unauthorized access to sensitive resources and data
Enables Multi-Tenancy: Allows multiple teams to share a cluster safely with isolated access
Supports Compliance: Helps meet regulatory requirements for access control and auditing
Reduces Risk: Minimizes the impact of compromised accounts through limited permissions
Improves Operational Efficiency: Enables teams to work independently without interfering with each other
Real-World Applications:

The skills you've learned apply directly to:

Setting up development, staging, and production environments
Managing access for different teams (developers, operators, security teams)
Implementing CI/CD pipelines with appropriate service account permissions
Securing microservices architectures with fine-grained access controls
Meeting enterprise security and compliance requirements
Next Steps:

To further enhance your Kubernetes security knowledge, consider exploring:

Pod Security Standards and Pod Security Policies
Network Policies for traffic segmentation
Secrets management and encryption at rest
Integration with external identity providers (LDAP, Active Directory)
Advanced auditing and monitoring solutions
This lab has provided you with a solid foundation in Kubernetes RBAC that will be essential for the Certified Kubernetes Administrator (CKA) certification and real-world Kubernetes deployments.
