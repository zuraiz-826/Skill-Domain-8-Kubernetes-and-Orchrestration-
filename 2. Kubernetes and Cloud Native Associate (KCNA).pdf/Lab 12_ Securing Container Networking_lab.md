Lab 12: Securing Container Networking
Objectives
By the end of this lab, students will be able to:

• Understand the fundamentals of Kubernetes Network Policies and their role in container security • Configure and implement Network Policies to control Pod-to-Pod communication • Test and validate network traffic restrictions using practical scenarios • Simulate network-based attacks and demonstrate how Network Policies provide mitigation • Apply security best practices for container networking in production environments • Troubleshoot common Network Policy configuration issues

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Namespaces) • Familiarity with YAML configuration files • Basic knowledge of networking concepts (IP addresses, ports, protocols) • Experience with command-line interface operations • Understanding of container security fundamentals

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes cluster already set up. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Multi-node Kubernetes cluster with Network Policy support • kubectl command-line tool pre-installed and configured • All necessary networking components and CNI plugins • Sample applications for testing purposes

Task 1: Understanding and Configuring Kubernetes Network Policies
Subtask 1.1: Explore the Current Network Environment
First, let's examine the existing cluster setup and understand the default networking behavior.

Check cluster nodes and networking setup:
# Verify cluster status
kubectl get nodes -o wide

# Check the CNI plugin being used
kubectl get pods -n kube-system | grep -E "(calico|flannel|weave|cilium)"

# List all namespaces
kubectl get namespaces
Create test namespaces for our lab:
# Create development namespace
kubectl create namespace development

# Create production namespace
kubectl create namespace production

# Create testing namespace
kubectl create namespace testing

# Verify namespaces creation
kubectl get namespaces
Subtask 1.2: Deploy Test Applications
Now we'll deploy sample applications across different namespaces to test network policies.

Deploy a web application in the development namespace:
# Create a simple nginx deployment
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: development
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
  name: web-app-service
  namespace: development
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Deploy a database application in the production namespace:
# Create a simple database deployment
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: production
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
        tier: backend
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "securepassword123"
        - name: MYSQL_DATABASE
          value: "testdb"
        ports:
        - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: production
spec:
  selector:
    app: database
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
EOF
Deploy a client application in the testing namespace:
# Create a client pod for testing connectivity
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
  namespace: testing
  labels:
    app: client
    role: tester
spec:
  containers:
  - name: client
    image: busybox:1.35
    command: ['sleep', '3600']
EOF
Verify all deployments are running:
# Check development namespace
kubectl get pods -n development -o wide

# Check production namespace
kubectl get pods -n production -o wide

# Check testing namespace
kubectl get pods -n testing -o wide

# Get all services
kubectl get services --all-namespaces
Subtask 1.3: Test Default Network Connectivity
Before implementing Network Policies, let's test the default connectivity between pods.

Test connectivity from client pod to web application:
# Get the web-app service IP
WEB_SERVICE_IP=$(kubectl get service web-app-service -n development -o jsonpath='{.spec.clusterIP}')
echo "Web Service IP: $WEB_SERVICE_IP"

# Test HTTP connectivity from client pod
kubectl exec -it client-pod -n testing -- wget -qO- --timeout=5 http://$WEB_SERVICE_IP
Test connectivity from client pod to database:
# Get the database service IP
DB_SERVICE_IP=$(kubectl get service database-service -n production -o jsonpath='{.spec.clusterIP}')
echo "Database Service IP: $DB_SERVICE_IP"

# Test TCP connectivity to database port
kubectl exec -it client-pod -n testing -- nc -zv $DB_SERVICE_IP 3306
Test cross-namespace pod-to-pod connectivity:
# Get a pod IP from development namespace
DEV_POD_IP=$(kubectl get pod -n development -l app=web-app -o jsonpath='{.items[0].status.podIP}')
echo "Development Pod IP: $DEV_POD_IP"

# Test direct pod connectivity
kubectl exec -it client-pod -n testing -- ping -c 3 $DEV_POD_IP
Task 2: Implementing Network Policies for Traffic Control
Subtask 2.1: Create a Deny-All Default Policy
We'll start by implementing a default deny-all policy to establish a secure baseline.

Create a default deny-all ingress policy for the production namespace:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
Create a default deny-all egress policy for the production namespace:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
EOF
Verify the policies are created:
# List network policies in production namespace
kubectl get networkpolicies -n production

# Describe the policies to see details
kubectl describe networkpolicy default-deny-all-ingress -n production
kubectl describe networkpolicy default-deny-all-egress -n production
Subtask 2.2: Test the Impact of Deny-All Policies
Now let's test how the deny-all policies affect connectivity.

Test connectivity to the database after applying deny-all policy:
# This should now fail due to the deny-all ingress policy
kubectl exec -it client-pod -n testing -- nc -zv $DB_SERVICE_IP 3306 || echo "Connection blocked by Network Policy"
Test connectivity from within the production namespace:
# Create a temporary pod in production namespace for testing
kubectl run temp-client -n production --image=busybox:1.35 --rm -it --restart=Never -- sh

# Inside the pod, try to connect to the database (this should also fail due to egress policy)
# nc -zv database-service.production.svc.cluster.local 3306
# exit
Subtask 2.3: Create Selective Allow Policies
Now we'll create specific policies to allow only necessary traffic.

Allow ingress traffic to database from development namespace only:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dev-to-database
  namespace: production
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
          name: development
    ports:
    - protocol: TCP
      port: 3306
EOF
Label the development namespace for the policy to work:
# Add label to development namespace
kubectl label namespace development name=development

# Verify the label
kubectl get namespace development --show-labels
Create an egress policy for database pods to allow DNS resolution:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-egress-dns
  namespace: production
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
Create a policy to allow web-app to communicate with database:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-database
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
      podSelector:
        matchLabels:
          app: database
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
Task 3: Testing Pod Communication Restrictions
Subtask 3.1: Verify Allowed Communications
Let's test that our allowed communications work as expected.

Test database connectivity from development namespace:
# Create a test pod in development namespace
kubectl run db-client -n development --image=mysql:8.0 --rm -it --restart=Never -- bash

# Inside the pod, test database connection
# mysql -h database-service.production.svc.cluster.local -u root -psecurepassword123 -e "SHOW DATABASES;"
# exit
Verify web application can still serve traffic within its namespace:
# Test internal connectivity within development namespace
kubectl run internal-client -n development --image=busybox:1.35 --rm -it --restart=Never -- wget -qO- --timeout=5 http://web-app-service.development.svc.cluster.local
Subtask 3.2: Verify Blocked Communications
Now let's confirm that unauthorized communications are properly blocked.

Test that testing namespace cannot access production database:
# This should fail
kubectl exec -it client-pod -n testing -- nc -zv $DB_SERVICE_IP 3306 || echo "Access correctly blocked"
Test that production pods cannot access external resources:
# Create a test pod in production namespace
kubectl run network-test -n production --image=busybox:1.35 --rm -it --restart=Never -- sh

# Inside the pod, try to access external resources (should fail)
# wget -qO- --timeout=5 http://google.com || echo "External access blocked"
# exit
Subtask 3.3: Create Advanced Network Policies
Let's implement more sophisticated policies for real-world scenarios.

Create a policy allowing specific ports and protocols:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress-specific-ports
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: testing
    ports:
    - protocol: TCP
      port: 80
  - from:
    - podSelector:
        matchLabels:
          role: monitoring
    ports:
    - protocol: TCP
      port: 8080
EOF
Create a time-based policy simulation using labels:
# Label testing namespace to allow access
kubectl label namespace testing name=testing

# Create a policy that allows access from testing namespace
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-testing-to-web
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: testing
    ports:
    - protocol: TCP
      port: 80
EOF
Test the new policy:
# This should now work
kubectl exec -it client-pod -n testing -- wget -qO- --timeout=5 http://$WEB_SERVICE_IP
Task 4: Simulating Network Attacks and Observing Mitigation
Subtask 4.1: Simulate Lateral Movement Attack
We'll simulate an attacker trying to move laterally through the network after compromising a pod.

Create a compromised pod scenario:
# Deploy a "compromised" pod in testing namespace
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: compromised-pod
  namespace: testing
  labels:
    app: compromised
    status: infected
spec:
  containers:
  - name: attacker
    image: nicolaka/netshoot:latest
    command: ['sleep', '3600']
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "NET_RAW"]
EOF
Attempt to scan the network from the compromised pod:
# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/compromised-pod -n testing --timeout=60s

# Attempt network scanning (this should be limited by our policies)
kubectl exec -it compromised-pod -n testing -- nmap -sT -p 1-1000 $DB_SERVICE_IP || echo "Scan blocked or limited"
Try to access unauthorized services:
# Attempt to access database directly
kubectl exec -it compromised-pod -n testing -- nc -zv $DB_SERVICE_IP 3306 || echo "Database access blocked"

# Attempt to access other namespaces
kubectl exec -it compromised-pod -n testing -- nslookup database-service.production.svc.cluster.local || echo "DNS resolution may be limited"
Subtask 4.2: Implement Attack Mitigation Policies
Now let's create policies specifically designed to mitigate attack scenarios.

Create a policy to isolate compromised pods:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-compromised-pods
  namespace: testing
spec:
  podSelector:
    matchLabels:
      status: infected
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
EOF
Test the isolation policy:
# This should now fail completely
kubectl exec -it compromised-pod -n testing -- wget -qO- --timeout=5 http://$WEB_SERVICE_IP || echo "Compromised pod isolated successfully"

# DNS should still work for basic functionality
kubectl exec -it compromised-pod -n testing -- nslookup kubernetes.default.svc.cluster.local
Subtask 4.3: Monitor and Log Network Policy Violations
Let's set up monitoring to detect policy violations.

Check for policy violations in logs:
# Check CNI plugin logs for policy denials (example for Calico)
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50 | grep -i "denied\|drop" || echo "No policy violations found in recent logs"

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=20
Create a monitoring pod to test policy enforcement:
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: policy-monitor
  namespace: testing
  labels:
    app: monitor
    role: security
spec:
  containers:
  - name: monitor
    image: busybox:1.35
    command: ['sh', '-c', 'while true; do echo "Monitoring network policies..."; sleep 30; done']
EOF
Subtask 4.4: Test Policy Effectiveness Against Common Attacks
Test against port scanning:
# Create a scanning script
kubectl exec -it compromised-pod -n testing -- sh -c '
for port in 22 80 443 3306 5432 6379; do
  echo "Scanning port $port on database..."
  nc -zv '$DB_SERVICE_IP' $port 2>&1 | head -1
done
' || echo "Port scanning mitigated by network policies"
Test against service enumeration:
# Attempt to enumerate services across namespaces
kubectl exec -it compromised-pod -n testing -- sh -c '
for ns in default kube-system development production; do
  echo "Attempting to access services in namespace: $ns"
  nc -zv kubernetes.default.svc.cluster.local 443 2>&1 | head -1
done
' || echo "Service enumeration limited by policies"
Task 5: Advanced Network Policy Configurations
Subtask 5.1: Implement Micro-segmentation
Create fine-grained network policies for micro-segmentation.

Create tier-based network policies:
# Policy for frontend tier
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-tier-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: testing
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
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
Create backend tier policy:
# Policy for backend tier
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-tier-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: development
      podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
Subtask 5.2: Implement CIDR-based Policies
Create policies based on IP ranges for external access control.

Create a policy allowing specific external IP ranges:
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-cidr
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
    ports:
    - protocol: TCP
      port: 80
EOF
Test CIDR-based restrictions:
# Check the current pod network CIDR
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# The policy will allow traffic from the pod network but block specific subnets
echo "CIDR-based policy applied successfully"
Troubleshooting Common Issues
Common Network Policy Problems and Solutions
Policy not taking effect:
# Check if CNI supports Network Policies
kubectl get pods -n kube-system | grep -E "(calico|cilium|weave)"

# Verify policy syntax
kubectl describe networkpolicy <policy-name> -n <namespace>
DNS resolution issues:
# Always allow DNS in egress policies
# Add this to your egress rules:
# - to: []
#   ports:
#   - protocol: UDP
#     port: 53
#   - protocol: TCP
#     port: 53
Testing connectivity issues:
# Use netshoot for advanced network debugging
kubectl run netshoot --rm -it --image nicolaka/netshoot -- bash
Lab Cleanup
Clean up the resources created during this lab:

# Delete all test pods
kubectl delete pod client-pod -n testing
kubectl delete pod compromised-pod -n testing
kubectl delete pod policy-monitor -n testing

# Delete deployments
kubectl delete deployment web-app -n development
kubectl delete deployment database -n production

# Delete services
kubectl delete service web-app-service -n development
kubectl delete service database-service -n production

# Delete network policies
kubectl delete networkpolicy --all -n development
kubectl delete networkpolicy --all -n production
kubectl delete networkpolicy --all -n testing

# Delete namespaces (this will delete everything in them)
kubectl delete namespace development
kubectl delete namespace production
kubectl delete namespace testing
Conclusion
In this comprehensive lab, you have successfully:

Mastered Network Policy Fundamentals: You learned how Kubernetes Network Policies work and their critical role in container security. Understanding these concepts is essential for implementing zero-trust networking in containerized environments.

Implemented Traffic Control: You configured various types of network policies including default deny-all policies, selective allow policies, and advanced micro-segmentation rules. These skills are directly applicable to real-world production environments where network security is paramount.

Tested Security Effectiveness: Through practical testing scenarios, you verified that your network policies work as intended, blocking unauthorized traffic while allowing legitimate communications. This hands-on validation approach is crucial for maintaining security confidence.

Simulated Attack Scenarios: You experienced how network policies can mitigate common attack vectors like lateral movement and service enumeration. Understanding these attack patterns and their mitigation strategies is valuable for security professionals.

Applied Advanced Configurations: You implemented sophisticated policies using CIDR blocks, tier-based segmentation, and namespace isolation. These advanced techniques are essential for complex enterprise environments.

Why This Matters: Container networking security is a critical component of modern cloud-native security strategies. Network policies provide the foundation for implementing zero-trust networking principles in Kubernetes environments. The skills you've developed in this lab are directly applicable to:

Production Security: Implementing defense-in-depth strategies for containerized applications
Compliance Requirements: Meeting regulatory requirements for network segmentation and access control
Incident Response: Quickly isolating compromised workloads to prevent lateral movement
Security Architecture: Designing secure multi-tenant Kubernetes environments
The hands-on experience gained from this lab prepares you for real-world scenarios where network security policies are essential for protecting containerized applications and data. These skills are highly valued in the industry and directly support your preparation for the Kubernetes and Cloud Native Security Associate (KCSA) certification.

Remember that network policies are just one layer of container security. In production environments, they should be combined with other security measures such as Pod Security Standards, RBAC, service mesh security, and runtime security monitoring for comprehensive protection.
