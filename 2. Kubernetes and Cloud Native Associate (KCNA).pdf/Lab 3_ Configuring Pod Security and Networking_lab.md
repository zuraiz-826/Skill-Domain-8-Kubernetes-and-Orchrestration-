Lab 3: Configuring Pod Security and Networking
Objectives
By the end of this lab, you will be able to:

• Understand and implement Pod Security Admission (PSA) policies in Kubernetes • Configure Baseline and Restricted security policies for namespaces • Create and apply Network Policies to control pod-to-pod communication • Test network connectivity between pods before and after policy implementation • Troubleshoot common security and networking issues in Kubernetes environments • Demonstrate practical knowledge of Kubernetes security controls for KCSA certification

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, namespaces, services) • Familiarity with YAML syntax and kubectl commands • Knowledge of Linux command line operations • Understanding of basic networking concepts (IP addresses, ports, protocols)

Note: Al Nafi provides ready-to-use Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to begin - no need to build your own VM or install Kubernetes manually.

Lab Environment Setup
Your Al Nafi cloud machine comes with: • Kubernetes cluster (v1.28+) with Pod Security Admission enabled • kubectl configured and ready to use • All necessary tools pre-installed

Verify your environment by running:

kubectl version --short
kubectl get nodes
Task 1: Understanding and Configuring Pod Security Admission Policies
Subtask 1.1: Explore Current Pod Security Settings
First, let's examine the current security configuration of your cluster.

Check existing namespaces and their security labels:
kubectl get namespaces --show-labels
Create a test namespace to work with:
kubectl create namespace security-demo
Verify the namespace was created:
kubectl get namespace security-demo -o yaml
Subtask 1.2: Configure Baseline Pod Security Policy
The Baseline policy prevents the most common privilege escalations while maintaining compatibility with most workloads.

Apply Baseline policy to the security-demo namespace:
kubectl label namespace security-demo \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
Verify the labels were applied:
kubectl get namespace security-demo --show-labels
Create a test pod that should be allowed under Baseline policy:
cat << EOF > baseline-allowed-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-allowed
  namespace: security-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    securityContext:
      runAsNonRoot: false
      allowPrivilegeEscalation: false
EOF
Apply the pod configuration:
kubectl apply -f baseline-allowed-pod.yaml
Verify the pod was created successfully:
kubectl get pods -n security-demo
kubectl describe pod baseline-allowed -n security-demo
Subtask 1.3: Test Baseline Policy Restrictions
Now let's try to create a pod that violates the Baseline policy.

Create a pod that should be rejected by Baseline policy:
cat << EOF > baseline-rejected-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-rejected
  namespace: security-demo
spec:
  containers:
  - name: privileged-container
    image: nginx:1.21
    securityContext:
      privileged: true
      runAsUser: 0
EOF
Try to apply this pod (it should be rejected):
kubectl apply -f baseline-rejected-pod.yaml
You should see an error message indicating the pod violates the security policy.

Subtask 1.4: Configure Restricted Pod Security Policy
The Restricted policy implements the most restrictive security controls.

Create a new namespace for restricted testing:
kubectl create namespace restricted-demo
Apply Restricted policy to the namespace:
kubectl label namespace restricted-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
Create a pod that complies with Restricted policy:
cat << EOF > restricted-compliant-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-compliant
  namespace: restricted-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 8080
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      runAsGroup: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
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
Apply the restricted-compliant pod:
kubectl apply -f restricted-compliant-pod.yaml
Verify the pod is running:
kubectl get pods -n restricted-demo
kubectl describe pod restricted-compliant -n restricted-demo
Task 2: Creating and Applying Network Policies
Subtask 2.1: Prepare the Network Policy Environment
Create namespaces for network policy testing:
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database
Label the namespaces for easy identification:
kubectl label namespace frontend tier=frontend
kubectl label namespace backend tier=backend
kubectl label namespace database tier=database
Subtask 2.2: Deploy Test Applications
Deploy a frontend application:
cat << EOF > frontend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
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
  name: frontend-service
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Deploy a backend application:
cat << EOF > backend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Deploy a database application:
cat << EOF > database-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password123"
        ports:
        - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: database
spec:
  selector:
    app: database
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
EOF
Apply all deployments:
kubectl apply -f frontend-app.yaml
kubectl apply -f backend-app.yaml
kubectl apply -f database-app.yaml
Verify all pods are running:
kubectl get pods -n frontend
kubectl get pods -n backend
kubectl get pods -n database
Task 3: Testing Pod-to-Pod Connectivity Before Network Policies
Subtask 3.1: Test Initial Connectivity
Before applying network policies, let's verify that pods can communicate freely.

Get the IP addresses of services:
kubectl get services -n backend
kubectl get services -n database
Test connectivity from frontend to backend:
# Get a frontend pod name
FRONTEND_POD=$(kubectl get pods -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# Test connection to backend service
kubectl exec -n frontend $FRONTEND_POD -- curl -s --connect-timeout 5 backend-service.backend.svc.cluster.local
Test connectivity from backend to database:
# Get a backend pod name
BACKEND_POD=$(kubectl get pods -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}')

# Test connection to database service (using telnet to test port 3306)
kubectl exec -n backend $BACKEND_POD -- timeout 5 bash -c "</dev/tcp/database-service.database.svc.cluster.local/3306" && echo "Connection successful" || echo "Connection failed"
Test connectivity from frontend directly to database (this should work initially):
kubectl exec -n frontend $FRONTEND_POD -- timeout 5 bash -c "</dev/tcp/database-service.database.svc.cluster.local/3306" && echo "Connection successful" || echo "Connection failed"
Subtask 3.2: Document Initial Connectivity Results
Record your findings:

Frontend to Backend: Should work
Backend to Database: Should work
Frontend to Database: Should work (but we'll restrict this)
Task 4: Implementing Network Policies
Subtask 4.1: Create a Default Deny Network Policy
First, let's create a default deny policy for the database namespace to block all traffic.

Create a default deny policy for the database namespace:
cat << EOF > database-default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
Apply the default deny policy:
kubectl apply -f database-default-deny.yaml
Verify the network policy was created:
kubectl get networkpolicy -n database
kubectl describe networkpolicy default-deny-all -n database
Subtask 4.2: Create Specific Allow Policies
Now let's create policies that allow only the necessary traffic.

Allow backend to access database:
cat << EOF > allow-backend-to-database.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 3306
EOF
Allow database pods to make DNS queries (egress):
cat << EOF > allow-database-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-dns
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
Create a policy to allow frontend to backend communication:
cat << EOF > allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
EOF
Apply all network policies:
kubectl apply -f allow-backend-to-database.yaml
kubectl apply -f allow-database-dns.yaml
kubectl apply -f allow-frontend-to-backend.yaml
Verify all policies are created:
kubectl get networkpolicy -A
Task 5: Testing Connectivity After Network Policies
Subtask 5.1: Test Allowed Connections
Test frontend to backend (should still work):
FRONTEND_POD=$(kubectl get pods -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n frontend $FRONTEND_POD -- curl -s --connect-timeout 5 backend-service.backend.svc.cluster.local
Test backend to database (should still work):
BACKEND_POD=$(kubectl get pods -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n backend $BACKEND_POD -- timeout 5 bash -c "</dev/tcp/database-service.database.svc.cluster.local/3306" && echo "Connection successful" || echo "Connection failed"
Subtask 5.2: Test Blocked Connections
Test frontend to database (should now be blocked):
kubectl exec -n frontend $FRONTEND_POD -- timeout 5 bash -c "</dev/tcp/database-service.database.svc.cluster.local/3306" && echo "Connection successful" || echo "Connection failed"
This should fail, demonstrating that the network policy is working correctly.

Subtask 5.3: Verify Network Policy Effectiveness
Check network policy status:
kubectl describe networkpolicy -n database
kubectl describe networkpolicy -n backend
Create a test pod to verify isolation:
cat << EOF > test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-connectivity
  namespace: default
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
EOF
Apply and test from the default namespace:
kubectl apply -f test-pod.yaml
kubectl exec test-connectivity -- timeout 5 sh -c "nc -zv database-service.database.svc.cluster.local 3306" || echo "Connection blocked as expected"
Task 6: Advanced Network Policy Configuration
Subtask 6.1: Create Egress Policies
Restrict backend egress to only database:
cat << EOF > backend-egress-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-policy
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 3306
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
Apply the egress policy:
kubectl apply -f backend-egress-policy.yaml
Subtask 6.2: Monitor Network Policy Logs
Check for any policy violations in system logs:
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50 | grep -i "denied\|blocked" || echo "No blocked connections found in recent logs"
Troubleshooting Common Issues
Issue 1: Pod Security Admission Not Working
Symptoms: Pods that should be rejected are being created successfully.

Solution:

Verify PSA is enabled in your cluster:
kubectl get --raw /api/v1 | grep PodSecurity
Check namespace labels:
kubectl get namespace <namespace-name> --show-labels
Issue 2: Network Policies Not Blocking Traffic
Symptoms: Traffic flows even after applying deny policies.

Solution:

Verify your CNI supports Network Policies:
kubectl get pods -n kube-system | grep -E "(calico|cilium|weave)"
Check if policies are correctly applied:
kubectl describe networkpolicy <policy-name> -n <namespace>
Issue 3: DNS Resolution Issues
Symptoms: Pods cannot resolve service names after applying network policies.

Solution:

Ensure DNS egress is allowed:
# Add this to your network policy egress rules
- to: []
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
Cleanup
To clean up the lab environment:

# Delete test namespaces
kubectl delete namespace security-demo restricted-demo frontend backend database

# Delete test files
rm -f *.yaml

# Delete test pod from default namespace
kubectl delete pod test-connectivity --ignore-not-found
Conclusion
In this lab, you have successfully:

• Implemented Pod Security Admission policies by configuring both Baseline and Restricted security policies, demonstrating how to enforce security controls at the namespace level

• Created and tested Network Policies to control pod-to-pod communication, implementing a defense-in-depth security strategy

• Verified policy effectiveness by testing connectivity before and after policy implementation, proving that security controls are working as expected

• Gained hands-on experience with Kubernetes security features that are essential for the KCSA certification and real-world container security

Why This Matters: Pod Security Admission and Network Policies are fundamental security controls in Kubernetes environments. PSA helps prevent privilege escalation and ensures workloads run with appropriate security contexts, while Network Policies implement microsegmentation to limit the blast radius of potential security breaches. These skills are crucial for maintaining secure, compliant Kubernetes clusters in production environments.

Next Steps: Practice creating more complex network policies with multiple ingress and egress rules, experiment with different Pod Security Standards, and explore how these security controls integrate with other Kubernetes security features like RBAC and admission controllers.
