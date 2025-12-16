Lab 1: Understanding the 4Cs of Cloud-Native Security
Lab Objectives
By the end of this lab, students will be able to:

• Understand the four layers of cloud-native security: Code, Container, Cluster, and Cloud • Analyze security considerations and best practices for each of the 4Cs • Configure workload isolation using Kubernetes namespaces and Role-Based Access Control (RBAC) • Implement container security scanning using open-source tools • Identify and remediate common security vulnerabilities in containerized applications • Apply defense-in-depth security principles across all four layers

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Linux command line operations • Fundamental knowledge of containers and Docker concepts • Basic familiarity with Kubernetes architecture and components • Understanding of YAML file structure and syntax • Knowledge of basic networking concepts

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 22.04 LTS with Docker installed • Kubernetes cluster (minikube) pre-configured • kubectl command-line tool • Trivy vulnerability scanner • Git for code management

Task 1: Analyzing Security Considerations for the 4Cs
Subtask 1.1: Understanding the 4Cs Framework
The 4Cs of cloud-native security represent layers of defense:

Code - Application source code and dependencies
Container - Container images and runtime security
Cluster - Kubernetes cluster configuration and policies
Cloud - Infrastructure and cloud provider security
Let's explore each layer in detail.

Subtask 1.2: Code Layer Security Analysis
First, let's examine a sample application and identify code-level security considerations.

# Create a working directory
mkdir ~/cloud-native-security-lab
cd ~/cloud-native-security-lab

# Clone a sample vulnerable application
git clone https://github.com/OWASP/NodeGoat.git
cd NodeGoat
Examine the application structure:

# List the application files
ls -la

# Check for sensitive files that shouldn't be in version control
find . -name "*.env" -o -name "*.key" -o -name "*.pem" -o -name "config.json"

# Look for hardcoded secrets in the code
grep -r "password\|secret\|key" --include="*.js" --include="*.json" . | head -10
Code Layer Security Checklist: • Scan for hardcoded secrets and credentials • Review dependencies for known vulnerabilities • Implement secure coding practices • Use static code analysis tools • Maintain an updated software bill of materials (SBOM)

Subtask 1.3: Container Layer Security Analysis
Now let's examine container security by building and analyzing a container image.

Create a Dockerfile for our sample application:

# Create a Dockerfile
cat > Dockerfile << 'EOF'
FROM node:14-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 4000
USER node
CMD ["npm", "start"]
EOF
Build the container image:

# Build the container image
docker build -t nodegoat:vulnerable .

# List the created image
docker images | grep nodegoat
Container Layer Security Checklist: • Use minimal base images • Run containers as non-root users • Scan images for vulnerabilities • Keep base images updated • Remove unnecessary packages and files

Subtask 1.4: Cluster Layer Security Analysis
Let's examine Kubernetes cluster security configurations.

Start your Kubernetes cluster:

# Start minikube if not already running
minikube start

# Verify cluster status
kubectl cluster-info

# Check current security context
kubectl auth whoami
Examine default cluster security settings:

# List all namespaces
kubectl get namespaces

# Check default service accounts
kubectl get serviceaccounts --all-namespaces

# Review cluster roles and bindings
kubectl get clusterroles | head -10
kubectl get clusterrolebindings | head -10
Cluster Layer Security Checklist: • Implement network policies • Configure RBAC properly • Use namespaces for isolation • Enable audit logging • Secure the API server

Subtask 1.5: Cloud Layer Security Analysis
Examine the underlying infrastructure security:

# Check node security information
kubectl get nodes -o wide

# Review node system information
kubectl describe node minikube | grep -A 10 "System Info"

# Check for security-related node conditions
kubectl get nodes -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].message}'
Cloud Layer Security Checklist: • Secure network configurations • Implement proper IAM policies • Enable logging and monitoring • Regular security updates • Backup and disaster recovery

Task 2: Configuring Workload Security with Namespaces and RBAC
Subtask 2.1: Creating Secure Namespaces
Create isolated namespaces for different environments:

# Create development namespace
kubectl create namespace development

# Create production namespace
kubectl create namespace production

# Create security namespace for security tools
kubectl create namespace security

# Verify namespaces were created
kubectl get namespaces
Add labels to namespaces for better organization:

# Label namespaces
kubectl label namespace development environment=dev
kubectl label namespace production environment=prod
kubectl label namespace security purpose=security-tools

# View labeled namespaces
kubectl get namespaces --show-labels
Subtask 2.2: Implementing Network Policies
Create network policies to isolate workloads:

# Create a network policy for the development namespace
cat > dev-network-policy.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dev-isolation
  namespace: development
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: dev
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          environment: dev
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
EOF

# Apply the network policy
kubectl apply -f dev-network-policy.yaml

# Verify the policy was created
kubectl get networkpolicies -n development
Subtask 2.3: Setting Up RBAC
Create service accounts with specific permissions:

# Create a service account for development
kubectl create serviceaccount dev-user -n development

# Create a service account for production (read-only)
kubectl create serviceaccount prod-reader -n production
Create custom roles:

# Create a role for development namespace
cat > dev-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: dev-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
EOF

# Create a read-only role for production
cat > prod-reader-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: prod-reader-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list"]
EOF

# Apply the roles
kubectl apply -f dev-role.yaml
kubectl apply -f prod-reader-role.yaml
Create role bindings:

# Bind the dev-role to dev-user
cat > dev-rolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: development
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: development
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Bind the prod-reader-role to prod-reader
cat > prod-reader-rolebinding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prod-reader-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: prod-reader
  namespace: production
roleRef:
  kind: Role
  name: prod-reader-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Apply the role bindings
kubectl apply -f dev-rolebinding.yaml
kubectl apply -f prod-reader-rolebinding.yaml
Subtask 2.4: Testing RBAC Permissions
Test the RBAC configuration:

# Test dev-user permissions in development namespace
kubectl auth can-i create pods --as=system:serviceaccount:development:dev-user -n development

# Test dev-user permissions in production namespace (should be denied)
kubectl auth can-i create pods --as=system:serviceaccount:development:dev-user -n production

# Test prod-reader permissions
kubectl auth can-i get pods --as=system:serviceaccount:production:prod-reader -n production
kubectl auth can-i create pods --as=system:serviceaccount:production:prod-reader -n production
Subtask 2.5: Deploying a Secure Workload
Deploy a sample application with security best practices:

# Create a secure deployment
cat > secure-app-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: development
  labels:
    app: secure-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      serviceAccountName: dev-user
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: secure-app
        image: nginx:1.21-alpine
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: var-cache-volume
          mountPath: /var/cache/nginx
        - name: var-run-volume
          mountPath: /var/run
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: var-cache-volume
        emptyDir: {}
      - name: var-run-volume
        emptyDir: {}
EOF

# Deploy the secure application
kubectl apply -f secure-app-deployment.yaml

# Verify the deployment
kubectl get deployments -n development
kubectl get pods -n development
Task 3: Performing Container Vulnerability Scanning
Subtask 3.1: Installing and Configuring Trivy
Trivy is already installed in your lab environment. Let's verify and configure it:

# Check Trivy version
trivy version

# Update Trivy database
trivy image --download-db-only
Subtask 3.2: Scanning Container Images
Scan the NodeGoat image we built earlier:

# Scan the vulnerable NodeGoat image
trivy image nodegoat:vulnerable

# Scan with specific severity levels
trivy image --severity HIGH,CRITICAL nodegoat:vulnerable

# Generate a detailed report
trivy image --format json --output nodegoat-scan-results.json nodegoat:vulnerable
Scan a base image for comparison:

# Scan the base Node.js image
trivy image node:14-alpine

# Compare with a newer version
trivy image node:18-alpine
Subtask 3.3: Analyzing Scan Results
Examine the scan results:

# View the JSON report
cat nodegoat-scan-results.json | jq '.Results[0].Vulnerabilities | length'

# Count vulnerabilities by severity
cat nodegoat-scan-results.json | jq '.Results[0].Vulnerabilities | group_by(.Severity) | map({severity: .[0].Severity, count: length})'

# List critical vulnerabilities
cat nodegoat-scan-results.json | jq '.Results[0].Vulnerabilities[] | select(.Severity == "CRITICAL") | {VulnerabilityID, PkgName, InstalledVersion, FixedVersion}'
Subtask 3.4: Creating a Secure Container Image
Create an improved Dockerfile addressing the vulnerabilities:

# Create a more secure Dockerfile
cat > Dockerfile.secure << 'EOF'
# Use a more recent and secure base image
FROM node:18-alpine3.17

# Create a non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies and clean up
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/* /var/tmp/*

# Copy application code
COPY --chown=nextjs:nodejs . .

# Remove unnecessary files
RUN rm -rf .git .gitignore README.md Dockerfile*

# Use non-root user
USER nextjs

# Expose port
EXPOSE 4000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:4000/health || exit 1

# Start the application
CMD ["npm", "start"]
EOF

# Build the secure image
docker build -f Dockerfile.secure -t nodegoat:secure .
Subtask 3.5: Comparing Scan Results
Scan the improved image:

# Scan the secure image
trivy image nodegoat:secure

# Compare vulnerability counts
echo "Vulnerable image vulnerabilities:"
trivy image --severity HIGH,CRITICAL --format json nodegoat:vulnerable | jq '.Results[0].Vulnerabilities | length'

echo "Secure image vulnerabilities:"
trivy image --severity HIGH,CRITICAL --format json nodegoat:secure | jq '.Results[0].Vulnerabilities | length'
Subtask 3.6: Implementing Continuous Scanning
Create a script for automated scanning:

# Create a scanning script
cat > scan-images.sh << 'EOF'
#!/bin/bash

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to scan image
scan_image() {
    local image=$1
    echo -e "${YELLOW}Scanning image: $image${NC}"
    
    # Get vulnerability count
    local vuln_count=$(trivy image --severity HIGH,CRITICAL --format json $image 2>/dev/null | jq '.Results[0].Vulnerabilities | length' 2>/dev/null || echo "0")
    
    if [ "$vuln_count" -eq 0 ]; then
        echo -e "${GREEN}✓ No HIGH/CRITICAL vulnerabilities found${NC}"
        return 0
    else
        echo -e "${RED}✗ Found $vuln_count HIGH/CRITICAL vulnerabilities${NC}"
        return 1
    fi
}

# Scan multiple images
images=("nodegoat:vulnerable" "nodegoat:secure" "nginx:1.21-alpine")

for image in "${images[@]}"; do
    scan_image $image
    echo "---"
done
EOF

# Make the script executable
chmod +x scan-images.sh

# Run the scanning script
./scan-images.sh
Subtask 3.7: Setting Up Admission Controllers
Create a policy to prevent vulnerable images from being deployed:

# Create a simple admission controller policy (for demonstration)
cat > image-security-policy.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: image-security-policy
  namespace: security
data:
  policy.rego: |
    package kubernetes.admission
    
    deny[msg] {
        input.request.kind.kind == "Pod"
        input.request.object.spec.containers[_].image
        contains(input.request.object.spec.containers[_].image, ":latest")
        msg := "Using 'latest' tag is not allowed for security reasons"
    }
    
    deny[msg] {
        input.request.kind.kind == "Pod"
        input.request.object.spec.securityContext.runAsRoot == true
        msg := "Running containers as root is not allowed"
    }
EOF

# Apply the policy
kubectl apply -f image-security-policy.yaml
Task 4: Implementing Security Monitoring and Compliance
Subtask 4.1: Setting Up Security Monitoring
Create a monitoring namespace and deploy basic monitoring:

# Create monitoring resources
kubectl create namespace monitoring

# Create a simple security monitoring deployment
cat > security-monitor.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: security-monitor
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: security-monitor
  template:
    metadata:
      labels:
        app: security-monitor
    spec:
      containers:
      - name: monitor
        image: alpine:3.17
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo 'Security monitoring active at $(date)'; sleep 60; done"]
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
EOF

kubectl apply -f security-monitor.yaml
Subtask 4.2: Creating Security Compliance Checks
Create a script to check security compliance:

# Create compliance check script
cat > security-compliance-check.sh << 'EOF'
#!/bin/bash

echo "=== Kubernetes Security Compliance Check ==="
echo

# Check 1: Verify RBAC is enabled
echo "1. Checking RBAC configuration..."
if kubectl auth can-i '*' '*' --as=system:anonymous 2>/dev/null; then
    echo "❌ RBAC might not be properly configured"
else
    echo "✅ RBAC appears to be properly configured"
fi

# Check 2: Look for pods running as root
echo
echo "2. Checking for pods running as root..."
root_pods=$(kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.securityContext.runAsUser}{"\n"}{end}' | grep -c " 0$" || true)
if [ "$root_pods" -gt 0 ]; then
    echo "❌ Found $root_pods pods potentially running as root"
else
    echo "✅ No pods found running as root"
fi

# Check 3: Verify network policies exist
echo
echo "3. Checking for network policies..."
network_policies=$(kubectl get networkpolicies --all-namespaces --no-headers 2>/dev/null | wc -l)
if [ "$network_policies" -eq 0 ]; then
    echo "❌ No network policies found"
else
    echo "✅ Found $network_policies network policies"
fi

# Check 4: Look for default service accounts in use
echo
echo "4. Checking for default service account usage..."
default_sa_pods=$(kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.spec.serviceAccountName}{"\n"}{end}' | grep -c "^$\|^default$" || true)
if [ "$default_sa_pods" -gt 0 ]; then
    echo "❌ Found $default_sa_pods pods using default service account"
else
    echo "✅ No pods using default service account"
fi

# Check 5: Verify resource limits are set
echo
echo "5. Checking for resource limits..."
no_limits_pods=$(kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.containers[*].resources.limits}{"\n"}{end}' | grep -c "^[^ ]* $" || true)
if [ "$no_limits_pods" -gt 0 ]; then
    echo "❌ Found $no_limits_pods pods without resource limits"
else
    echo "✅ All pods have resource limits configured"
fi

echo
echo "=== Compliance Check Complete ==="
EOF

# Make executable and run
chmod +x security-compliance-check.sh
./security-compliance-check.sh
Troubleshooting Common Issues
Issue 1: Trivy Database Update Fails
# If Trivy database update fails, try:
trivy image --skip-update nodegoat:vulnerable

# Or clear cache and retry:
trivy image --clear-cache
trivy image --download-db-only
Issue 2: RBAC Permission Denied
# Check current user permissions:
kubectl auth whoami
kubectl auth can-i --list

# Verify service account exists:
kubectl get serviceaccounts -n development
Issue 3: Network Policy Not Working
# Verify network policy support:
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.containerRuntimeVersion}'

# Check if CNI supports network policies:
kubectl describe node minikube | grep -i network
Issue 4: Container Build Failures
# Check Docker daemon status:
docker version

# Clean up Docker resources if needed:
docker system prune -f

# Rebuild with verbose output:
docker build --no-cache -t nodegoat:secure .
Lab Summary and Key Takeaways
What You Accomplished
In this lab, you successfully:

Analyzed the 4Cs of Cloud-Native Security:

Examined code-level security considerations including secret management and dependency scanning
Evaluated container security through image analysis and vulnerability scanning
Implemented cluster security using namespaces, RBAC, and network policies
Reviewed cloud infrastructure security fundamentals
Configured Secure Workload Isolation:

Created isolated namespaces for different environments
Implemented Role-Based Access Control (RBAC) with custom roles and service accounts
Applied network policies to restrict inter-namespace communication
Deployed applications following security best practices
Performed Comprehensive Vulnerability Scanning:

Used Trivy to scan container images for security vulnerabilities
Analyzed scan results and identified critical security issues
Created improved container images with reduced attack surface
Implemented automated scanning workflows
Established Security Monitoring and Compliance:

Set up basic security monitoring infrastructure
Created compliance checking scripts to validate security configurations
Implemented policies to prevent insecure deployments
Why This Matters
Understanding and implementing the 4Cs of cloud-native security is crucial because:

Defense in Depth: Each layer provides additional security controls, creating multiple barriers against threats
Shared Responsibility: Cloud-native security requires securing every layer from code to infrastructure
Compliance Requirements: Many regulatory frameworks require comprehensive security controls across all layers
Risk Mitigation: Vulnerabilities in any layer can compromise the entire application stack
Operational Excellence: Proper security implementation enables confident deployment and scaling of applications
Next Steps
To continue building your cloud-native security expertise:

Explore advanced Kubernetes security features like Pod Security Standards and OPA Gatekeeper
Learn about service mesh security with tools like Istio
Implement comprehensive logging and monitoring with security-focused tools
Study container runtime security and behavioral analysis
Practice incident response procedures for containerized environments
This lab provided hands-on experience with fundamental cloud-native security concepts that form the foundation for the Kubernetes and Cloud Native Security Associate (KCSA) certification and real-world security implementations.
