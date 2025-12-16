Lab 5: Threat Modeling for Kubernetes
Objectives
By the end of this lab, students will be able to:

• Understand the fundamentals of threat modeling in Kubernetes environments • Map trust boundaries and data flows in a sample Kubernetes application • Identify potential attack vectors including privilege escalation and lateral movement • Apply security mitigations using Pod Security Admission and network isolation • Implement defense-in-depth strategies for Kubernetes workloads • Analyze and document security risks in cloud-native applications

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with YAML configuration files • Basic knowledge of Linux command line operations • Understanding of networking concepts (IP addresses, ports, protocols) • Awareness of common cybersecurity principles

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 22.04 LTS with kubectl pre-configured • Minikube for local Kubernetes cluster • Docker runtime environment • Text editors (nano, vim) • Network analysis tools

Task 1: Setting Up the Lab Environment and Sample Application
Subtask 1.1: Initialize Kubernetes Cluster
First, let's start our Minikube cluster and verify it's running properly.

# Start Minikube cluster
minikube start --driver=docker

# Verify cluster status
kubectl cluster-info

# Check node status
kubectl get nodes
Subtask 1.2: Deploy Sample Multi-Tier Application
We'll deploy a sample e-commerce application with multiple components to analyze.

Create the application namespace:

# Create dedicated namespace
kubectl create namespace ecommerce-app

# Set default namespace context
kubectl config set-context --current --namespace=ecommerce-app
Create the database deployment:

cat << 'EOF' > database.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-db
  namespace: ecommerce-app
  labels:
    app: mysql-db
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-db
  template:
    metadata:
      labels:
        app: mysql-db
        tier: database
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword123"
        - name: MYSQL_DATABASE
          value: "ecommerce"
        - name: MYSQL_USER
          value: "appuser"
        - name: MYSQL_PASSWORD
          value: "apppassword123"
        ports:
        - containerPort: 3306
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: ecommerce-app
spec:
  selector:
    app: mysql-db
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
EOF

kubectl apply -f database.yaml
Create the backend API deployment:

cat << 'EOF' > backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  namespace: ecommerce-app
  labels:
    app: api-backend
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
        tier: backend
    spec:
      containers:
      - name: api
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: DB_HOST
          value: "mysql-service"
        - name: DB_USER
          value: "appuser"
        - name: DB_PASSWORD
          value: "apppassword123"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: ecommerce-app
spec:
  selector:
    app: api-backend
  ports:
  - port: 8080
    targetPort: 80
  type: ClusterIP
EOF

kubectl apply -f backend.yaml
Create the frontend deployment:

cat << 'EOF' > frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  namespace: ecommerce-app
  labels:
    app: web-frontend
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
        tier: frontend
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        - name: API_ENDPOINT
          value: "http://api-service:8080"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: ecommerce-app
spec:
  selector:
    app: web-frontend
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f frontend.yaml
Subtask 1.3: Verify Application Deployment
Check that all components are running:

# Check all pods in the namespace
kubectl get pods -o wide

# Check services
kubectl get services

# Check deployments
kubectl get deployments

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod --all --timeout=300s
Task 2: Mapping Trust Boundaries and Data Flows
Subtask 2.1: Identify System Components and Trust Boundaries
Let's analyze our application architecture and identify trust boundaries.

Create a documentation file for our threat model:

cat << 'EOF' > threat-model-analysis.md
# E-commerce Application Threat Model

## System Architecture Overview

### Components:
1. **Web Frontend (web-frontend)**
   - Technology: Nginx
   - Replicas: 3
   - Exposure: LoadBalancer service
   - Trust Level: Public-facing (Untrusted)

2. **API Backend (api-backend)**
   - Technology: Nginx (simulating API server)
   - Replicas: 2
   - Exposure: ClusterIP service
   - Trust Level: Internal (Semi-trusted)

3. **Database (mysql-db)**
   - Technology: MySQL 8.0
   - Replicas: 1
   - Exposure: ClusterIP service
   - Trust Level: Internal (Trusted)

### Trust Boundaries Identified:
1. **Internet → Frontend**: External users to web application
2. **Frontend → Backend**: Web tier to application tier
3. **Backend → Database**: Application tier to data tier
4. **Pod → Node**: Container to host system
5. **Namespace → Cluster**: Application boundary within cluster

EOF

cat threat-model-analysis.md
Subtask 2.2: Map Data Flows
Let's trace the data flows through our application:

cat << 'EOF' >> threat-model-analysis.md

## Data Flow Analysis

### Primary Data Flows:
1. **User Request Flow**:
   Internet → LoadBalancer → web-frontend → api-service → api-backend → mysql-service → mysql-db

2. **Response Flow**:
   mysql-db → mysql-service → api-backend → api-service → web-frontend → LoadBalancer → Internet

3. **Internal Communication**:
   - Frontend pods communicate with API service (HTTP/8080)
   - Backend pods communicate with MySQL service (TCP/3306)
   - All communication within cluster network

### Data Types:
- User credentials and session data
- Product catalog information
- Order and payment data
- Application logs and metrics

EOF
Subtask 2.3: Analyze Network Connectivity
Let's examine the actual network connectivity in our cluster:

# Get detailed network information
kubectl get pods -o wide

# Check service endpoints
kubectl get endpoints

# Examine network policies (should be empty initially)
kubectl get networkpolicies

# Test connectivity between components
kubectl exec -it $(kubectl get pod -l app=web-frontend -o jsonpath='{.items[0].metadata.name}') -- nslookup api-service

kubectl exec -it $(kubectl get pod -l app=api-backend -o jsonpath='{.items[0].metadata.name}') -- nslookup mysql-service
Task 3: Identifying Potential Attack Vectors
Subtask 3.1: Analyze Privilege Escalation Risks
Let's examine the security context of our pods to identify privilege escalation risks:

# Check current security contexts
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.securityContext}{"\t"}{.spec.containers[*].securityContext}{"\n"}{end}'

# Get detailed security information for each deployment
kubectl describe deployment web-frontend | grep -A 10 -B 5 -i security
kubectl describe deployment api-backend | grep -A 10 -B 5 -i security
kubectl describe deployment mysql-db | grep -A 10 -B 5 -i security
Document the findings:

cat << 'EOF' >> threat-model-analysis.md

## Attack Vector Analysis

### 1. Privilege Escalation Risks

#### Current Security Posture:
- **No security contexts defined**: Containers run with default privileges
- **Root access**: Containers may run as root user
- **Privileged capabilities**: Default capabilities may be excessive
- **Host access**: No restrictions on host filesystem access

#### Potential Attack Vectors:
1. **Container Breakout**: Attacker gains access to host system
2. **Privilege Escalation**: Escalate from container user to root
3. **Capability Abuse**: Misuse of Linux capabilities
4. **Host Path Mounting**: Access to sensitive host directories

EOF
Subtask 3.2: Identify Lateral Movement Opportunities
Analyze network segmentation and lateral movement risks:

# Test network connectivity between different tiers
echo "Testing lateral movement possibilities..."

# From frontend to backend (expected)
kubectl exec -it $(kubectl get pod -l app=web-frontend -o jsonpath='{.items[0].metadata.name}') -- wget -qO- --timeout=5 http://api-service:8080 && echo "Frontend → Backend: ACCESSIBLE"

# From frontend to database (should be restricted)
kubectl exec -it $(kubectl get pod -l app=web-frontend -o jsonpath='{.items[0].metadata.name}') -- nc -zv mysql-service 3306 && echo "Frontend → Database: ACCESSIBLE (RISK!)"

# From backend to database (expected)
kubectl exec -it $(kubectl get pod -l app=api-backend -o jsonpath='{.items[0].metadata.name}') -- nc -zv mysql-service 3306 && echo "Backend → Database: ACCESSIBLE"
Document lateral movement risks:

cat << 'EOF' >> threat-model-analysis.md

### 2. Lateral Movement Risks

#### Network Segmentation Analysis:
- **No network policies**: All pods can communicate with all services
- **Flat network**: No micro-segmentation between tiers
- **Service discovery**: All services discoverable via DNS
- **Port accessibility**: All service ports accessible cluster-wide

#### Potential Attack Vectors:
1. **Cross-tier access**: Frontend can directly access database
2. **Service enumeration**: Attackers can discover all services
3. **Protocol abuse**: Unrestricted protocol usage
4. **Data exfiltration**: Direct database access from compromised frontend

EOF
Subtask 3.3: Assess Container and Image Security
Examine container images and configurations for vulnerabilities:

# Check image information
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# Examine resource limits and requests
kubectl describe pods | grep -A 5 -B 5 -i "limits\|requests"

# Check for secrets and environment variables
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].env[*]}{"\n"}{end}'
Document container security issues:

cat << 'EOF' >> threat-model-analysis.md

### 3. Container Security Risks

#### Image Security Issues:
- **Base image vulnerabilities**: Using standard images without security scanning
- **Outdated packages**: Potential unpatched vulnerabilities
- **Unnecessary packages**: Increased attack surface

#### Configuration Issues:
- **Hardcoded credentials**: Database passwords in environment variables
- **No resource limits**: Potential for resource exhaustion attacks
- **Default configurations**: Using default settings without hardening

#### Runtime Security Issues:
- **No admission controls**: No validation of pod security standards
- **Unrestricted capabilities**: Containers have default Linux capabilities
- **No AppArmor/SELinux**: Missing mandatory access controls

EOF
Task 4: Applying Security Mitigations
Subtask 4.1: Implement Pod Security Admission
First, let's enable Pod Security Standards for our namespace:

# Label namespace with Pod Security Standards
kubectl label namespace ecommerce-app pod-security.kubernetes.io/enforce=restricted
kubectl label namespace ecommerce-app pod-security.kubernetes.io/audit=restricted
kubectl label namespace ecommerce-app pod-security.kubernetes.io/warn=restricted

# Verify labels
kubectl get namespace ecommerce-app --show-labels
Now let's create secure versions of our deployments:

cat << 'EOF' > secure-database.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-mysql-db
  namespace: ecommerce-app
  labels:
    app: secure-mysql-db
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secure-mysql-db
  template:
    metadata:
      labels:
        app: secure-mysql-db
        tier: database
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: mysql
        image: mysql:8.0
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
          runAsNonRoot: true
          runAsUser: 999
          runAsGroup: 999
          capabilities:
            drop:
            - ALL
            add:
            - CHOWN
            - DAC_OVERRIDE
            - SETGID
            - SETUID
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_DATABASE
          value: "ecommerce"
        - name: MYSQL_USER
          value: "appuser"
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: user-password
        ports:
        - containerPort: 3306
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: secure-mysql-service
  namespace: ecommerce-app
spec:
  selector:
    app: secure-mysql-db
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
EOF
Create secrets for database credentials:

# Create secret for database credentials
kubectl create secret generic mysql-secret \
  --from-literal=root-password=secureRootPass123! \
  --from-literal=user-password=secureUserPass123! \
  --namespace=ecommerce-app

# Verify secret creation
kubectl get secrets
Create secure backend deployment:

cat << 'EOF' > secure-backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-api-backend
  namespace: ecommerce-app
  labels:
    app: secure-api-backend
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-api-backend
  template:
    metadata:
      labels:
        app: secure-api-backend
        tier: backend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        fsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: api
        image: nginx:alpine
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 101
          runAsGroup: 101
          capabilities:
            drop:
            - ALL
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "secure-mysql-service"
        - name: DB_USER
          value: "appuser"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: user-password
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
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
---
apiVersion: v1
kind: Service
metadata:
  name: secure-api-service
  namespace: ecommerce-app
spec:
  selector:
    app: secure-api-backend
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
EOF
Create secure frontend deployment:

cat << 'EOF' > secure-frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-web-frontend
  namespace: ecommerce-app
  labels:
    app: secure-web-frontend
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-web-frontend
  template:
    metadata:
      labels:
        app: secure-web-frontend
        tier: frontend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        fsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: web
        image: nginx:alpine
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 101
          runAsGroup: 101
          capabilities:
            drop:
            - ALL
        ports:
        - containerPort: 8080
        env:
        - name: API_ENDPOINT
          value: "http://secure-api-service:8080"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
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
---
apiVersion: v1
kind: Service
metadata:
  name: secure-web-service
  namespace: ecommerce-app
spec:
  selector:
    app: secure-web-frontend
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
EOF
Deploy the secure applications:

# Deploy secure versions
kubectl apply -f secure-database.yaml
kubectl apply -f secure-backend.yaml
kubectl apply -f secure-frontend.yaml

# Wait for deployments to be ready
kubectl wait --for=condition=available deployment/secure-mysql-db --timeout=300s
kubectl wait --for=condition=available deployment/secure-api-backend --timeout=300s
kubectl wait --for=condition=available deployment/secure-web-frontend --timeout=300s

# Verify secure deployments
kubectl get pods -l tier=database
kubectl get pods -l tier=backend  
kubectl get pods -l tier=frontend
Subtask 4.2: Implement Network Isolation
Now let's implement network policies to restrict communication between tiers:

cat << 'EOF' > network-policies.yaml
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: ecommerce-app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow frontend to communicate with backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: ecommerce-app
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# Allow backend to communicate with database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: ecommerce-app
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 3306
---
# Allow external traffic to frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-to-frontend
  namespace: ecommerce-app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 8080
EOF

kubectl apply -f network-policies.yaml
Verify network policies are applied:

# Check network policies
kubectl get networkpolicies

# Describe network policies
kubectl describe networkpolicy default-deny-ingress
kubectl describe networkpolicy allow-frontend-to-backend
kubectl describe networkpolicy allow-backend-to-database
kubectl describe networkpolicy allow-external-to-frontend
Subtask 4.3: Test Network Isolation
Let's test that our network policies are working correctly:

echo "Testing network isolation..."

# Test 1: Frontend should be able to reach backend
echo "Test 1: Frontend → Backend (should work)"
kubectl exec -it $(kubectl get pod -l app=secure-web-frontend -o jsonpath='{.items[0].metadata.name}') -- wget -qO- --timeout=5 http://secure-api-service:8080 && echo "SUCCESS: Frontend can reach backend" || echo "FAILED: Frontend cannot reach backend"

# Test 2: Frontend should NOT be able to reach database directly
echo "Test 2: Frontend → Database (should fail)"
kubectl exec -it $(kubectl get pod -l app=secure-web-frontend -o jsonpath='{.items[0].metadata.name}') -- nc -zv secure-mysql-service 3306 && echo "FAILED: Frontend can reach database (security issue!)" || echo "SUCCESS: Frontend blocked from database"

# Test 3: Backend should be able to reach database
echo "Test 3: Backend → Database (should work)"
kubectl exec -it $(kubectl get pod -l app=secure-api-backend -o jsonpath='{.items[0].metadata.name}') -- nc -zv secure-mysql-service 3306 && echo "SUCCESS: Backend can reach database" || echo "FAILED: Backend cannot reach database"
Subtask 4.4: Implement Additional Security Controls
Create a Pod Security Policy equivalent using OPA Gatekeeper (if available) or document additional controls:

cat << 'EOF' > additional-security-controls.yaml
# Resource Quota to prevent resource exhaustion
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ecommerce-quota
  namespace: ecommerce-app
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "10"
    services: "5"
---
# Limit Range to enforce resource limits
apiVersion: v1
kind: LimitRange
metadata:
  name: ecommerce-limits
  namespace: ecommerce-app
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
EOF

kubectl apply -f additional-security-controls.yaml
Verify additional controls:

# Check resource quota
kubectl get resourcequota

# Check limit range
kubectl get limitrange

# Describe quota usage
kubectl describe resourcequota ecommerce-quota
Task 5: Validation and Testing
Subtask 5.1: Security Validation Tests
Let's run comprehensive tests to validate our security implementations:

# Create a test script
cat << 'EOF' > security-validation.sh
#!/bin/bash

echo "=== Kubernetes Security Validation Tests ==="
echo

# Test 1: Pod Security Standards Compliance
echo "1. Testing Pod Security Standards Compliance..."
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.securityContext.runAsNonRoot}{"\t"}{.spec.containers[*].securityContext.allowPrivilegeEscalation}{"\n"}{end}' | while read pod runAsNonRoot allowPrivEsc; do
    if [[ "$runAsNonRoot" == "true" && "$allowPrivEsc" == "false" ]]; then
        echo "✓ $pod: Compliant with security standards"
    else
        echo "✗ $pod: Non-compliant with security standards"
    fi
done

echo

# Test 2: Network Policy Enforcement
echo "2. Testing Network Policy Enforcement..."

# Get pod names
FRONTEND_POD=$(kubectl get pod -l app=secure-web-frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_POD=$(kubectl get pod -l app=secure-api-backend -o jsonpath='{.items[0].metadata.name}')

# Test frontend to backend (should work)
if kubectl exec $FRONTEND_POD -- wget -qO- --timeout=5 http://secure-api-service:8080 >/dev/null 2>&1; then
    echo "✓ Frontend → Backend: Allowed (correct)"
else
    echo "✗ Frontend → Backend: Blocked (incorrect)"
fi

# Test frontend to database (should be blocked)
if kubectl exec $FRONTEND_POD -- nc -zv secure-mysql-service 3306 >/dev/null 2>&1; then
    echo "✗ Frontend → Database: Allowed (security risk!)"
else
    echo "✓ Frontend → Database: Blocked (correct)"
fi

# Test backend to database (should work)
if kubectl exec $BACKEND_POD -- nc -zv secure-mysql-service 3306 >/dev/null 2>&1; then
    echo "✓ Backend → Database: Allowed (correct)"
else
    echo "✗ Backend → Database: Blocked (incorrect)"
fi

echo

# Test 3: Resource Limits
echo "3. Testing Resource Limits..."
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.limits}{"\n"}{end}' | while read pod limits; do
    if [[ "$limits" != "{}" && "$limits" != "" ]]; then
        echo "✓ $pod: Has resource limits defined"
    else
        echo "✗ $pod: Missing resource limits"
    fi
done

echo

# Test 4: Secret Usage
echo "4. Testing Secret Usage..."
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].env[?(@.valueFrom.secretKeyRef)]}{"\n"}{end}' | while read pod secrets; do
    if [[ "$secrets" != "" ]]; then
        echo "✓ $pod: Uses secrets for sensitive data"
    else
        echo "? $pod: May have hardcoded credentials"
    fi
done

echo
echo "=== Security Validation Complete ==="
EOF

chmod +x security-validation.sh
./security-validation.sh
Subtask 5.2: Generate Security Report
Create a comprehensive security report:

cat << 'EOF' > security-report.md
# Kubernetes Security Assessment Report

## Executive Summary
This report documents the security assessment and hardening of the e-commerce application deployed on Kubernetes.

## Security Improvements Implemented

### 1. Pod Security Standards
- **Implemented**: Restricted Pod Security Standards
- **Controls**: 
  - runAsNonRoot: true
  - allowPrivilegeEscalation: false
  - readOnlyRootFilesystem: true (where possible)
  - Dropped all capabilities, added only necessary ones
  - Seccomp profile: RuntimeDefault

### 2. Network Segmentation
- **Implemented**: Network policies for micro-segmentation
- **Controls**:
  - Default deny all ingress traffic
  - Allow frontend → backend communication (port 8080)
  - Allow backend → database communication (port 3306)
  - Block direct frontend → database communication

### 3. Secret Management
- **Implemented**: Kubernetes secrets for sensitive data
- **Controls**:
  - Database credentials stored in secrets
  - Environment variables reference secrets
  - No hardcoded passwords in deployments

### 4. Resource Management
- **Implemented**: Resource quotas and limits
- **Controls**:
  - CPU and memory limits per container
  - Namespace-level resource quotas
  - Prevention of resource exhaustion attacks

## Risk Mitigation Summary

| Risk Category | Before | After | Mitigation |
|---------------|--------|-------|------------|
| Privilege Escalation | High | Low | Pod Security Standards |
| Lateral Movement | High | Low | Network Policies |
| Resource Exhaustion | Medium | Low | Resource Limits |
| Credential Exposure | High | Low | Secret Management |
| Container Breakout | High | Medium | Security Contexts |

## Recommendations for Further Improvement

1. **Image Security**:
   - Implement image vulnerability scanning
   - Use minimal base images (distroless)
   - Sign and verify container images

2. **Runtime Security**:
   - Deploy runtime security monitoring (Falco)
   - Implement admission controllers (OPA Gatekeeper)
   - Enable audit logging

3. **Access Control**:
   - Implement RBAC with least privilege
   - Use service accounts with minimal permissions
   - Enable Pod Security Admission

4. **Monitoring and Alerting**:
   - Deploy security
