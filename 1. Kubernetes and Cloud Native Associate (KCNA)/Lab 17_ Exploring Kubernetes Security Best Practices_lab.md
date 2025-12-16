Lab 17: Exploring Kubernetes Security Best Practices
Objectives
By the end of this lab, you will be able to:

• Understand and implement Pod Security Standards in Kubernetes clusters • Configure and apply Network Policies to control Pod-to-Pod communication • Set up image vulnerability scanning using Trivy • Implement security best practices for container workloads • Troubleshoot common security configuration issues in Kubernetes

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with YAML configuration files • Basic knowledge of Linux command line operations • Understanding of container security concepts • Completion of previous Kubernetes labs or equivalent experience

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 22.04 LTS with kubectl pre-configured • Kubernetes cluster (kind or minikube) ready to use • All necessary tools pre-installed • Internet access for downloading container images

Task 1: Implementing Pod Security Standards
Pod Security Standards replace the deprecated PodSecurityPolicies and provide a simpler way to enforce security policies.

Subtask 1.1: Understanding Pod Security Standards
Pod Security Standards define three security profiles: • Privileged: Unrestricted policy (no restrictions) • Baseline: Minimally restrictive policy (prevents known privilege escalations) • Restricted: Heavily restricted policy (follows Pod hardening best practices)

Subtask 1.2: Enable Pod Security Admission
First, let's check if Pod Security Admission is enabled in your cluster:

kubectl api-versions | grep admissionregistration
Create a namespace with Pod Security Standards enforcement:

# Create a restricted namespace
kubectl create namespace secure-apps

# Apply Pod Security Standards labels
kubectl label namespace secure-apps \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
Subtask 1.3: Test Pod Security Standards
Create a test deployment that violates security policies:

# Create file: insecure-pod.yaml
cat << 'EOF' > insecure-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  namespace: secure-apps
spec:
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      privileged: true
      runAsUser: 0
    ports:
    - containerPort: 80
EOF
Try to apply this insecure pod:

kubectl apply -f insecure-pod.yaml
You should see warnings or errors about security policy violations.

Subtask 1.4: Create a Compliant Pod
Now create a security-compliant pod:

# Create file: secure-pod.yaml
cat << 'EOF' > secure-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: secure-apps
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
    image: nginx:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    ports:
    - containerPort: 8080
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
Apply the secure pod:

kubectl apply -f secure-pod.yaml
Verify the pod is running:

kubectl get pods -n secure-apps
kubectl describe pod secure-pod -n secure-apps
Task 2: Implementing Network Policies
Network Policies control traffic flow between Pods and other network endpoints.

Subtask 2.1: Prepare the Environment
Create namespaces for our network policy demonstration:

# Create namespaces
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database

# Label namespaces for easy identification
kubectl label namespace frontend tier=frontend
kubectl label namespace backend tier=backend
kubectl label namespace database tier=database
Subtask 2.2: Deploy Test Applications
Deploy applications in each namespace:

# Create file: frontend-app.yaml
cat << 'EOF' > frontend-app.yaml
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
        image: nginx:alpine
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
# Create file: backend-app.yaml
cat << 'EOF' > backend-app.yaml
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
        image: httpd:alpine
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
# Create file: database-app.yaml
cat << 'EOF' > database-app.yaml
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
      - name: postgres
        image: postgres:alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "password123"
        ports:
        - containerPort: 5432
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
  - port: 5432
    targetPort: 5432
  type: ClusterIP
EOF
Deploy all applications:

kubectl apply -f frontend-app.yaml
kubectl apply -f backend-app.yaml
kubectl apply -f database-app.yaml
Verify deployments:

kubectl get pods -n frontend
kubectl get pods -n backend
kubectl get pods -n database
Subtask 2.3: Test Initial Connectivity
Before applying network policies, test connectivity between namespaces:

# Get a frontend pod name
FRONTEND_POD=$(kubectl get pods -n frontend -o jsonpath='{.items[0].metadata.name}')

# Test connectivity to backend
kubectl exec -n frontend $FRONTEND_POD -- wget -qO- --timeout=2 http://backend-service.backend.svc.cluster.local

# Test connectivity to database
kubectl exec -n frontend $FRONTEND_POD -- nc -zv database-service.database.svc.cluster.local 5432
Subtask 2.4: Implement Network Policies
Create a network policy to restrict database access:

# Create file: database-network-policy.yaml
cat << 'EOF' > database-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  egress:
  - {}  # Allow all egress traffic
EOF
Create a network policy for backend services:

# Create file: backend-network-policy.yaml
cat << 'EOF' > backend-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to: {}  # Allow DNS resolution
    ports:
    - protocol: UDP
      port: 53
EOF
Apply the network policies:

kubectl apply -f database-network-policy.yaml
kubectl apply -f backend-network-policy.yaml
Subtask 2.5: Test Network Policy Enforcement
Test that the database is now protected:

# This should fail - frontend cannot directly access database
kubectl exec -n frontend $FRONTEND_POD -- nc -zv database-service.database.svc.cluster.local 5432

# This should work - frontend can access backend
kubectl exec -n frontend $FRONTEND_POD -- wget -qO- --timeout=2 http://backend-service.backend.svc.cluster.local

# Test from backend to database (should work)
BACKEND_POD=$(kubectl get pods -n backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n backend $BACKEND_POD -- nc -zv database-service.database.svc.cluster.local 5432
Task 3: Container Image Scanning with Trivy
Trivy is an open-source vulnerability scanner for containers and other artifacts.

Subtask 3.1: Install Trivy
Install Trivy on your system:

# Download and install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.48.0

# Verify installation
trivy version
Subtask 3.2: Scan Container Images
Scan a container image for vulnerabilities:

# Scan nginx image
trivy image nginx:latest

# Scan with specific severity levels
trivy image --severity HIGH,CRITICAL nginx:latest

# Generate JSON report
trivy image --format json --output nginx-scan.json nginx:latest
Subtask 3.3: Scan Images in Kubernetes
Create a script to scan all images in your cluster:

# Create file: scan-cluster-images.sh
cat << 'EOF' > scan-cluster-images.sh
#!/bin/bash

echo "Scanning all container images in the cluster..."

# Get all unique images from all namespaces
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u > cluster-images.txt

# Scan each image
while IFS= read -r image; do
    echo "Scanning image: $image"
    trivy image --severity HIGH,CRITICAL --quiet "$image"
    echo "---"
done < cluster-images.txt

rm cluster-images.txt
EOF

chmod +x scan-cluster-images.sh
Run the cluster image scan:

./scan-cluster-images.sh
Subtask 3.4: Implement Image Scanning in CI/CD
Create a sample admission controller configuration that would reject images with high vulnerabilities:

# Create file: image-policy.yaml
cat << 'EOF' > image-policy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: image-security-policy
  namespace: kube-system
data:
  policy.yaml: |
    apiVersion: v1
    kind: Policy
    rules:
    - name: "scan-images"
      match:
      - apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
      validate:
        message: "Images must be scanned and have no HIGH or CRITICAL vulnerabilities"
        pattern:
          spec:
            containers:
            - name: "*"
              image: "!*:latest"  # Discourage latest tags
EOF
Subtask 3.5: Create Secure Image Build Process
Create a sample Dockerfile with security best practices:

# Create file: Dockerfile.secure
cat << 'EOF' > Dockerfile.secure
# Use specific version, not latest
FROM nginx:1.25-alpine

# Create non-root user
RUN addgroup -g 1001 -S nginx-user && \
    adduser -u 1001 -D -S -G nginx-user nginx-user

# Remove unnecessary packages and clean cache
RUN apk del --purge wget curl && \
    rm -rf /var/cache/apk/*

# Set proper permissions
RUN chown -R nginx-user:nginx-user /var/cache/nginx && \
    chown -R nginx-user:nginx-user /var/log/nginx && \
    chown -R nginx-user:nginx-user /etc/nginx/conf.d

# Use non-root user
USER nginx-user

# Expose non-privileged port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
EOF
Task 4: Additional Security Configurations
Subtask 4.1: Configure RBAC (Role-Based Access Control)
Create a service account with limited permissions:

# Create file: rbac-config.yaml
cat << 'EOF' > rbac-config.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: secure-apps
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: secure-apps
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: secure-apps
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: secure-apps
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
Apply RBAC configuration:

kubectl apply -f rbac-config.yaml
Subtask 4.2: Configure Resource Quotas and Limits
Create resource quotas to prevent resource exhaustion:

# Create file: resource-quota.yaml
cat << 'EOF' > resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: secure-apps-quota
  namespace: secure-apps
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "10"
    services: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: secure-apps-limits
  namespace: secure-apps
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
Apply resource constraints:

kubectl apply -f resource-quota.yaml
Subtask 4.3: Verify Security Configurations
Check all security configurations:

# Check Pod Security Standards
kubectl get namespace secure-apps --show-labels

# Check Network Policies
kubectl get networkpolicies --all-namespaces

# Check RBAC
kubectl get serviceaccounts -n secure-apps
kubectl get roles -n secure-apps
kubectl get rolebindings -n secure-apps

# Check Resource Quotas
kubectl get resourcequota -n secure-apps
kubectl describe resourcequota secure-apps-quota -n secure-apps
Troubleshooting Common Issues
Issue 1: Pod Security Standards Not Working
Problem: Pods are not being blocked by security policies.

Solution:

# Check if Pod Security Admission is enabled
kubectl api-versions | grep admissionregistration

# Verify namespace labels
kubectl get namespace secure-apps --show-labels

# Check cluster configuration
kubectl get nodes -o wide
Issue 2: Network Policies Not Enforcing
Problem: Network policies are not blocking traffic.

Solution:

# Check if your CNI supports Network Policies
kubectl get pods -n kube-system | grep -E "(calico|cilium|weave)"

# Verify network policy syntax
kubectl describe networkpolicy database-policy -n database

# Test with verbose output
kubectl exec -n frontend $FRONTEND_POD -- nc -zv database-service.database.svc.cluster.local 5432
Issue 3: Trivy Scanning Errors
Problem: Trivy fails to scan images.

Solution:

# Update Trivy database
trivy image --download-db-only

# Check internet connectivity
curl -I https://github.com

# Scan with debug output
trivy image --debug nginx:latest
Lab Validation
Verify your lab completion by running these validation commands:

# Check Pod Security Standards
echo "=== Pod Security Standards ==="
kubectl get pods -n secure-apps
kubectl describe pod secure-pod -n secure-apps | grep -A 10 "Security Context"

# Check Network Policies
echo "=== Network Policies ==="
kubectl get networkpolicies --all-namespaces

# Check Image Scanning
echo "=== Image Scanning ==="
trivy image --severity HIGH,CRITICAL --quiet nginx:alpine | head -10

# Check RBAC
echo "=== RBAC Configuration ==="
kubectl auth can-i get pods --as=system:serviceaccount:secure-apps:app-service-account -n secure-apps

# Check Resource Quotas
echo "=== Resource Quotas ==="
kubectl describe resourcequota secure-apps-quota -n secure-apps
Conclusion
Congratulations! You have successfully completed Lab 17: Exploring Kubernetes Security Best Practices. In this comprehensive lab, you have:

• Implemented Pod Security Standards to enforce security policies at the pod level, replacing deprecated PodSecurityPolicies with a more modern and flexible approach • Configured Network Policies to control traffic flow between pods and namespaces, implementing a zero-trust network security model • Set up container image vulnerability scanning using Trivy to identify and address security vulnerabilities before deployment • Applied additional security configurations including RBAC, resource quotas, and security contexts

Why This Matters: Security in Kubernetes is not optional—it's essential. As containerized applications become more prevalent in production environments, implementing these security best practices helps protect against:

Container breakouts and privilege escalation attacks
Lateral movement within the cluster through network segmentation
Vulnerable dependencies in container images
Resource exhaustion attacks through proper quotas and limits
Unauthorized access through proper RBAC implementation
These skills are fundamental for the Kubernetes and Cloud Native Associate (KCNA) certification and are critical for anyone working with Kubernetes in production environments. The security practices you've learned today form the foundation of a comprehensive Kubernetes security strategy that protects both your applications and your infrastructure.

Remember to regularly update your security policies, scan images for new vulnerabilities, and review access controls as your applications and teams evolve. Security is an ongoing process, not a one-time configuration.
