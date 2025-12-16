Lab 20: API Deprecations
Objectives
By the end of this lab, you will be able to:

• Understand the concept of API deprecation in Kubernetes and its impact on cluster operations • Identify deprecated APIs in existing Kubernetes manifests using built-in tools • Update deprecated API versions to current stable versions • Validate and test updated manifests to ensure functionality is preserved • Implement best practices for managing API version transitions in production environments • Use kubectl tools to check API version compatibility across different Kubernetes versions

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with YAML syntax and Kubernetes manifest structure • Experience using kubectl command-line tool • Understanding of Kubernetes API versioning concepts (alpha, beta, stable) • Knowledge of basic Linux command-line operations

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch. Your lab environment includes:

• Ubuntu 22.04 LTS with kubectl pre-installed • A running Kubernetes cluster (kind or minikube) • All necessary tools for API deprecation analysis • Sample manifests with deprecated APIs ready for testing

Lab Tasks
Task 1: Understanding API Deprecations and Setting Up the Environment
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure your Kubernetes cluster is running and check the current version.

# Check cluster status
kubectl cluster-info

# Check Kubernetes version
kubectl version --short

# Verify nodes are ready
kubectl get nodes
Subtask 1.2: Create Sample Manifests with Deprecated APIs
Create a directory for this lab and sample manifests that contain deprecated APIs.

# Create lab directory
mkdir ~/api-deprecation-lab
cd ~/api-deprecation-lab

# Create a manifest with deprecated APIs (common in older Kubernetes versions)
cat > deprecated-deployment.yaml << 'EOF'
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.local
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
EOF
# Create another manifest with deprecated PodSecurityPolicy
cat > deprecated-psp.yaml << 'EOF'
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
EOF
Subtask 1.3: Understanding API Deprecation Timeline
Let's check what APIs are available in your cluster and understand the deprecation timeline.

# List all available API versions
kubectl api-versions | sort

# Get detailed information about API resources
kubectl api-resources | grep -E "(extensions|policy)" | head -10

# Check for deprecated APIs in your cluster
kubectl get --raw /api/v1 | jq '.resources[] | select(.name == "pods") | .verbs'
Task 2: Identify Deprecated APIs in Sample Manifests
Subtask 2.1: Use kubectl to Validate Manifests
Let's try to apply the deprecated manifests and see what warnings or errors we get.

# Try to apply the deprecated deployment (this may show warnings)
kubectl apply --dry-run=client -f deprecated-deployment.yaml

# Check for API deprecation warnings
kubectl apply --dry-run=server -f deprecated-deployment.yaml
Subtask 2.2: Use kubectl-convert Plugin for API Analysis
Install and use the kubectl-convert plugin to identify deprecated APIs.

# Install kubectl-convert plugin
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert"
chmod +x kubectl-convert
sudo mv kubectl-convert /usr/local/bin/

# Use kubectl-convert to check for deprecated APIs
kubectl-convert -f deprecated-deployment.yaml --output-version apps/v1
Subtask 2.3: Create a Deprecation Analysis Script
Create a script to systematically check for deprecated APIs.

cat > check-deprecations.sh << 'EOF'
#!/bin/bash

echo "=== API Deprecation Analysis ==="
echo "Checking manifests for deprecated APIs..."

# Function to check individual files
check_file() {
    local file=$1
    echo "Analyzing: $file"
    
    # Extract API versions from the file
    grep -n "apiVersion:" "$file" | while read -r line; do
        echo "  Found: $line"
        
        # Check if it's a known deprecated API
        if echo "$line" | grep -q "extensions/v1beta1"; then
            echo "  ⚠️  WARNING: extensions/v1beta1 is deprecated"
        fi
        
        if echo "$line" | grep -q "policy/v1beta1"; then
            echo "  ⚠️  WARNING: policy/v1beta1 is deprecated"
        fi
    done
    echo ""
}

# Check all YAML files in current directory
for file in *.yaml; do
    if [ -f "$file" ]; then
        check_file "$file"
    fi
done

echo "=== Analysis Complete ==="
EOF

chmod +x check-deprecations.sh
./check-deprecations.sh
Task 3: Update Manifests to Use Latest API Versions
Subtask 3.1: Update Deployment API Version
Create updated versions of the manifests using current stable API versions.

# Create updated deployment manifest
cat > updated-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
EOF
Subtask 3.2: Create Service Manifest
Since our Ingress references a service, let's create that as well.

cat > nginx-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF
Subtask 3.3: Replace PodSecurityPolicy with Pod Security Standards
Since PodSecurityPolicy is deprecated and removed in Kubernetes 1.25+, let's create a namespace with Pod Security Standards instead.

cat > pod-security-namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
EOF
Subtask 3.4: Create API Version Comparison
Let's create a comparison document to understand the changes.

cat > api-changes-summary.md << 'EOF'
# API Version Changes Summary

## Deployment Changes
- **Old**: `extensions/v1beta1` (deprecated in 1.9, removed in 1.16)
- **New**: `apps/v1` (stable since 1.9)
- **Key Changes**: No significant changes in structure

## Ingress Changes
- **Old**: `extensions/v1beta1` (deprecated in 1.14, removed in 1.22)
- **New**: `networking.k8s.io/v1` (stable since 1.19)
- **Key Changes**: 
  - `backend.serviceName` → `backend.service.name`
  - `backend.servicePort` → `backend.service.port.number`
  - Added required `pathType` field

## PodSecurityPolicy Changes
- **Old**: `policy/v1beta1` (deprecated in 1.21, removed in 1.25)
- **New**: Pod Security Standards (built-in admission controller)
- **Key Changes**: 
  - No longer uses custom resources
  - Applied via namespace labels
  - Three levels: privileged, baseline, restricted
EOF
Task 4: Test Updated Manifests for Functionality
Subtask 4.1: Validate Updated Manifests
Let's validate our updated manifests before applying them.

# Validate the updated deployment
kubectl apply --dry-run=client -f updated-deployment.yaml
kubectl apply --dry-run=client -f nginx-service.yaml
kubectl apply --dry-run=client -f pod-security-namespace.yaml

# Check for any validation errors
echo "Validation complete. Checking for syntax errors..."
Subtask 4.2: Apply Updated Manifests
Now let's apply the updated manifests to the cluster.

# Apply the service first (dependency for ingress)
kubectl apply -f nginx-service.yaml

# Apply the updated deployment
kubectl apply -f updated-deployment.yaml

# Apply the namespace with pod security standards
kubectl apply -f pod-security-namespace.yaml

# Verify resources are created
kubectl get deployments
kubectl get services
kubectl get ingress
kubectl get namespaces restricted-namespace -o yaml
Subtask 4.3: Test Functionality
Let's verify that our updated resources work correctly.

# Check deployment status
kubectl rollout status deployment/nginx-deployment

# Get pod details
kubectl get pods -l app=nginx

# Test service connectivity (from within cluster)
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- nginx-service

# Check ingress status (if ingress controller is available)
kubectl describe ingress nginx-ingress
Subtask 4.4: Test Pod Security Standards
Let's test the pod security standards in our restricted namespace.

# Try to create a privileged pod in the restricted namespace (should fail)
cat > test-privileged-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
  namespace: restricted-namespace
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF

# This should show warnings/errors due to pod security standards
kubectl apply -f test-privileged-pod.yaml

# Create a compliant pod
cat > test-compliant-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: compliant-test
  namespace: restricted-namespace
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: test
    image: nginx:1.20
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: false
      runAsNonRoot: true
      runAsUser: 1000
EOF

kubectl apply -f test-compliant-pod.yaml
kubectl get pods -n restricted-namespace
Task 5: Implement Best Practices for API Version Management
Subtask 5.1: Create API Version Monitoring Script
Create a script to regularly check for deprecated APIs in your cluster.

cat > monitor-api-versions.sh << 'EOF'
#!/bin/bash

echo "=== Kubernetes API Version Monitor ==="
echo "Cluster Version: $(kubectl version --short --client)"
echo "Server Version: $(kubectl version --short --output=json | jq -r '.serverVersion.gitVersion')"
echo ""

# Function to check for deprecated APIs in running resources
check_running_resources() {
    echo "Checking running resources for deprecated APIs..."
    
    # Check deployments
    echo "Deployments using extensions/v1beta1:"
    kubectl get deployments --all-namespaces -o json | jq -r '.items[] | select(.apiVersion == "extensions/v1beta1") | "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null || echo "None found"
    
    # Check ingresses
    echo "Ingresses using extensions/v1beta1:"
    kubectl get ingresses --all-namespaces -o json | jq -r '.items[] | select(.apiVersion == "extensions/v1beta1") | "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null || echo "None found"
    
    echo ""
}

# Function to check API server capabilities
check_api_capabilities() {
    echo "Available API versions:"
    kubectl api-versions | grep -E "(apps|networking|policy)" | sort
    echo ""
    
    echo "Deprecated APIs still available:"
    kubectl api-versions | grep -E "(extensions/v1beta1|policy/v1beta1)" || echo "None found (good!)"
    echo ""
}

check_running_resources
check_api_capabilities

echo "=== Monitor Complete ==="
EOF

chmod +x monitor-api-versions.sh
./monitor-api-versions.sh
Subtask 5.2: Create Migration Checklist
Create a comprehensive checklist for API migrations.

cat > api-migration-checklist.md << 'EOF'
# API Migration Checklist

## Pre-Migration Assessment
- [ ] Identify all manifests using deprecated APIs
- [ ] Check Kubernetes cluster version compatibility
- [ ] Review breaking changes in API documentation
- [ ] Test migrations in development environment
- [ ] Backup existing configurations

## Migration Process
- [ ] Update API versions in manifest files
- [ ] Modify field names and structure as required
- [ ] Validate updated manifests with dry-run
- [ ] Test functionality in staging environment
- [ ] Plan rollback strategy

## Post-Migration Verification
- [ ] Verify all resources are running correctly
- [ ] Check application functionality
- [ ] Monitor for any errors or warnings
- [ ] Update documentation and runbooks
- [ ] Schedule regular API deprecation checks

## Common API Migrations

### Deployment (extensions/v1beta1 → apps/v1)
- No structural changes required
- Simply update apiVersion field

### Ingress (extensions/v1beta1 → networking.k8s.io/v1)
- Update backend field structure
- Add pathType field (required)
- Update serviceName/servicePort to service.name/service.port

### PodSecurityPolicy (policy/v1beta1 → Pod Security Standards)
- Replace PSP resources with namespace labels
- Use pod-security.kubernetes.io/* labels
- Choose appropriate security level (privileged/baseline/restricted)
EOF
Subtask 5.3: Set Up Automated Deprecation Alerts
Create a script that can be run periodically to alert about deprecated APIs.

cat > deprecation-alert.sh << 'EOF'
#!/bin/bash

# Configuration
ALERT_FILE="/tmp/k8s-deprecation-alerts.log"
EMAIL_ALERT=false  # Set to true if you want email alerts

# Function to log alerts
log_alert() {
    local message="$1"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] ALERT: $message" | tee -a "$ALERT_FILE"
}

# Check for deprecated APIs in use
check_deprecated_apis() {
    echo "Scanning for deprecated APIs..."
    
    # Check for extensions/v1beta1 deployments
    deprecated_deployments=$(kubectl get deployments --all-namespaces -o json 2>/dev/null | jq -r '.items[] | select(.apiVersion == "extensions/v1beta1") | "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null)
    
    if [ ! -z "$deprecated_deployments" ]; then
        log_alert "Found deployments using deprecated extensions/v1beta1 API: $deprecated_deployments"
    fi
    
    # Check for extensions/v1beta1 ingresses
    deprecated_ingresses=$(kubectl get ingresses --all-namespaces -o json 2>/dev/null | jq -r '.items[] | select(.apiVersion == "extensions/v1beta1") | "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null)
    
    if [ ! -z "$deprecated_ingresses" ]; then
        log_alert "Found ingresses using deprecated extensions/v1beta1 API: $deprecated_ingresses"
    fi
    
    # Check for PodSecurityPolicies
    deprecated_psp=$(kubectl get psp 2>/dev/null | tail -n +2 | wc -l)
    
    if [ "$deprecated_psp" -gt 0 ]; then
        log_alert "Found $deprecated_psp PodSecurityPolicy resources (deprecated in 1.21, removed in 1.25)"
    fi
}

# Main execution
echo "Starting deprecation check at $(date)"
check_deprecated_apis

if [ -f "$ALERT_FILE" ] && [ -s "$ALERT_FILE" ]; then
    echo "Alerts generated. Check $ALERT_FILE for details."
    if [ "$EMAIL_ALERT" = true ]; then
        echo "Email alerts would be sent here (configure your email system)"
    fi
else
    echo "No deprecated APIs found. Cluster is up to date!"
fi
EOF

chmod +x deprecation-alert.sh
./deprecation-alert.sh
Troubleshooting Common Issues
Issue 1: Manifest Validation Errors
If you encounter validation errors when applying updated manifests:

# Check the exact error message
kubectl apply -f updated-deployment.yaml --validate=true

# Use explain to understand required fields
kubectl explain ingress.spec.rules.http.paths.backend

# Verify API version availability
kubectl api-resources | grep ingress
Issue 2: Ingress Not Working
If the updated ingress doesn't work:

# Check if ingress controller is installed
kubectl get pods -n ingress-nginx

# Verify ingress class
kubectl get ingressclass

# Check ingress events
kubectl describe ingress nginx-ingress
Issue 3: Pod Security Standards Blocking Pods
If pods are blocked by pod security standards:

# Check namespace labels
kubectl get namespace restricted-namespace -o yaml

# Review pod security violations
kubectl get events -n restricted-namespace

# Test with a minimal compliant pod
kubectl run test --image=nginx --dry-run=server -n restricted-namespace
Cleanup
Clean up the resources created during this lab:

# Remove test resources
kubectl delete -f updated-deployment.yaml
kubectl delete -f nginx-service.yaml
kubectl delete -f pod-security-namespace.yaml
kubectl delete pod test-pod --ignore-not-found
kubectl delete -f test-compliant-pod.yaml --ignore-not-found

# Remove lab files
cd ~
rm -rf api-deprecation-lab
Conclusion
In this lab, you have successfully:

• Identified deprecated APIs in Kubernetes manifests using various tools and techniques, understanding how API deprecation affects cluster operations and application deployments

• Updated manifests from deprecated API versions (extensions/v1beta1, policy/v1beta1) to current stable versions (apps/v1, networking.k8s.io/v1, Pod Security Standards), learning the structural changes required for each API migration

• Tested functionality of updated manifests to ensure that applications continue to work correctly after API version updates, validating that no functionality is lost during the migration process

• Implemented monitoring and alerting systems to proactively identify deprecated APIs in your cluster, creating automated tools that can be integrated into CI/CD pipelines for ongoing API version management

This knowledge is crucial for maintaining Kubernetes clusters as they evolve, ensuring that your applications remain compatible with newer Kubernetes versions, and avoiding disruptions when deprecated APIs are removed. The skills you've learned here will help you manage API deprecations proactively in production environments, making you better prepared for the CKAD certification and real-world Kubernetes operations.

Understanding API deprecation management is essential for any Kubernetes administrator or developer, as it directly impacts application availability and cluster upgrade strategies. The automated monitoring scripts and migration checklists you've created can be adapted for use in production environments to maintain cluster health and compatibility.
