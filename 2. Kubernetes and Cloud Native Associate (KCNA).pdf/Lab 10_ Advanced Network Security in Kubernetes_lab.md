Lab 10: Advanced Network Security in Kubernetes
Objectives
By the end of this lab, students will be able to:

• Understand and implement Kubernetes Network Policies to control pod-to-pod communication • Deploy and configure Istio service mesh for advanced traffic management and security • Create security policies that restrict network access based on labels and namespaces • Monitor network traffic and identify security threats using service mesh observability tools • Simulate network-based attacks and demonstrate how security policies mitigate them • Apply defense-in-depth strategies for Kubernetes network security

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments, namespaces) • Familiarity with YAML configuration files • Basic knowledge of networking concepts (TCP/IP, ports, protocols) • Experience with command-line interface (CLI) operations • Understanding of container security fundamentals

Required Knowledge Level: Intermediate familiarity with Kubernetes administration

Lab Environment Setup
Al Nafi Cloud Environment: This lab uses Al Nafi's pre-configured Linux-based cloud machines. Simply click Start Lab to access your environment - no VM setup required! Your environment includes:

• Ubuntu 22.04 LTS with Docker and Kubernetes pre-installed • kubectl configured and ready to use • Internet access for downloading required tools • All necessary permissions for cluster administration

Task 1: Setting Up the Lab Environment and Network Policies
Subtask 1.1: Verify Kubernetes Cluster and Create Namespaces
First, let's verify our cluster is running and create the necessary namespaces for our security demonstration.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Create namespaces for our lab
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database
kubectl create namespace monitoring

# Verify namespaces were created
kubectl get namespaces
Subtask 1.2: Deploy Sample Applications
We'll deploy a multi-tier application to demonstrate network security policies.

Create the frontend application:

# Create frontend deployment
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: frontend
  labels:
    app: frontend
    tier: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: BACKEND_URL
          value: "http://backend-service.backend.svc.cluster.local:8080"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: frontend
spec:
  selector:
    app: frontend
    tier: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Create the backend application:

# Create backend deployment
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: backend
  labels:
    app: backend
    tier: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
      - name: backend
        image: httpd:2.4
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "database-service.database.svc.cluster.local"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: backend
spec:
  selector:
    app: backend
    tier: api
  ports:
  - port: 8080
    targetPort: 80
  type: ClusterIP
EOF
Create the database application:

# Create database deployment
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-app
  namespace: database
  labels:
    app: database
    tier: data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: data
  template:
    metadata:
      labels:
        app: database
        tier: data
    spec:
      containers:
      - name: database
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "appdb"
        - name: POSTGRES_USER
          value: "dbuser"
        - name: POSTGRES_PASSWORD
          value: "dbpass123"
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: database
spec:
  selector:
    app: database
    tier: data
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
EOF
Subtask 1.3: Verify Application Deployment
# Check all deployments
kubectl get deployments --all-namespaces

# Check all services
kubectl get services --all-namespaces

# Verify pods are running
kubectl get pods --all-namespaces -o wide
Subtask 1.4: Test Initial Connectivity (Before Network Policies)
Let's test that our applications can communicate freely before implementing security policies:

# Get a frontend pod name
FRONTEND_POD=$(kubectl get pods -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# Test connectivity from frontend to backend
kubectl exec -n frontend $FRONTEND_POD -- curl -s http://backend-service.backend.svc.cluster.local:8080

# Test connectivity from frontend to database (this should work but shouldn't in production)
kubectl exec -n frontend $FRONTEND_POD -- nc -zv database-service.database.svc.cluster.local 5432

# Get a backend pod name
BACKEND_POD=$(kubectl get pods -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}')

# Test connectivity from backend to database
kubectl exec -n backend $BACKEND_POD -- nc -zv database-service.database.svc.cluster.local 5432
Task 2: Implementing Network Policies for Pod-Level Communication Control
Subtask 2.1: Create Default Deny Network Policy
First, we'll implement a default deny policy to block all traffic, then selectively allow what's needed:

# Create default deny policy for frontend namespace
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Create default deny policy for backend namespace
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Create default deny policy for database namespace
cat << EOF | kubectl apply -f -
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
Subtask 2.2: Test Connectivity After Default Deny
# Test connectivity - these should now fail
kubectl exec -n frontend $FRONTEND_POD -- timeout 5 curl -s http://backend-service.backend.svc.cluster.local:8080 || echo "Connection blocked as expected"

kubectl exec -n backend $BACKEND_POD -- timeout 5 nc -zv database-service.database.svc.cluster.local 5432 || echo "Connection blocked as expected"
Subtask 2.3: Create Selective Allow Policies
Now we'll create policies that allow only the necessary communication:

# Allow frontend to communicate with backend
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
      tier: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    - podSelector:
        matchLabels:
          app: frontend
          tier: web
    ports:
    - protocol: TCP
      port: 8080
EOF

# Allow backend to communicate with database
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
      tier: data
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend
    - podSelector:
        matchLabels:
          app: backend
          tier: api
    ports:
    - protocol: TCP
      port: 5432
EOF

# Allow egress from frontend to backend
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-egress
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
      tier: web
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend
    ports:
    - protocol: TCP
      port: 8080
  - to: []
    ports:
    - protocol: UDP
      port: 53
EOF

# Allow egress from backend to database
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
      tier: api
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432
  - to: []
    ports:
    - protocol: UDP
      port: 53
EOF
Subtask 2.4: Label Namespaces for Network Policies
# Add labels to namespaces for network policy selectors
kubectl label namespace frontend name=frontend
kubectl label namespace backend name=backend
kubectl label namespace database name=database
Subtask 2.5: Test Selective Connectivity
# Test allowed connections
echo "Testing frontend to backend connection:"
kubectl exec -n frontend $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" http://backend-service.backend.svc.cluster.local:8080

echo "Testing backend to database connection:"
kubectl exec -n backend $BACKEND_POD -- nc -zv database-service.database.svc.cluster.local 5432

# Test blocked connections (frontend should not access database directly)
echo "Testing frontend to database connection (should be blocked):"
kubectl exec -n frontend $FRONTEND_POD -- timeout 5 nc -zv database-service.database.svc.cluster.local 5432 || echo "Connection properly blocked"
Task 3: Deploying and Configuring Istio Service Mesh
Subtask 3.1: Install Istio
# Download and install Istio
curl -L https://istio.io/downloadIstio | sh -

# Add Istio to PATH
export PATH=$PWD/istio-*/bin:$PATH

# Install Istio with demo profile
istioctl install --set values.defaultRevision=default -y

# Verify installation
kubectl get pods -n istio-system

# Enable automatic sidecar injection for our namespaces
kubectl label namespace frontend istio-injection=enabled
kubectl label namespace backend istio-injection=enabled
kubectl label namespace database istio-injection=enabled
Subtask 3.2: Restart Deployments to Inject Sidecars
# Restart deployments to inject Istio sidecars
kubectl rollout restart deployment/frontend-app -n frontend
kubectl rollout restart deployment/backend-app -n backend
kubectl rollout restart deployment/database-app -n database

# Wait for rollouts to complete
kubectl rollout status deployment/frontend-app -n frontend
kubectl rollout status deployment/backend-app -n backend
kubectl rollout status deployment/database-app -n database

# Verify sidecars are injected (should show 2/2 containers per pod)
kubectl get pods -n frontend
kubectl get pods -n backend
kubectl get pods -n database
Subtask 3.3: Install Istio Observability Tools
# Install Kiali, Prometheus, Grafana, and Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# Wait for deployments to be ready
kubectl wait --for=condition=available --timeout=300s deployment/kiali -n istio-system
kubectl wait --for=condition=available --timeout=300s deployment/prometheus -n istio-system
Subtask 3.4: Configure Istio Security Policies
Create Istio authorization policies for fine-grained access control:

# Create authorization policy for backend service
cat << EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-policy
  namespace: backend
spec:
  selector:
    matchLabels:
      app: backend
  rules:
  - from:
    - source:
        namespaces: ["frontend"]
        principals: ["cluster.local/ns/frontend/sa/default"]
    to:
    - operation:
        methods: ["GET", "POST"]
        ports: ["8080"]
EOF

# Create authorization policy for database service
cat << EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: database-policy
  namespace: database
spec:
  selector:
    matchLabels:
      app: database
  rules:
  - from:
    - source:
        namespaces: ["backend"]
        principals: ["cluster.local/ns/backend/sa/default"]
    to:
    - operation:
        ports: ["5432"]
EOF
Subtask 3.5: Enable mTLS (Mutual TLS)
# Enable strict mTLS for all services
cat << EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

# Verify mTLS is working
kubectl get peerauthentication --all-namespaces
Task 4: Simulating Network Attacks and Observing Mitigation
Subtask 4.1: Deploy a Malicious Pod
Create a pod that will attempt unauthorized access:

# Create a malicious namespace
kubectl create namespace attacker

# Deploy attacker pod (without Istio injection)
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: attacker-pod
  namespace: attacker
  labels:
    app: attacker
spec:
  containers:
  - name: attacker
    image: nicolaka/netshoot
    command: ["/bin/bash"]
    args: ["-c", "while true; do sleep 30; done"]
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod/attacker-pod -n attacker
Subtask 4.2: Simulate Attack Scenarios
# Scenario 1: Direct database access attempt
echo "=== Attempting direct database access from attacker ==="
kubectl exec -n attacker attacker-pod -- timeout 10 nc -zv database-service.database.svc.cluster.local 5432 || echo "Attack blocked by network policy"

# Scenario 2: Backend service access attempt
echo "=== Attempting backend service access from attacker ==="
kubectl exec -n attacker attacker-pod -- timeout 10 curl -s http://backend-service.backend.svc.cluster.local:8080 || echo "Attack blocked by network policy"

# Scenario 3: Port scanning attempt
echo "=== Attempting port scan ==="
kubectl exec -n attacker attacker-pod -- timeout 10 nmap -p 1-1000 backend-service.backend.svc.cluster.local || echo "Port scan blocked"
Subtask 4.3: Deploy Legitimate Test Pod with Istio
# Deploy a legitimate test pod in frontend namespace
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: frontend
  labels:
    app: test-client
spec:
  containers:
  - name: test-client
    image: nicolaka/netshoot
    command: ["/bin/bash"]
    args: ["-c", "while true; do sleep 30; done"]
EOF

# Wait for pod to be ready and sidecar injected
kubectl wait --for=condition=ready pod/test-pod -n frontend

# Test legitimate access
echo "=== Testing legitimate access patterns ==="
kubectl exec -n frontend test-pod -c test-client -- curl -s -o /dev/null -w "%{http_code}" http://backend-service.backend.svc.cluster.local:8080
Subtask 4.4: Monitor Traffic with Kiali
# Generate some traffic for monitoring
for i in {1..20}; do
  kubectl exec -n frontend test-pod -c test-client -- curl -s http://backend-service.backend.svc.cluster.local:8080 > /dev/null
  sleep 2
done

# Port forward to access Kiali dashboard
echo "Starting Kiali dashboard..."
kubectl port-forward -n istio-system svc/kiali 20001:20001 &
KIALI_PID=$!

echo "Kiali dashboard available at: http://localhost:20001"
echo "Username: admin, Password: admin"
echo "Press Enter to continue after viewing the dashboard..."
read

# Stop port forwarding
kill $KIALI_PID
Subtask 4.5: Analyze Security Metrics
# Check Istio proxy statistics
kubectl exec -n frontend $FRONTEND_POD -c istio-proxy -- pilot-agent request GET stats/prometheus | grep istio_requests_total

# Check authorization policy effectiveness
kubectl exec -n backend $BACKEND_POD -c istio-proxy -- pilot-agent request GET stats/prometheus | grep istio_request_total | grep response_code

# View network policy events
kubectl get events --all-namespaces --field-selector reason=NetworkPolicyViolation
Task 5: Advanced Security Monitoring and Alerting
Subtask 5.1: Create Security Monitoring Dashboard
# Port forward to Grafana
kubectl port-forward -n istio-system svc/grafana 3000:3000 &
GRAFANA_PID=$!

echo "Grafana dashboard available at: http://localhost:3000"
echo "Username: admin, Password: admin"
echo "Navigate to Istio Service Dashboard to view security metrics"
echo "Press Enter to continue after viewing the dashboard..."
read

# Stop port forwarding
kill $GRAFANA_PID
Subtask 5.2: Set Up Traffic Policies for Rate Limiting
# Create rate limiting policy
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit-backend
  namespace: backend
spec:
  workloadSelector:
    labels:
      app: backend
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.local_ratelimit
        typed_config:
          "@type": type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
          value:
            stat_prefix: rate_limiter
            token_bucket:
              max_tokens: 10
              tokens_per_fill: 10
              fill_interval: 60s
            filter_enabled:
              runtime_key: local_rate_limit_enabled
              default_value:
                numerator: 100
                denominator: HUNDRED
            filter_enforced:
              runtime_key: local_rate_limit_enforced
              default_value:
                numerator: 100
                denominator: HUNDRED
EOF
Subtask 5.3: Test Rate Limiting
# Test rate limiting by sending rapid requests
echo "Testing rate limiting..."
for i in {1..15}; do
  RESPONSE=$(kubectl exec -n frontend test-pod -c test-client -- curl -s -o /dev/null -w "%{http_code}" http://backend-service.backend.svc.cluster.local:8080)
  echo "Request $i: HTTP $RESPONSE"
  sleep 1
done
Task 6: Security Audit and Compliance Verification
Subtask 6.1: Verify Network Policy Compliance
# List all network policies
echo "=== Current Network Policies ==="
kubectl get networkpolicies --all-namespaces

# Verify policy coverage
echo "=== Verifying Policy Coverage ==="
kubectl describe networkpolicy default-deny-all -n frontend
kubectl describe networkpolicy allow-frontend-to-backend -n backend
kubectl describe networkpolicy allow-backend-to-database -n database
Subtask 6.2: Audit Istio Security Configuration
# Check Istio security configuration
echo "=== Istio Security Configuration ==="
kubectl get peerauthentication --all-namespaces
kubectl get authorizationpolicy --all-namespaces

# Verify mTLS status
istioctl authn tls-check frontend-service.frontend.svc.cluster.local
istioctl authn tls-check backend-service.backend.svc.cluster.local
istioctl authn tls-check database-service.database.svc.cluster.local
Subtask 6.3: Generate Security Report
# Create a comprehensive security report
cat << EOF > security-audit-report.txt
=== Kubernetes Network Security Audit Report ===
Date: $(date)

1. Network Policies Status:
$(kubectl get networkpolicies --all-namespaces)

2. Istio Security Policies:
$(kubectl get authorizationpolicy --all-namespaces)

3. mTLS Configuration:
$(kubectl get peerauthentication --all-namespaces)

4. Service Mesh Status:
$(kubectl get pods -n istio-system)

5. Application Security Status:
- Frontend: Protected by network policies and Istio authorization
- Backend: Rate limited and access controlled
- Database: Restricted to backend access only

6. Security Violations Detected:
$(kubectl get events --all-namespaces --field-selector reason=NetworkPolicyViolation)

=== End of Report ===
EOF

echo "Security audit report generated: security-audit-report.txt"
cat security-audit-report.txt
Troubleshooting Common Issues
Network Policy Issues
# If network policies aren't working, check:
# 1. CNI plugin supports network policies
kubectl get nodes -o wide

# 2. Verify policy syntax
kubectl describe networkpolicy <policy-name> -n <namespace>

# 3. Check pod labels match selectors
kubectl get pods --show-labels -n <namespace>
Istio Issues
# If Istio sidecars aren't injecting:
# 1. Verify namespace labeling
kubectl get namespace -L istio-injection

# 2. Check Istio installation
istioctl verify-install

# 3. Restart deployments after labeling namespace
kubectl rollout restart deployment/<deployment-name> -n <namespace>
Connectivity Issues
# Debug connectivity problems:
# 1. Check service endpoints
kubectl get endpoints -n <namespace>

# 2. Verify DNS resolution
kubectl exec -n <namespace> <pod-name> -- nslookup <service-name>

# 3. Test with netcat
kubectl exec -n <namespace> <pod-name> -- nc -zv <service-name> <port>
Cleanup
# Remove all lab resources
kubectl delete namespace frontend backend database attacker monitoring
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
kubectl delete -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# Uninstall Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system
Conclusion
In this comprehensive lab, you have successfully:

• Implemented Defense-in-Depth Security: You configured multiple layers of network security using both Kubernetes Network Policies and Istio service mesh, demonstrating how different security mechanisms work together to protect applications.

• Mastered Network Policy Configuration: You learned to create default-deny policies and then selectively allow traffic based on labels, namespaces, and ports, implementing the principle of least privilege for network access.

• Deployed and Configured Service Mesh Security: You successfully installed Istio, enabled automatic sidecar injection, and configured advanced security features including mutual TLS (mTLS) and authorization policies.

• Demonstrated Attack Mitigation: You simulated real-world attack scenarios including unauthorized database access, service enumeration, and port scanning, then observed how your security policies effectively blocked these attempts.

• Implemented Advanced Security Features: You configured rate limiting, traffic monitoring, and security observability tools, providing comprehensive visibility into your application's security posture.

• Performed Security Auditing: You learned to verify security configurations, generate compliance reports, and troubleshoot common security issues in Kubernetes environments.

Why This Matters: Network security in Kubernetes is critical because containers and microservices create a larger attack surface with more network communication paths. The skills you've developed in this lab are essential for:

Production Security: Implementing these patterns in real-world applications prevents data breaches and unauthorized access
Compliance Requirements: Many regulatory frameworks require network segmentation and access controls
Incident Response: Understanding how to monitor and analyze network traffic helps in detecting and responding to security incidents
Career Development: These skills are highly valued in DevSecOps, Cloud Security, and Kubernetes administration roles
The combination of Kubernetes Network Policies and Istio service mesh provides enterprise-grade security capabilities that are essential for modern cloud-native applications. You now have hands-on experience with the tools and techniques used by security professionals to protect Kubernetes environments in production.
