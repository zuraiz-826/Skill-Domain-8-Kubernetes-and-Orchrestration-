Lab 10: Network Policies for Pod Isolation
Objectives
By the end of this lab, you will be able to:

• Understand the concept of network policies in Kubernetes and their importance for pod security • Create pods with open network configurations and identify potential security risks • Define and implement NetworkPolicy resources to restrict incoming traffic between pods • Test network policies by attempting connections from different pods to verify isolation • Troubleshoot common network policy issues and validate policy effectiveness • Apply network segmentation best practices in Kubernetes environments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, services, namespaces) • Familiarity with kubectl command-line tool • Knowledge of YAML syntax and Kubernetes manifest files • Understanding of basic networking concepts (IP addresses, ports, protocols) • Experience with Linux command line and basic troubleshooting

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Kubernetes cluster with network policy support enabled • kubectl configured and ready to use • All necessary tools pre-installed • Sample applications for testing

Lab Tasks
Task 1: Create Pods with Open Network Configuration
Subtask 1.1: Set Up the Lab Environment
First, let's create a dedicated namespace for our network policy experiments and verify our cluster supports network policies.

# Create a new namespace for the lab
kubectl create namespace netpol-lab

# Set the namespace as default for this session
kubectl config set-context --current --namespace=netpol-lab

# Verify network policy support (should show a network plugin like Calico, Cilium, or Weave)
kubectl get nodes -o wide
kubectl get pods -n kube-system | grep -E "(calico|cilium|weave|flannel)"
Subtask 1.2: Create Frontend Application Pod
Create a frontend web application that will serve as our client pod.

# Create frontend pod manifest
cat << 'EOF' > frontend-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: netpol-lab
  labels:
    app: frontend
    tier: web
spec:
  containers:
  - name: frontend
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"
EOF

# Apply the frontend pod
kubectl apply -f frontend-pod.yaml

# Verify the pod is running
kubectl get pods -l app=frontend
Subtask 1.3: Create Backend Database Pod
Create a backend database pod that will be our target for network policy restrictions.

# Create backend pod manifest
cat << 'EOF' > backend-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: netpol-lab
  labels:
    app: backend
    tier: database
spec:
  containers:
  - name: backend
    image: postgres:13
    ports:
    - containerPort: 5432
    env:
    - name: POSTGRES_DB
      value: "testdb"
    - name: POSTGRES_USER
      value: "testuser"
    - name: POSTGRES_PASSWORD
      value: "testpass"
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
EOF

# Apply the backend pod
kubectl apply -f backend-pod.yaml

# Wait for the pod to be ready
kubectl wait --for=condition=Ready pod/backend --timeout=60s
Subtask 1.4: Create Test Client Pod
Create an additional test client pod to simulate unauthorized access attempts.

# Create test client pod manifest
cat << 'EOF' > test-client-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-client
  namespace: netpol-lab
  labels:
    app: test-client
    tier: testing
spec:
  containers:
  - name: test-client
    image: busybox:1.35
    command: ['sleep', '3600']
    resources:
      requests:
        memory: "32Mi"
        cpu: "25m"
      limits:
        memory: "64Mi"
        cpu: "50m"
EOF

# Apply the test client pod
kubectl apply -f test-client-pod.yaml

# Verify all pods are running
kubectl get pods -o wide
Subtask 1.5: Test Open Network Connectivity
Before implementing network policies, let's verify that all pods can communicate freely.

# Get pod IP addresses
kubectl get pods -o wide

# Test connectivity from frontend to backend (replace BACKEND_IP with actual IP)
BACKEND_IP=$(kubectl get pod backend -o jsonpath='{.status.podIP}')
echo "Backend IP: $BACKEND_IP"

# Test connection from frontend pod
kubectl exec frontend -- curl -m 5 http://$BACKEND_IP:5432 2>/dev/null && echo "Connection successful" || echo "Connection failed (expected for PostgreSQL)"

# Test connection from test-client pod
kubectl exec test-client -- nc -zv $BACKEND_IP 5432

# Test connection using telnet
kubectl exec test-client -- telnet $BACKEND_IP 5432 <<< "quit"
Task 2: Define NetworkPolicy to Restrict Incoming Traffic
Subtask 2.1: Create a Deny-All Network Policy
First, let's create a default deny-all policy to block all incoming traffic to our backend pod.

# Create deny-all network policy
cat << 'EOF' > deny-all-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-backend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  # No ingress rules defined = deny all ingress traffic
EOF

# Apply the deny-all policy
kubectl apply -f deny-all-netpol.yaml

# Verify the network policy was created
kubectl get networkpolicy
kubectl describe networkpolicy deny-all-backend
Subtask 2.2: Test Connectivity After Deny-All Policy
Now let's verify that the deny-all policy blocks all incoming connections to the backend.

# Test connection from frontend (should fail)
kubectl exec frontend -- timeout 5 bash -c "echo > /dev/tcp/$BACKEND_IP/5432" 2>/dev/null && echo "Connection successful" || echo "Connection blocked by network policy"

# Test connection from test-client (should fail)
kubectl exec test-client -- timeout 5 nc -zv $BACKEND_IP 5432 2>/dev/null && echo "Connection successful" || echo "Connection blocked by network policy"

# Verify the backend pod is still running
kubectl get pod backend
Subtask 2.3: Create Selective Allow Policy
Now let's create a more specific policy that allows only the frontend pod to connect to the backend.

# Remove the deny-all policy first
kubectl delete networkpolicy deny-all-backend

# Create selective allow policy
cat << 'EOF' > allow-frontend-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5432
EOF

# Apply the selective allow policy
kubectl apply -f allow-frontend-netpol.yaml

# Verify the policy
kubectl get networkpolicy
kubectl describe networkpolicy allow-frontend-to-backend
Subtask 2.4: Create Namespace-Based Network Policy
Let's also create a policy that demonstrates namespace-based restrictions.

# Create a second namespace for testing
kubectl create namespace external-ns

# Create a pod in the external namespace
cat << 'EOF' > external-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: external-client
  namespace: external-ns
  labels:
    app: external-client
spec:
  containers:
  - name: client
    image: busybox:1.35
    command: ['sleep', '3600']
EOF

kubectl apply -f external-pod.yaml

# Create namespace-restricted policy
cat << 'EOF' > namespace-restricted-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace-only
  namespace: netpol-lab
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
          name: netpol-lab
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5432
EOF

# Apply the namespace-restricted policy (this will replace the previous one)
kubectl apply -f namespace-restricted-netpol.yaml
Task 3: Test NetworkPolicy by Attempting Connections
Subtask 3.1: Test Allowed Connections
Let's verify that allowed connections still work after implementing our network policies.

# Test connection from frontend pod (should succeed)
echo "Testing connection from frontend pod..."
kubectl exec frontend -- timeout 10 bash -c "echo > /dev/tcp/$BACKEND_IP/5432" 2>/dev/null && echo "✓ Frontend can connect to backend" || echo "✗ Frontend cannot connect to backend"

# Alternative test using nc
kubectl exec frontend -- timeout 5 nc -zv $BACKEND_IP 5432 2>&1 | grep -q "open" && echo "✓ Frontend connection verified" || echo "✗ Frontend connection failed"
Subtask 3.2: Test Blocked Connections
Now let's verify that unauthorized connections are properly blocked.

# Test connection from test-client pod (should fail)
echo "Testing connection from test-client pod..."
kubectl exec test-client -- timeout 5 nc -zv $BACKEND_IP 5432 2>&1 | grep -q "open" && echo "✗ Test-client can connect (policy not working)" || echo "✓ Test-client blocked by network policy"

# Test connection from external namespace (should fail)
echo "Testing connection from external namespace..."
kubectl exec -n external-ns external-client -- timeout 5 nc -zv $BACKEND_IP 5432 2>&1 | grep -q "open" && echo "✗ External client can connect (policy not working)" || echo "✓ External client blocked by network policy"
Subtask 3.3: Comprehensive Connectivity Testing
Let's perform a comprehensive test of all connection scenarios.

# Create a test script for comprehensive testing
cat << 'EOF' > test-connectivity.sh
#!/bin/bash

BACKEND_IP=$(kubectl get pod backend -n netpol-lab -o jsonpath='{.status.podIP}')
echo "Backend IP: $BACKEND_IP"
echo "========================================="

# Test 1: Frontend to Backend (should work)
echo "Test 1: Frontend → Backend"
kubectl exec -n netpol-lab frontend -- timeout 5 nc -zv $BACKEND_IP 5432 2>/dev/null && echo "✓ PASS: Connection allowed" || echo "✗ FAIL: Connection blocked"

# Test 2: Test-client to Backend (should be blocked)
echo "Test 2: Test-client → Backend"
kubectl exec -n netpol-lab test-client -- timeout 5 nc -zv $BACKEND_IP 5432 2>/dev/null && echo "✗ FAIL: Connection allowed (should be blocked)" || echo "✓ PASS: Connection blocked"

# Test 3: External namespace to Backend (should be blocked)
echo "Test 3: External-client → Backend"
kubectl exec -n external-ns external-client -- timeout 5 nc -zv $BACKEND_IP 5432 2>/dev/null && echo "✗ FAIL: Connection allowed (should be blocked)" || echo "✓ PASS: Connection blocked"

# Test 4: Backend to Frontend (should work - no egress restrictions)
echo "Test 4: Backend → Frontend"
FRONTEND_IP=$(kubectl get pod frontend -n netpol-lab -o jsonpath='{.status.podIP}')
kubectl exec -n netpol-lab backend -- timeout 5 nc -zv $FRONTEND_IP 80 2>/dev/null && echo "✓ PASS: Connection allowed" || echo "✗ FAIL: Connection blocked"

echo "========================================="
echo "Network Policy Testing Complete"
EOF

# Make the script executable and run it
chmod +x test-connectivity.sh
./test-connectivity.sh
Subtask 3.4: Test Policy Modifications
Let's test how policy changes affect connectivity in real-time.

# Add test-client to allowed pods
cat << 'EOF' > updated-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-and-testclient
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - podSelector:
        matchLabels:
          app: test-client
    ports:
    - protocol: TCP
      port: 5432
EOF

# Apply the updated policy
kubectl apply -f updated-netpol.yaml

# Test connectivity again
echo "Testing after policy update..."
kubectl exec test-client -- timeout 5 nc -zv $BACKEND_IP 5432 2>&1 | grep -q "open" && echo "✓ Test-client now allowed" || echo "✗ Test-client still blocked"

# Revert to original policy
kubectl apply -f allow-frontend-netpol.yaml
echo "Reverted to original policy"
Subtask 3.5: Monitor Network Policy Effects
Let's examine how to monitor and troubleshoot network policies.

# Check network policy status
kubectl get networkpolicy -o wide

# Describe the network policy for detailed information
kubectl describe networkpolicy allow-frontend-to-backend

# Check pod labels to ensure they match policy selectors
kubectl get pods --show-labels

# Verify policy is applied to correct pods
kubectl get pods -l app=backend -o wide

# Check for any network policy related events
kubectl get events --field-selector reason=NetworkPolicyViolation
Advanced Testing and Troubleshooting
Troubleshooting Common Issues
Issue 1: Network Policies Not Working
# Check if your cluster supports network policies
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.containerRuntimeVersion}'

# Verify CNI plugin supports network policies
kubectl get pods -n kube-system | grep -E "(calico|cilium|weave)"

# Check for network policy controller logs
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50
Issue 2: Unexpected Connection Behavior
# Verify pod labels match policy selectors exactly
kubectl get pods --show-labels | grep -E "(frontend|backend)"

# Check if multiple network policies are conflicting
kubectl get networkpolicy -o yaml

# Test with verbose output
kubectl exec test-client -- nc -zv $BACKEND_IP 5432
Performance and Security Considerations
# Create a more complex policy with multiple rules
cat << 'EOF' > complex-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: complex-backend-policy
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: netpol-lab
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  - to: []  # Allow DNS
    ports:
    - protocol: UDP
      port: 53
EOF

# Apply complex policy
kubectl apply -f complex-netpol.yaml
Cleanup
# Remove all created resources
kubectl delete namespace netpol-lab
kubectl delete namespace external-ns

# Remove local files
rm -f *.yaml *.sh

# Reset kubectl context
kubectl config set-context --current --namespace=default
Conclusion
In this lab, you have successfully:

• Created pods with open network configurations and understood the security implications of unrestricted pod-to-pod communication • Implemented NetworkPolicy resources to control ingress traffic and learned how to use pod selectors and namespace selectors effectively • Tested network isolation by attempting connections from various pods and verified that policies work as expected • Explored advanced policy configurations including namespace-based restrictions and egress controls • Learned troubleshooting techniques for network policy issues and how to monitor policy effectiveness

Why This Matters:

Network policies are crucial for implementing zero-trust security in Kubernetes environments. By default, all pods in a cluster can communicate with each other, which poses significant security risks in production environments. Network policies allow you to:

Implement microsegmentation by isolating different application tiers
Reduce attack surface by limiting unnecessary network communications
Meet compliance requirements for data protection and network security
Follow security best practices by applying the principle of least privilege
This knowledge is essential for the CKAD certification and real-world Kubernetes deployments where security is paramount. Understanding network policies helps you design secure, scalable applications that follow cloud-native security principles.

The skills you've learned here directly apply to production scenarios where you need to protect sensitive workloads, implement multi-tenant architectures, and ensure regulatory compliance in containerized environments.
