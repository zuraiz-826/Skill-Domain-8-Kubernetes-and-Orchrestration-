Lab 8: Implementing Security with Authentication, Authorization, and Admission Control
Objectives
By the end of this lab, you will be able to:

• Understand the fundamentals of Kubernetes security model including authentication, authorization, and admission control • Create and configure service accounts for applications and services • Implement Role-Based Access Control (RBAC) to manage permissions • Test access control mechanisms by attempting authorized and unauthorized actions • Configure admission controllers to enforce security policies and resource limits • Validate that security policies are working correctly in a Kubernetes cluster

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with command-line interface operations • Knowledge of YAML file structure and syntax • Understanding of Linux file permissions and user management concepts • Basic knowledge of kubectl commands

Note: Al Nafi provides ready-to-use Linux-based cloud machines with Kubernetes pre-installed. Simply click "Start Lab" to begin - no need to build your own VM or install Kubernetes manually.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with: • Kubernetes cluster (single-node for learning purposes) • kubectl command-line tool • Text editors (nano, vim) • All necessary permissions to complete this lab

Task 1: Understanding Kubernetes Security Architecture
Subtask 1.1: Explore Current Security Context
First, let's examine the current security context and understand what's already configured.

Check your current user context:
kubectl config current-context
View cluster information:
kubectl cluster-info
List existing service accounts:
kubectl get serviceaccounts --all-namespaces
Examine the default service account:
kubectl describe serviceaccount default
Subtask 1.2: Understand RBAC Components
List existing roles and cluster roles:
kubectl get roles --all-namespaces
kubectl get clusterroles
Examine a built-in cluster role:
kubectl describe clusterrole view
List role bindings:
kubectl get rolebindings --all-namespaces
kubectl get clusterrolebindings
Task 2: Create Service Accounts and Implement RBAC
Subtask 2.1: Create a Dedicated Namespace
Create a new namespace for our security lab:
kubectl create namespace security-lab
Set the namespace as default for this session:
kubectl config set-context --current --namespace=security-lab
Verify the namespace creation:
kubectl get namespaces
Subtask 2.2: Create Service Accounts
Create a service account for a developer role:
kubectl create serviceaccount developer-sa -n security-lab
Create a service account for a viewer role:
kubectl create serviceaccount viewer-sa -n security-lab
Create a service account for an admin role:
kubectl create serviceaccount admin-sa -n security-lab
Verify service account creation:
kubectl get serviceaccounts -n security-lab
Examine the developer service account details:
kubectl describe serviceaccount developer-sa -n security-lab
Subtask 2.3: Create Custom Roles
Create a developer role with specific permissions:
Create a file named developer-role.yaml:

cat > developer-role.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-lab
  name: developer-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
EOF
Apply the developer role:
kubectl apply -f developer-role.yaml
Create a viewer role with read-only permissions:
Create a file named viewer-role.yaml:

cat > viewer-role.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-lab
  name: viewer-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list"]
EOF
Apply the viewer role:
kubectl apply -f viewer-role.yaml
Verify role creation:
kubectl get roles -n security-lab
Subtask 2.4: Create Role Bindings
Bind the developer role to the developer service account:
Create a file named developer-rolebinding.yaml:

cat > developer-rolebinding.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: security-lab
subjects:
- kind: ServiceAccount
  name: developer-sa
  namespace: security-lab
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
EOF
Apply the developer role binding:
kubectl apply -f developer-rolebinding.yaml
Bind the viewer role to the viewer service account:
Create a file named viewer-rolebinding.yaml:

cat > viewer-rolebinding.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: viewer-binding
  namespace: security-lab
subjects:
- kind: ServiceAccount
  name: viewer-sa
  namespace: security-lab
roleRef:
  kind: Role
  name: viewer-role
  apiGroup: rbac.authorization.k8s.io
EOF
Apply the viewer role binding:
kubectl apply -f viewer-rolebinding.yaml
Bind the admin service account to cluster-admin role:
Create a file named admin-clusterrolebinding.yaml:

cat > admin-clusterrolebinding.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-binding
subjects:
- kind: ServiceAccount
  name: admin-sa
  namespace: security-lab
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
Apply the admin cluster role binding:
kubectl apply -f admin-clusterrolebinding.yaml
Verify all role bindings:
kubectl get rolebindings -n security-lab
kubectl get clusterrolebindings | grep admin-binding
Task 3: Test Access Control Mechanisms
Subtask 3.1: Create Test Resources
Create a test deployment for our access control tests:
Create a file named test-deployment.yaml:

cat > test-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: security-lab
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
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
EOF
Deploy the test application:
kubectl apply -f test-deployment.yaml
Verify the deployment:
kubectl get deployments -n security-lab
kubectl get pods -n security-lab
Subtask 3.2: Test Service Account Permissions
Get service account tokens for testing:
# Get the developer service account token
DEVELOPER_TOKEN=$(kubectl create token developer-sa -n security-lab)
echo "Developer token: $DEVELOPER_TOKEN"

# Get the viewer service account token
VIEWER_TOKEN=$(kubectl create token viewer-sa -n security-lab)
echo "Viewer token: $VIEWER_TOKEN"

# Get the admin service account token
ADMIN_TOKEN=$(kubectl create token admin-sa -n security-lab)
echo "Admin token: $ADMIN_TOKEN"
Test developer permissions (should succeed):
# Test listing pods as developer
kubectl --token=$DEVELOPER_TOKEN get pods -n security-lab

# Test creating a configmap as developer
kubectl --token=$DEVELOPER_TOKEN create configmap test-config --from-literal=key1=value1 -n security-lab

# Test scaling deployment as developer
kubectl --token=$DEVELOPER_TOKEN scale deployment test-app --replicas=3 -n security-lab
Test viewer permissions (read operations should succeed):
# Test listing pods as viewer (should work)
kubectl --token=$VIEWER_TOKEN get pods -n security-lab

# Test listing deployments as viewer (should work)
kubectl --token=$VIEWER_TOKEN get deployments -n security-lab
Test viewer unauthorized actions (should fail):
# Test creating a configmap as viewer (should fail)
kubectl --token=$VIEWER_TOKEN create configmap viewer-test --from-literal=key1=value1 -n security-lab

# Test deleting a pod as viewer (should fail)
kubectl --token=$VIEWER_TOKEN delete pod $(kubectl get pods -n security-lab -o jsonpath='{.items[0].metadata.name}') -n security-lab
Test admin permissions (should succeed for everything):
# Test cluster-wide operations as admin
kubectl --token=$ADMIN_TOKEN get nodes

# Test creating resources in any namespace
kubectl --token=$ADMIN_TOKEN create configmap admin-test --from-literal=admin=true -n default
Subtask 3.3: Verify Access Control Results
Check what resources were created by different service accounts:
kubectl get configmaps -n security-lab
kubectl get configmaps -n default
Examine the current state of our test deployment:
kubectl get deployment test-app -n security-lab
kubectl describe deployment test-app -n security-lab
Task 4: Configure Admission Controllers
Subtask 4.1: Understand Current Admission Controllers
Check which admission controllers are enabled:
kubectl describe pod kube-apiserver-$(hostname) -n kube-system | grep admission
Create a resource quota to demonstrate admission control:
Create a file named resource-quota.yaml:

cat > resource-quota.yaml << EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: security-lab-quota
  namespace: security-lab
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "10"
    services: "5"
    configmaps: "10"
EOF
Apply the resource quota:
kubectl apply -f resource-quota.yaml
Verify the resource quota:
kubectl describe resourcequota security-lab-quota -n security-lab
Subtask 4.2: Test Resource Quota Enforcement
Create a deployment that exceeds resource limits:
Create a file named resource-heavy-deployment.yaml:

cat > resource-heavy-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-heavy-app
  namespace: security-lab
spec:
  replicas: 5
  selector:
    matchLabels:
      app: resource-heavy-app
  template:
    metadata:
      labels:
        app: resource-heavy-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "1Gi"
            cpu: "1"
          limits:
            memory: "2Gi"
            cpu: "2"
EOF
Try to apply the resource-heavy deployment:
kubectl apply -f resource-heavy-deployment.yaml
Check the deployment status:
kubectl get deployment resource-heavy-app -n security-lab
kubectl describe deployment resource-heavy-app -n security-lab
Check resource quota usage:
kubectl describe resourcequota security-lab-quota -n security-lab
Subtask 4.3: Configure Network Policies (Additional Admission Control)
Create a network policy to restrict pod communication:
Create a file named network-policy.yaml: ```bash cat > network-policy.yaml << EOF apiVersion: networking.k8s.io/v1 kind: NetworkPolicy metadata: name: deny-all-ingress namespace: security-lab spec: podSelector: {} policyTypes: - Ingress
apiVersion: networking.k8s.io/v1 kind: NetworkPolicy metadata: name: allow-test-app-ingress namespace: security-lab spec: podSelector: matchLabels: app: test-app policyTypes:

Ingress ingress:
from:
podSelector: matchLabels: access: allowed
ports:

protocol: TCP port: 80 EOF

2. **Apply the network policies**:
```bash
kubectl apply -f network-policy.yaml
Verify network policies:
kubectl get networkpolicies -n security-lab
kubectl describe networkpolicy deny-all-ingress -n security-lab
Task 5: Validate Security Implementation
Subtask 5.1: Comprehensive Security Testing
Create a test pod to verify network policies:
Create a file named test-client.yaml:

cat > test-client.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-client
  namespace: security-lab
  labels:
    access: allowed
spec:
  containers:
  - name: client
    image: busybox
    command: ['sleep', '3600']
EOF
Apply the test client:
kubectl apply -f test-client.yaml
Test network connectivity:
# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/test-client -n security-lab --timeout=60s

# Get the IP of one of the test-app pods
TEST_APP_IP=$(kubectl get pod -l app=test-app -n security-lab -o jsonpath='{.items[0].status.podIP}')

# Test connection from allowed client (should work)
kubectl exec test-client -n security-lab -- wget -qO- --timeout=5 http://$TEST_APP_IP
Create an unauthorized client and test:
Create a file named unauthorized-client.yaml:

cat > unauthorized-client.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: unauthorized-client
  namespace: security-lab
spec:
  containers:
  - name: client
    image: busybox
    command: ['sleep', '3600']
EOF
Apply and test unauthorized client:
kubectl apply -f unauthorized-client.yaml

# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/unauthorized-client -n security-lab --timeout=60s

# Test connection from unauthorized client (should fail/timeout)
kubectl exec unauthorized-client -n security-lab -- timeout 10 wget -qO- http://$TEST_APP_IP || echo "Connection blocked by network policy"
Subtask 5.2: Security Audit and Verification
Review all security components created:
echo "=== Service Accounts ==="
kubectl get serviceaccounts -n security-lab

echo "=== Roles ==="
kubectl get roles -n security-lab

echo "=== Role Bindings ==="
kubectl get rolebindings -n security-lab

echo "=== Resource Quotas ==="
kubectl get resourcequotas -n security-lab

echo "=== Network Policies ==="
kubectl get networkpolicies -n security-lab
Test final access control scenarios:
# Test developer can still create resources within quota
kubectl --token=$DEVELOPER_TOKEN create configmap final-test --from-literal=test=passed -n security-lab

# Test viewer still cannot create resources
kubectl --token=$VIEWER_TOKEN create configmap viewer-final-test --from-literal=test=failed -n security-lab || echo "Access denied as expected"

# Check quota usage
kubectl describe resourcequota security-lab-quota -n security-lab
Generate a security summary report:
echo "=== SECURITY LAB SUMMARY REPORT ==="
echo "Namespace: security-lab"
echo "Service Accounts Created: $(kubectl get sa -n security-lab --no-headers | wc -l)"
echo "Roles Created: $(kubectl get roles -n security-lab --no-headers | wc -l)"
echo "Role Bindings Created: $(kubectl get rolebindings -n security-lab --no-headers | wc -l)"
echo "Network Policies Active: $(kubectl get networkpolicies -n security-lab --no-headers | wc -l)"
echo "Resource Quotas Enforced: $(kubectl get resourcequotas -n security-lab --no-headers | wc -l)"
echo "Pods Running: $(kubectl get pods -n security-lab --no-headers | grep Running | wc -l)"
Troubleshooting Common Issues
Issue 1: Service Account Token Not Working
Problem: Authentication fails with service account token Solution:

# Recreate the token
kubectl delete token <token-name> -n security-lab
kubectl create token <service-account-name> -n security-lab
Issue 2: RBAC Permissions Not Applied
Problem: Role bindings don't seem to work Solution:

# Check role binding details
kubectl describe rolebinding <binding-name> -n security-lab
# Verify the role exists
kubectl get role <role-name> -n security-lab
Issue 3: Resource Quota Not Enforcing
Problem: Pods are created despite quota limits Solution:

# Check quota status
kubectl describe resourcequota -n security-lab
# Ensure pods have resource requests/limits specified
Issue 4: Network Policy Not Working
Problem: Network policies don't block traffic Solution:

# Verify your cluster supports network policies
kubectl get pods -n kube-system | grep -i network
# Check if CNI plugin supports network policies
Cleanup Instructions
To clean up the lab environment:

# Delete the namespace (this removes all resources)
kubectl delete namespace security-lab

# Remove cluster role binding
kubectl delete clusterrolebinding admin-binding

# Reset kubectl context
kubectl config set-context --current --namespace=default
Conclusion
In this comprehensive lab, you have successfully implemented and tested Kubernetes security mechanisms including:

Authentication & Authorization: • Created multiple service accounts with different permission levels • Implemented Role-Based Access Control (RBAC) with custom roles • Tested access control by attempting both authorized and unauthorized actions • Verified that different service accounts have appropriate permissions

Admission Control: • Configured resource quotas to enforce resource limits • Implemented network policies to control pod-to-pod communication • Tested admission controllers by attempting to exceed defined limits • Validated that security policies are properly enforced

Key Security Concepts Learned: • Service Accounts provide identity for applications running in pods • RBAC enables fine-grained access control using roles and role bindings • Admission Controllers enforce policies and validate requests before resources are created • Resource Quotas prevent resource exhaustion and ensure fair resource allocation • Network Policies provide network-level security by controlling traffic flow

Why This Matters: Security is fundamental to any production Kubernetes environment. The skills you've learned in this lab are essential for: • Protecting sensitive applications and data • Implementing the principle of least privilege • Ensuring compliance with security standards • Preventing unauthorized access and resource abuse • Building secure, multi-tenant Kubernetes clusters

These security practices are critical for the Kubernetes and Cloud Native Associate (KCNA) certification and are fundamental skills for anyone working with Kubernetes in production environments. The hands-on experience gained from this lab provides practical knowledge that directly applies to real-world Kubernetes security implementations.

