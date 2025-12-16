Lab 16: Threat Modeling in Kubernetes
Objectives
By the end of this lab, students will be able to:

• Understand the fundamentals of threat modeling in Kubernetes environments • Identify and map trust boundaries within a Kubernetes cluster • Recognize critical assets and potential attack vectors in containerized applications • Simulate common attack scenarios including privilege escalation • Implement security mitigations using Role-Based Access Control (RBAC) • Apply the STRIDE threat modeling methodology to Kubernetes workloads • Evaluate and strengthen the security posture of Kubernetes deployments

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with Linux command line operations • Basic knowledge of YAML configuration files • Understanding of container security concepts • Access to kubectl command-line tool

Note: Al Nafi provides ready-to-use Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Lab Environment Setup
Task 1: Verify Lab Environment
Subtask 1.1: Check Kubernetes Cluster Status
First, let's verify that our Kubernetes cluster is running and accessible.

# Check cluster information
kubectl cluster-info

# Verify node status
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
Subtask 1.2: Create Lab Namespace
Create a dedicated namespace for our threat modeling exercises.

# Create namespace for the lab
kubectl create namespace threat-modeling-lab

# Set the namespace as default for this session
kubectl config set-context --current --namespace=threat-modeling-lab

# Verify namespace creation
kubectl get namespaces
Task 2: Deploy Sample Application and Map Trust Boundaries
Subtask 2.1: Deploy Multi-Tier Application
We'll deploy a sample e-commerce application with multiple components to analyze trust boundaries.

# Create the application deployment file
cat << 'EOF' > ecommerce-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: threat-modeling-lab
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
          value: "http://backend-service:8080"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: threat-modeling-lab
spec:
  selector:
    app: frontend
    tier: web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: threat-modeling-lab
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
          value: "database-service"
        - name: DB_PORT
          value: "5432"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: threat-modeling-lab
spec:
  selector:
    app: backend
    tier: api
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: threat-modeling-lab
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
          value: "ecommerce"
        - name: POSTGRES_USER
          value: "dbuser"
        - name: POSTGRES_PASSWORD
          value: "dbpass123"
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
  namespace: threat-modeling-lab
spec:
  selector:
    app: database
    tier: data
  ports:
  - port: 5432
    targetPort: 5432
EOF

# Deploy the application
kubectl apply -f ecommerce-app.yaml

# Verify deployment
kubectl get deployments
kubectl get services
kubectl get pods
Subtask 2.2: Identify Trust Boundaries
Now let's map the trust boundaries in our application:

# Create a trust boundary analysis script
cat << 'EOF' > trust-boundary-analysis.sh
#!/bin/bash

echo "=== TRUST BOUNDARY ANALYSIS ==="
echo ""

echo "1. EXTERNAL TRUST BOUNDARY:"
echo "   - Internet -> LoadBalancer -> Frontend Service"
echo "   - Trust Level: UNTRUSTED"
echo ""

echo "2. FRONTEND TRUST BOUNDARY:"
echo "   - Frontend Pods -> Backend Service"
echo "   - Trust Level: SEMI-TRUSTED"
echo ""

echo "3. BACKEND TRUST BOUNDARY:"
echo "   - Backend Pods -> Database Service"
echo "   - Trust Level: TRUSTED"
echo ""

echo "4. INTERNAL CLUSTER BOUNDARIES:"
echo "   - Pod-to-Pod communication within namespace"
echo "   - Service-to-Service communication"
echo "   - Node-to-Node communication"
echo ""

echo "=== NETWORK POLICIES ANALYSIS ==="
kubectl get networkpolicies -n threat-modeling-lab
if [ $? -ne 0 ]; then
    echo "No network policies found - ALL TRAFFIC ALLOWED"
    echo "RISK: Unrestricted east-west traffic"
fi

echo ""
echo "=== SERVICE ACCOUNT ANALYSIS ==="
kubectl get serviceaccounts -n threat-modeling-lab
kubectl describe serviceaccount default -n threat-modeling-lab

EOF

chmod +x trust-boundary-analysis.sh
./trust-boundary-analysis.sh
Subtask 2.3: Identify Critical Assets
Create an asset inventory to understand what needs protection:

# Create asset inventory script
cat << 'EOF' > asset-inventory.sh
#!/bin/bash

echo "=== CRITICAL ASSETS INVENTORY ==="
echo ""

echo "1. DATA ASSETS:"
echo "   - Customer database (PostgreSQL)"
echo "   - Application secrets and credentials"
echo "   - Configuration data"
echo ""

echo "2. COMPUTE ASSETS:"
echo "   - Kubernetes nodes"
echo "   - Container runtime"
echo "   - Application pods"
echo ""

echo "3. NETWORK ASSETS:"
echo "   - Cluster network"
echo "   - Service mesh (if applicable)"
echo "   - Load balancer"
echo ""

echo "4. SECRETS ANALYSIS:"
kubectl get secrets -n threat-modeling-lab
echo ""

echo "5. CONFIGMAPS ANALYSIS:"
kubectl get configmaps -n threat-modeling-lab
echo ""

echo "6. PERSISTENT VOLUMES:"
kubectl get pv
kubectl get pvc -n threat-modeling-lab

EOF

chmod +x asset-inventory.sh
./asset-inventory.sh
Task 3: Simulate Attack Scenarios
Subtask 3.1: Create Vulnerable Pod for Testing
Deploy a pod with elevated privileges to simulate attack scenarios:

# Create a vulnerable pod configuration
cat << 'EOF' > vulnerable-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-pod
  namespace: threat-modeling-lab
  labels:
    app: vulnerable
spec:
  containers:
  - name: vulnerable-container
    image: ubuntu:20.04
    command: ["/bin/sleep", "3600"]
    securityContext:
      privileged: true
      runAsUser: 0
      capabilities:
        add:
        - SYS_ADMIN
        - NET_ADMIN
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
  serviceAccountName: default
EOF

# Deploy vulnerable pod
kubectl apply -f vulnerable-pod.yaml

# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/vulnerable-pod --timeout=60s
Subtask 3.2: Simulate Container Escape Attack
Demonstrate how a privileged container can access the host system:

# Execute commands in the vulnerable pod to show container escape
echo "=== CONTAINER ESCAPE SIMULATION ==="

# Access host filesystem
kubectl exec vulnerable-pod -- ls -la /host

# Show host processes
kubectl exec vulnerable-pod -- ps aux

# Show host network interfaces
kubectl exec vulnerable-pod -- ip addr show

# Demonstrate access to host Docker socket (if available)
kubectl exec vulnerable-pod -- ls -la /host/var/run/docker.sock 2>/dev/null || echo "Docker socket not accessible"

# Show mounted secrets
kubectl exec vulnerable-pod -- find /var/run/secrets -type f -exec echo "Found secret: {}" \; -exec cat {} \; 2>/dev/null
Subtask 3.3: Simulate Privilege Escalation
Create a scenario showing privilege escalation through service account tokens:

# Create a script to demonstrate privilege escalation
cat << 'EOF' > privilege-escalation-demo.sh
#!/bin/bash

echo "=== PRIVILEGE ESCALATION SIMULATION ==="
echo ""

# Show current service account
echo "1. Current Service Account:"
kubectl exec vulnerable-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
echo ""

# Show service account token
echo "2. Service Account Token (first 50 chars):"
kubectl exec vulnerable-pod -- head -c 50 /var/run/secrets/kubernetes.io/serviceaccount/token
echo "..."
echo ""

# Attempt to list pods using the service account token
echo "3. Attempting to list pods using service account token:"
kubectl exec vulnerable-pod -- sh -c '
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
curl -k -H "Authorization: Bearer $TOKEN" \
     https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods
' 2>/dev/null | head -20

echo ""
echo "4. Checking RBAC permissions:"
kubectl auth can-i --list --as=system:serviceaccount:threat-modeling-lab:default

EOF

chmod +x privilege-escalation-demo.sh
./privilege-escalation-demo.sh
Subtask 3.4: Simulate Network-Based Attacks
Demonstrate lateral movement within the cluster:

# Create network attack simulation
cat << 'EOF' > network-attack-simulation.sh
#!/bin/bash

echo "=== NETWORK ATTACK SIMULATION ==="
echo ""

# Show network connectivity from vulnerable pod
echo "1. Testing connectivity to other services:"

# Test connection to backend service
kubectl exec vulnerable-pod -- nc -zv backend-service 8080 2>&1 | grep -E "(open|succeeded)"

# Test connection to database service
kubectl exec vulnerable-pod -- nc -zv database-service 5432 2>&1 | grep -E "(open|succeeded)"

# Scan for other services in the namespace
echo ""
echo "2. Service discovery within namespace:"
kubectl exec vulnerable-pod -- nslookup backend-service
kubectl exec vulnerable-pod -- nslookup database-service

# Show DNS enumeration
echo ""
echo "3. DNS enumeration:"
kubectl exec vulnerable-pod -- cat /etc/resolv.conf

# Attempt to connect to Kubernetes API
echo ""
echo "4. Kubernetes API accessibility:"
kubectl exec vulnerable-pod -- nc -zv kubernetes.default.svc 443 2>&1 | grep -E "(open|succeeded)"

EOF

chmod +x network-attack-simulation.sh
./network-attack-simulation.sh
Task 4: Apply Security Mitigations
Subtask 4.1: Implement Role-Based Access Control (RBAC)
Create restrictive RBAC policies to limit service account permissions:

# Create RBAC configuration
cat << 'EOF' > rbac-security.yaml
# Create a restricted service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: restricted-sa
  namespace: threat-modeling-lab
---
# Create a role with minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: threat-modeling-lab
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
# Bind the role to the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: threat-modeling-lab
subjects:
- kind: ServiceAccount
  name: restricted-sa
  namespace: threat-modeling-lab
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
# Create a secure pod using the restricted service account
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: threat-modeling-lab
spec:
  serviceAccountName: restricted-sa
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: secure-container
    image: ubuntu:20.04
    command: ["/bin/sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "128Mi"
        cpu: "100m"
      requests:
        memory: "64Mi"
        cpu: "50m"
EOF

# Apply RBAC configuration
kubectl apply -f rbac-security.yaml

# Wait for secure pod to be ready
kubectl wait --for=condition=Ready pod/secure-pod --timeout=60s
Subtask 4.2: Implement Network Policies
Create network policies to restrict traffic between components:

# Create network policies
cat << 'EOF' > network-policies.yaml
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: threat-modeling-lab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow frontend to receive traffic from external sources
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: threat-modeling-lab
spec:
  podSelector:
    matchLabels:
      tier: web
  policyTypes:
  - Ingress
  ingress:
  - {}
---
# Allow backend to receive traffic only from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: threat-modeling-lab
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: web
    ports:
    - protocol: TCP
      port: 8080
---
# Allow database to receive traffic only from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-from-backend
  namespace: threat-modeling-lab
spec:
  podSelector:
    matchLabels:
      tier: data
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 5432
EOF

# Apply network policies
kubectl apply -f network-policies.yaml

# Verify network policies
kubectl get networkpolicies -n threat-modeling-lab
Subtask 4.3: Implement Pod Security Standards
Create pod security policies to enforce security standards:

# Create pod security policy
cat << 'EOF' > pod-security-policy.yaml
# Update deployments with security contexts
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-frontend
  namespace: threat-modeling-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-frontend
      tier: web
  template:
    metadata:
      labels:
        app: secure-frontend
        tier: web
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: frontend
        image: nginx:1.21
        ports:
        - containerPort: 80
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        resources:
          limits:
            memory: "256Mi"
            cpu: "200m"
          requests:
            memory: "128Mi"
            cpu: "100m"
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: var-cache-nginx
          mountPath: /var/cache/nginx
        - name: var-run
          mountPath: /var/run
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: var-cache-nginx
        emptyDir: {}
      - name: var-run
        emptyDir: {}
EOF

# Apply secure deployment
kubectl apply -f pod-security-policy.yaml

# Verify secure deployment
kubectl get deployment secure-frontend -n threat-modeling-lab
Subtask 4.4: Test Security Mitigations
Verify that our security controls are working:

# Create security validation script
cat << 'EOF' > security-validation.sh
#!/bin/bash

echo "=== SECURITY VALIDATION ==="
echo ""

echo "1. Testing RBAC restrictions:"
echo "Secure pod permissions:"
kubectl auth can-i --list --as=system:serviceaccount:threat-modeling-lab:restricted-sa

echo ""
echo "2. Testing network policy restrictions:"
# Try to connect from secure pod to database (should fail)
kubectl exec secure-pod -- nc -zv database-service 5432 2>&1 | grep -E "(refused|timeout|failed)" || echo "Connection unexpectedly succeeded"

echo ""
echo "3. Testing pod security context:"
kubectl exec secure-pod -- id
kubectl exec secure-pod -- ls -la /

echo ""
echo "4. Comparing with vulnerable pod:"
echo "Vulnerable pod user:"
kubectl exec vulnerable-pod -- id 2>/dev/null || echo "Vulnerable pod not accessible"

echo ""
echo "5. Network policy status:"
kubectl get networkpolicies -n threat-modeling-lab

EOF

chmod +x security-validation.sh
./security-validation.sh
Task 5: Threat Modeling Analysis Using STRIDE
Subtask 5.1: Apply STRIDE Methodology
Create a comprehensive threat analysis using the STRIDE framework:

# Create STRIDE analysis script
cat << 'EOF' > stride-analysis.sh
#!/bin/bash

echo "=== STRIDE THREAT MODELING ANALYSIS ==="
echo ""

echo "S - SPOOFING:"
echo "  Threats:"
echo "  - Service account token theft"
echo "  - Pod impersonation"
echo "  - DNS spoofing"
echo "  Mitigations:"
echo "  - Strong RBAC policies ✓"
echo "  - Service mesh with mTLS"
echo "  - Network policies ✓"
echo ""

echo "T - TAMPERING:"
echo "  Threats:"
echo "  - Container image tampering"
echo "  - Configuration modification"
echo "  - Data corruption"
echo "  Mitigations:"
echo "  - Image signing and verification"
echo "  - Read-only root filesystem ✓"
echo "  - Admission controllers"
echo ""

echo "R - REPUDIATION:"
echo "  Threats:"
echo "  - Lack of audit trails"
echo "  - Anonymous access"
echo "  Mitigations:"
echo "  - Audit logging"
echo "  - Service account tracking ✓"
echo "  - Activity monitoring"
echo ""

echo "I - INFORMATION DISCLOSURE:"
echo "  Threats:"
echo "  - Secret exposure"
echo "  - Log information leakage"
echo "  - Network traffic interception"
echo "  Mitigations:"
echo "  - Secrets management"
echo "  - Network encryption"
echo "  - Least privilege access ✓"
echo ""

echo "D - DENIAL OF SERVICE:"
echo "  Threats:"
echo "  - Resource exhaustion"
echo "  - Network flooding"
echo "  - Pod disruption"
echo "  Mitigations:"
echo "  - Resource limits ✓"
echo "  - Network policies ✓"
echo "  - Pod disruption budgets"
echo ""

echo "E - ELEVATION OF PRIVILEGE:"
echo "  Threats:"
echo "  - Container escape"
echo "  - Privilege escalation"
echo "  - Cluster admin access"
echo "  Mitigations:"
echo "  - Non-root containers ✓"
echo "  - Security contexts ✓"
echo "  - RBAC restrictions ✓"
echo ""

EOF

chmod +x stride-analysis.sh
./stride-analysis.sh
Subtask 5.2: Create Threat Model Documentation
Generate a comprehensive threat model report:

# Create threat model report
cat << 'EOF' > threat-model-report.md
# Kubernetes Threat Model Report

## Executive Summary
This threat model analyzes the security posture of a multi-tier e-commerce application deployed on Kubernetes.

## Architecture Overview
- **Frontend Tier**: NGINX web servers (2 replicas)
- **Backend Tier**: Apache HTTP servers (2 replicas)  
- **Database Tier**: PostgreSQL database (1 replica)

## Trust Boundaries Identified
1. **External Boundary**: Internet → Load Balancer → Frontend
2. **Application Boundary**: Frontend → Backend → Database
3. **Cluster Boundary**: Pod-to-Pod communication
4. **Node Boundary**: Container-to-Host access

## Critical Assets
- Customer database containing PII
- Application secrets and API keys
- Kubernetes cluster infrastructure
- Service account tokens

## Threat Analysis (STRIDE)

### High-Risk Threats
1. **Container Escape** (Elevation of Privilege)
   - Impact: Full host compromise
   - Likelihood: Medium
   - Mitigation: Security contexts, non-root containers

2. **Lateral Movement** (Information Disclosure)
   - Impact: Data breach
   - Likelihood: High
   - Mitigation: Network policies, microsegmentation

3. **Privilege Escalation** (Elevation of Privilege)
   - Impact: Cluster compromise
   - Likelihood: Medium
   - Mitigation: RBAC, least privilege

## Implemented Mitigations
- ✅ Role-Based Access Control (RBAC)
- ✅ Network Policies
- ✅ Pod Security Contexts
- ✅ Resource Limits
- ✅ Non-root Containers

## Recommendations
1. Implement admission controllers
2. Enable audit logging
3. Deploy service mesh for mTLS
4. Implement image scanning
5. Regular security assessments

## Risk Assessment
- **Overall Risk Level**: Medium
- **Residual Risk**: Low (after mitigations)
EOF

echo "Threat model report generated: threat-model-report.md"
cat threat-model-report.md
Task 6: Security Monitoring and Detection
Subtask 6.1: Implement Basic Security Monitoring
Set up monitoring to detect security events:

# Create security monitoring script
cat << 'EOF' > security-monitoring.sh
#!/bin/bash

echo "=== SECURITY MONITORING SETUP ==="
echo ""

echo "1. Checking for security events:"
# Check for failed authentication attempts
kubectl get events --field-selector reason=FailedMount -n threat-modeling-lab

echo ""
echo "2. Monitoring privileged containers:"
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.securityContext.privileged}{"\n"}{end}' -n threat-modeling-lab

echo ""
echo "3. Checking for containers running as root:"
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.securityContext.runAsUser}{"\n"}{end}' -n threat-modeling-lab

echo ""
echo "4. Network policy violations (simulated):"
# This would typically come from a CNI plugin or security tool
echo "No network policy violations detected"

echo ""
echo "5. Resource usage monitoring:"
kubectl top pods -n threat-modeling-lab 2>/dev/null || echo "Metrics server not available"

EOF

chmod +x security-monitoring.sh
./security-monitoring.sh
Subtask 6.2: Create Security Alerts
Implement basic alerting for security events:

# Create alerting configuration
cat << 'EOF' > security-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-alerts-config
  namespace: threat-modeling-lab
data:
  alert-rules.yaml: |
    groups:
    - name: kubernetes-security
      rules:
      - alert: PrivilegedContainerRunning
        expr: kube_pod_container_status_running{container="vulnerable-container"} == 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Privileged container detected"
          description: "A privileged container is running in the cluster"
      
      - alert: RootContainerRunning
        expr: kube_pod_container_info{container_id!="",image!~".*pause.*"} and on(pod,namespace) kube_pod_container_status_running == 1
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Container running as root"
          description: "Container {{ $labels.container }} in pod {{ $labels.pod }} is running as root"
EOF

kubectl apply -f security-alerts.yaml
echo "Security alerts configuration applied"
Cleanup and Summary
Cleanup Resources
# Clean up lab resources
echo "Cleaning up lab resources..."

# Delete vulnerable resources first
kubectl delete pod vulnerable-pod -n threat-modeling-lab --force --grace-period=0

# Delete other resources
kubectl delete -f ecommerce-app.yaml
kubectl delete -f rbac-security.yaml
kubectl delete -f network-policies.yaml
kubectl delete -f pod-security-policy.yaml
kubectl delete -f security-alerts.yaml

# Delete namespace
kubectl delete namespace threat-modeling-lab

echo "Cleanup completed"
Conclusion
In this comprehensive lab, you have successfully:

Key Accomplishments
Threat Modeling Fundamentals: You learned to systematically analyze security threats in Kubernetes environments using industry-standard methodologies like STRIDE.

Trust Boundary Mapping: You identified and mapped critical trust boundaries in a multi-tier application, understanding how data flows between different security zones.

Attack Simulation: You simulated real-world attack scenarios including container escape, privilege escalation, and lateral movement, gaining hands-on experience with common Kubernetes security vulnerabilities.

Security Mitigations: You implemented multiple layers of security controls including RBAC, network policies, pod security contexts, and resource limits to create a defense-in-depth strategy.

Practical Security Skills: You developed practical skills in securing Kubernetes workloads that directly apply to real-world cloud-native environments.

Why This Matters
Kubernetes security is critical in today's cloud-native landscape. The skills you've developed in this lab are essential for:

Cloud Security Engineers who need to secure containerized applications
DevSecOps Professionals implementing security in CI/CD pipelines
Security Architects designing secure cloud-native systems
Kubernetes Administrators managing production clusters
The threat modeling approach you've learned provides a systematic way to identify, analyze, and mitigate security risks before they become real-world incidents. By understanding both attack techniques and defensive measures, you're better prepared to build and maintain secure Kubernetes environments.

Next Steps
To further develop your Kubernetes security expertise:

Explore advanced topics like service mesh security and admission controllers
Practice with additional attack scenarios and security tools
Study real-world Kubernetes security incidents and their mitigations
Consider pursuing the Kubernetes and Cloud Native Security Associate (KCSA) certification
The foundation you've built in this lab will serve you well as you continue your journey in cloud-native security.
