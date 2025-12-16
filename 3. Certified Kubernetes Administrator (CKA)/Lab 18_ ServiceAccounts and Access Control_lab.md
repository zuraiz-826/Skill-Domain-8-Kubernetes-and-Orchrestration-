Lab 18: ServiceAccounts and Access Control
Objectives
By the end of this lab, you will be able to:

• Understand the concept of ServiceAccounts in Kubernetes and their role in security • Create and configure ServiceAccounts for specific applications • Implement Role-Based Access Control (RBAC) using Roles and RoleBindings • Assign appropriate permissions to ServiceAccounts using RBAC • Test and verify application access permissions • Troubleshoot common ServiceAccount and RBAC issues • Apply security best practices for Kubernetes access control

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with kubectl command-line tool • Knowledge of YAML file structure and syntax • Understanding of Linux command-line operations • Basic knowledge of security concepts and access control principles

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes manually.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-configured • Kubernetes cluster (single-node for learning purposes) • Text editors (nano, vim) for file editing • All necessary tools and dependencies pre-installed

Task 1: Understanding ServiceAccounts and Creating a Custom ServiceAccount
Subtask 1.1: Explore Default ServiceAccounts
First, let's examine the default ServiceAccounts that exist in your Kubernetes cluster.

List all ServiceAccounts in the default namespace:
kubectl get serviceaccounts
Get detailed information about the default ServiceAccount:
kubectl describe serviceaccount default
View ServiceAccounts across all namespaces:
kubectl get serviceaccounts --all-namespaces
Expected Output: You should see the default ServiceAccount and potentially others like those used by system components.

Subtask 1.2: Create a Custom ServiceAccount
Now we'll create a ServiceAccount specifically for our application.

Create a new ServiceAccount using kubectl:
kubectl create serviceaccount webapp-service-account
Verify the ServiceAccount was created:
kubectl get serviceaccount webapp-service-account
Get detailed information about your new ServiceAccount:
kubectl describe serviceaccount webapp-service-account
Subtask 1.3: Create ServiceAccount Using YAML Manifest
For better version control and documentation, let's also create a ServiceAccount using a YAML file.

Create a YAML file for the ServiceAccount:
nano database-service-account.yaml
Add the following content to the file:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database-service-account
  namespace: default
  labels:
    app: database
    tier: backend
automountServiceAccountToken: true
Apply the ServiceAccount manifest:
kubectl apply -f database-service-account.yaml
Verify both ServiceAccounts exist:
kubectl get serviceaccounts
Task 2: Implementing Role-Based Access Control (RBAC)
Subtask 2.1: Create a Role with Specific Permissions
We'll create a Role that defines what actions our ServiceAccount can perform.

Create a Role YAML file for web application permissions:
nano webapp-role.yaml
Add the following Role definition:
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: webapp-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
Apply the Role:
kubectl apply -f webapp-role.yaml
Verify the Role was created:
kubectl get roles
Subtask 2.2: Create a Role for Database Operations
Create another Role for database-related permissions:
nano database-role.yaml
Add the following Role definition:
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: database-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["get", "list", "watch"]
Apply the database Role:
kubectl apply -f database-role.yaml
List all Roles:
kubectl get roles
Subtask 2.3: Create RoleBindings to Connect ServiceAccounts with Roles
Now we'll bind our ServiceAccounts to their respective Roles.

Create a RoleBinding for the webapp ServiceAccount:
nano webapp-rolebinding.yaml
Add the following RoleBinding definition:
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: webapp-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: webapp-service-account
  namespace: default
roleRef:
  kind: Role
  name: webapp-role
  apiGroup: rbac.authorization.k8s.io
Apply the webapp RoleBinding:
kubectl apply -f webapp-rolebinding.yaml
Create a RoleBinding for the database ServiceAccount:
nano database-rolebinding.yaml
Add the following RoleBinding definition:
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: database-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: database-service-account
  namespace: default
roleRef:
  kind: Role
  name: database-role
  apiGroup: rbac.authorization.k8s.io
Apply the database RoleBinding:
kubectl apply -f database-rolebinding.yaml
Verify both RoleBindings were created:
kubectl get rolebindings
Task 3: Testing Application Access and Verifying Permissions
Subtask 3.1: Deploy Test Applications with ServiceAccounts
Let's create test applications that use our ServiceAccounts to verify the permissions work correctly.

Create a test webapp deployment:
nano webapp-deployment.yaml
Add the following deployment configuration:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      serviceAccountName: webapp-service-account
      containers:
      - name: webapp
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: SERVICE_ACCOUNT
          value: "webapp-service-account"
Apply the webapp deployment:
kubectl apply -f webapp-deployment.yaml
Create a test database deployment:
nano database-deployment.yaml
Add the following deployment configuration:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-deployment
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
    spec:
      serviceAccountName: database-service-account
      containers:
      - name: database
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: "testpassword"
        - name: SERVICE_ACCOUNT
          value: "database-service-account"
Apply the database deployment:
kubectl apply -f database-deployment.yaml
Verify both deployments are running:
kubectl get deployments
kubectl get pods
Subtask 3.2: Test ServiceAccount Permissions Using kubectl auth
We'll use kubectl's built-in authorization testing features to verify our RBAC setup.

Test webapp ServiceAccount permissions for pods:
kubectl auth can-i get pods --as=system:serviceaccount:default:webapp-service-account
Test webapp ServiceAccount permissions for secrets (should be denied):
kubectl auth can-i get secrets --as=system:serviceaccount:default:webapp-service-account
Test database ServiceAccount permissions for secrets:
kubectl auth can-i get secrets --as=system:serviceaccount:default:database-service-account
Test database ServiceAccount permissions for services (should be denied):
kubectl auth can-i get services --as=system:serviceaccount:default:database-service-account
Create a comprehensive permission test script:
nano test-permissions.sh
Add the following script content:
#!/bin/bash

echo "=== Testing webapp-service-account permissions ==="
echo "Can get pods: $(kubectl auth can-i get pods --as=system:serviceaccount:default:webapp-service-account)"
echo "Can list services: $(kubectl auth can-i list services --as=system:serviceaccount:default:webapp-service-account)"
echo "Can get secrets: $(kubectl auth can-i get secrets --as=system:serviceaccount:default:webapp-service-account)"
echo "Can create pods: $(kubectl auth can-i create pods --as=system:serviceaccount:default:webapp-service-account)"

echo ""
echo "=== Testing database-service-account permissions ==="
echo "Can get secrets: $(kubectl auth can-i get secrets --as=system:serviceaccount:default:database-service-account)"
echo "Can get pods: $(kubectl auth can-i get pods --as=system:serviceaccount:default:database-service-account)"
echo "Can create pods: $(kubectl auth can-i create pods --as=system:serviceaccount:default:database-service-account)"
echo "Can get services: $(kubectl auth can-i get services --as=system:serviceaccount:default:database-service-account)"
Make the script executable and run it:
chmod +x test-permissions.sh
./test-permissions.sh
Subtask 3.3: Test Permissions from Inside the Pods
Let's verify permissions from within the running containers.

Get the webapp pod name:
WEBAPP_POD=$(kubectl get pods -l app=webapp -o jsonpath='{.items[0].metadata.name}')
echo $WEBAPP_POD
Execute commands inside the webapp pod to test API access:
kubectl exec -it $WEBAPP_POD -- /bin/bash
Inside the pod, test API access (run these commands inside the pod):
# Install curl if not available
apt-get update && apt-get install -y curl

# Get the service account token
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Test getting pods (should work)
curl -H "Authorization: Bearer $TOKEN" \
     -k https://kubernetes.default.svc/api/v1/namespaces/default/pods

# Test getting secrets (should fail)
curl -H "Authorization: Bearer $TOKEN" \
     -k https://kubernetes.default.svc/api/v1/namespaces/default/secrets

# Exit the pod
exit
Test the database pod permissions:
DATABASE_POD=$(kubectl get pods -l app=database -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $DATABASE_POD -- /bin/bash
Inside the database pod, test API access:
# Install curl
apt-get update && apt-get install -y curl

# Get the service account token
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Test getting secrets (should work)
curl -H "Authorization: Bearer $TOKEN" \
     -k https://kubernetes.default.svc/api/v1/namespaces/default/secrets

# Test getting services (should fail)
curl -H "Authorization: Bearer $TOKEN" \
     -k https://kubernetes.default.svc/api/v1/namespaces/default/services

# Exit the pod
exit
Task 4: Advanced RBAC Configuration and ClusterRoles
Subtask 4.1: Create a ClusterRole for Cross-Namespace Access
Sometimes applications need access to resources across multiple namespaces. Let's create a ClusterRole for this purpose.

Create a ClusterRole YAML file:
nano monitoring-clusterrole.yaml
Add the following ClusterRole definition:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
Apply the ClusterRole:
kubectl apply -f monitoring-clusterrole.yaml
Create a ServiceAccount for monitoring:
kubectl create serviceaccount monitoring-service-account
Create a ClusterRoleBinding:
nano monitoring-clusterrolebinding.yaml
Add the ClusterRoleBinding definition:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: monitoring-service-account
  namespace: default
roleRef:
  kind: ClusterRole
  name: monitoring-clusterrole
  apiGroup: rbac.authorization.k8s.io
Apply the ClusterRoleBinding:
kubectl apply -f monitoring-clusterrolebinding.yaml
Subtask 4.2: Test ClusterRole Permissions
Test the monitoring ServiceAccount's cluster-wide permissions:
kubectl auth can-i get pods --all-namespaces --as=system:serviceaccount:default:monitoring-service-account
Test access to system namespaces:
kubectl auth can-i get pods --namespace=kube-system --as=system:serviceaccount:default:monitoring-service-account
Task 5: Security Best Practices and Cleanup
Subtask 5.1: Implement Security Best Practices
Create a ServiceAccount with minimal permissions:
nano minimal-service-account.yaml
Add a minimal ServiceAccount configuration:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: minimal-service-account
  namespace: default
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: minimal-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]
  resourceNames: ["specific-pod-name"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: minimal-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: minimal-service-account
  namespace: default
roleRef:
  kind: Role
  name: minimal-role
  apiGroup: rbac.authorization.k8s.io
Apply the minimal configuration:
kubectl apply -f minimal-service-account.yaml
Subtask 5.2: Audit and Review RBAC Configuration
List all ServiceAccounts, Roles, and RoleBindings:
echo "=== ServiceAccounts ==="
kubectl get serviceaccounts

echo "=== Roles ==="
kubectl get roles

echo "=== RoleBindings ==="
kubectl get rolebindings

echo "=== ClusterRoles (custom) ==="
kubectl get clusterroles | grep -v system:

echo "=== ClusterRoleBindings (custom) ==="
kubectl get clusterrolebindings | grep -v system:
Create an audit script:
nano audit-rbac.sh
Add the audit script content:
#!/bin/bash

echo "RBAC Audit Report"
echo "=================="
echo "Date: $(date)"
echo ""

echo "ServiceAccounts in default namespace:"
kubectl get serviceaccounts -o custom-columns=NAME:.metadata.name,SECRETS:.secrets[*].name

echo ""
echo "Roles and their permissions:"
for role in $(kubectl get roles -o jsonpath='{.items[*].metadata.name}'); do
    echo "Role: $role"
    kubectl describe role $role | grep -A 20 "Rules:"
    echo "---"
done

echo ""
echo "RoleBindings:"
kubectl get rolebindings -o custom-columns=NAME:.metadata.name,ROLE:.roleRef.name,SUBJECTS:.subjects[*].name
Make executable and run the audit:
chmod +x audit-rbac.sh
./audit-rbac.sh
Troubleshooting Common Issues
Issue 1: ServiceAccount Token Not Mounting
Problem: Applications can't access the Kubernetes API even with proper RBAC.

Solution: Check if automountServiceAccountToken is set to true:

kubectl get serviceaccount webapp-service-account -o yaml | grep automountServiceAccountToken
Issue 2: Permission Denied Errors
Problem: Getting 403 Forbidden errors when testing permissions.

Solution: Verify the RoleBinding connects the correct ServiceAccount to the correct Role:

kubectl describe rolebinding webapp-rolebinding
Issue 3: ClusterRole Not Working
Problem: ClusterRole permissions not taking effect.

Solution: Ensure you're using ClusterRoleBinding, not RoleBinding:

kubectl get clusterrolebindings monitoring-clusterrolebinding
Lab Cleanup
To clean up the resources created in this lab:

# Delete deployments
kubectl delete deployment webapp-deployment database-deployment

# Delete ServiceAccounts
kubectl delete serviceaccount webapp-service-account database-service-account monitoring-service-account minimal-service-account

# Delete Roles
kubectl delete role webapp-role database-role minimal-role

# Delete RoleBindings
kubectl delete rolebinding webapp-rolebinding database-rolebinding minimal-rolebinding

# Delete ClusterRole and ClusterRoleBinding
kubectl delete clusterrole monitoring-clusterrole
kubectl delete clusterrolebinding monitoring-clusterrolebinding

# Remove YAML files
rm -f *.yaml *.sh
Conclusion
In this comprehensive lab, you have successfully:

• Created custom ServiceAccounts for different application types, understanding how they provide identity for pods and applications running in Kubernetes

• Implemented Role-Based Access Control (RBAC) by creating Roles with specific permissions and binding them to ServiceAccounts through RoleBindings

• Tested and verified permissions using multiple methods including kubectl auth commands and direct API calls from within pods

• Explored advanced RBAC concepts including ClusterRoles and ClusterRoleBindings for cluster-wide permissions

• Applied security best practices by implementing the principle of least privilege and understanding how to audit RBAC configurations

Why This Matters: ServiceAccounts and RBAC are fundamental security mechanisms in Kubernetes that ensure applications have only the minimum permissions they need to function. This approach, known as the principle of least privilege, is crucial for maintaining a secure Kubernetes environment. In production environments, proper RBAC configuration prevents unauthorized access to sensitive resources and helps maintain compliance with security standards.

The skills you've learned in this lab are essential for the Certified Kubernetes Application Developer (CKAD) certification and are directly applicable to real-world Kubernetes deployments where security and access control are paramount concerns.

By mastering ServiceAccounts and RBAC, you're now equipped to design and implement secure, well-architected Kubernetes applications that follow industry best practices for access control and security.
