Lab 20: Automating Compliance Checks
Objectives
By the end of this lab, students will be able to:

• Install and configure compliance checking tools (kube-bench and Polaris) in a Kubernetes cluster • Generate comprehensive compliance reports for cluster security posture • Analyze compliance findings and understand security implications • Create automated remediation scripts for common compliance issues • Implement policy-based automation for continuous compliance monitoring • Understand the importance of compliance automation in cloud-native environments

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (pods, deployments, services) • Familiarity with Linux command line operations • Knowledge of YAML configuration files • Understanding of basic security concepts • Experience with kubectl commands • Familiarity with shell scripting basics

Lab Environment
Al Nafi provides ready-to-use Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl configured • A running Kubernetes cluster (single-node for lab purposes) • Internet access for downloading tools • Pre-configured user with sudo privileges

Task 1: Installing Compliance Checking Tools
Subtask 1.1: Install kube-bench
kube-bench is a tool that checks whether Kubernetes is deployed according to security best practices as defined in the CIS Kubernetes Benchmark.

Download and install kube-bench
# Create a directory for compliance tools
mkdir -p ~/compliance-tools
cd ~/compliance-tools

# Download the latest kube-bench release
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.6.15/kube-bench_0.6.15_linux_amd64.tar.gz -o kube-bench.tar.gz

# Extract the archive
tar -xzf kube-bench.tar.gz

# Make kube-bench executable and move to PATH
sudo mv kube-bench /usr/local/bin/
sudo chmod +x /usr/local/bin/kube-bench

# Verify installation
kube-bench version
Download CIS benchmark configuration files
# Clone the kube-bench repository for configuration files
git clone https://github.com/aquasecurity/kube-bench.git
cd kube-bench

# Copy configuration files to the appropriate location
sudo mkdir -p /etc/kube-bench
sudo cp -r cfg/ /etc/kube-bench/
Subtask 1.2: Install Polaris
Polaris is a tool that validates Kubernetes resources against best practices and provides recommendations for improvement.

Install Polaris CLI
# Download Polaris CLI
curl -L https://github.com/FairwindsOps/polaris/releases/download/8.5.1/polaris_linux_amd64.tar.gz -o polaris.tar.gz

# Extract and install
tar -xzf polaris.tar.gz
sudo mv polaris /usr/local/bin/
sudo chmod +x /usr/local/bin/polaris

# Verify installation
polaris version
Deploy Polaris to the cluster
# Create Polaris namespace
kubectl create namespace polaris

# Deploy Polaris using the official manifest
kubectl apply -f https://github.com/FairwindsOps/polaris/releases/download/8.5.1/dashboard.yaml

# Wait for Polaris to be ready
kubectl wait --for=condition=available --timeout=300s deployment/polaris-dashboard -n polaris

# Verify Polaris deployment
kubectl get pods -n polaris
Subtask 1.3: Install Additional Compliance Tools
Install kubectl-who-can plugin
# Download kubectl-who-can plugin
curl -L https://github.com/aquasecurity/kubectl-who-can/releases/download/v0.4.0/kubectl-who-can_linux_x86_64.tar.gz -o kubectl-who-can.tar.gz

# Extract and install
tar -xzf kubectl-who-can.tar.gz
sudo mv kubectl-who-can /usr/local/bin/
sudo chmod +x /usr/local/bin/kubectl-who-can
Task 2: Generating Compliance Reports
Subtask 2.1: Run kube-bench Security Assessment
Execute basic kube-bench scan
# Run kube-bench with default settings
sudo kube-bench run --config-dir /etc/kube-bench/cfg --config /etc/kube-bench/cfg/config.yaml

# Save results to a file for analysis
sudo kube-bench run --config-dir /etc/kube-bench/cfg --config /etc/kube-bench/cfg/config.yaml > ~/compliance-tools/kube-bench-report.txt
Generate JSON report for automation
# Generate machine-readable JSON report
sudo kube-bench run --config-dir /etc/kube-bench/cfg --config /etc/kube-bench/cfg/config.yaml --json > ~/compliance-tools/kube-bench-report.json

# View summary of findings
cat ~/compliance-tools/kube-bench-report.json | jq '.Totals'
Create a detailed analysis script
# Create a script to analyze kube-bench results
cat > ~/compliance-tools/analyze-kube-bench.sh << 'EOF'
#!/bin/bash

REPORT_FILE="$1"

if [ ! -f "$REPORT_FILE" ]; then
    echo "Usage: $0 <kube-bench-report.json>"
    exit 1
fi

echo "=== KUBE-BENCH COMPLIANCE ANALYSIS ==="
echo "Report generated on: $(date)"
echo

# Extract totals
TOTAL_PASS=$(cat "$REPORT_FILE" | jq '.Totals.total_pass')
TOTAL_FAIL=$(cat "$REPORT_FILE" | jq '.Totals.total_fail')
TOTAL_WARN=$(cat "$REPORT_FILE" | jq '.Totals.total_warn')
TOTAL_INFO=$(cat "$REPORT_FILE" | jq '.Totals.total_info')

echo "Summary:"
echo "  PASS: $TOTAL_PASS"
echo "  FAIL: $TOTAL_FAIL"
echo "  WARN: $TOTAL_WARN"
echo "  INFO: $TOTAL_INFO"
echo

# Calculate compliance percentage
TOTAL_CHECKS=$((TOTAL_PASS + TOTAL_FAIL))
if [ $TOTAL_CHECKS -gt 0 ]; then
    COMPLIANCE_PERCENTAGE=$(echo "scale=2; $TOTAL_PASS * 100 / $TOTAL_CHECKS" | bc)
    echo "Compliance Percentage: $COMPLIANCE_PERCENTAGE%"
else
    echo "Compliance Percentage: N/A"
fi

echo
echo "=== FAILED CHECKS (Require Immediate Attention) ==="
cat "$REPORT_FILE" | jq -r '.Controls[] | .tests[] | .results[] | select(.result == "FAIL") | "- " + .test_number + ": " + .test_desc'

echo
echo "=== WARNINGS (Should Be Reviewed) ==="
cat "$REPORT_FILE" | jq -r '.Controls[] | .tests[] | .results[] | select(.result == "WARN") | "- " + .test_number + ": " + .test_desc'
EOF

chmod +x ~/compliance-tools/analyze-kube-bench.sh

# Run the analysis
~/compliance-tools/analyze-kube-bench.sh ~/compliance-tools/kube-bench-report.json
Subtask 2.2: Generate Polaris Compliance Report
Access Polaris dashboard
# Port forward to access Polaris dashboard
kubectl port-forward --namespace polaris svc/polaris-dashboard 8080:80 &

# Note: In a real environment, you would access this via browser at http://localhost:8080
# For this lab, we'll use the CLI to generate reports
Generate Polaris CLI report
# Create a sample deployment for testing
kubectl create deployment nginx-test --image=nginx:latest
kubectl expose deployment nginx-test --port=80 --target-port=80

# Wait for deployment
kubectl wait --for=condition=available --timeout=60s deployment/nginx-test

# Run Polaris audit
polaris audit --output-file ~/compliance-tools/polaris-report.json --format json

# Generate human-readable report
polaris audit --output-file ~/compliance-tools/polaris-report.yaml --format yaml

# View summary
cat ~/compliance-tools/polaris-report.json | jq '.Results | length'
Create Polaris analysis script
cat > ~/compliance-tools/analyze-polaris.sh << 'EOF'
#!/bin/bash

REPORT_FILE="$1"

if [ ! -f "$REPORT_FILE" ]; then
    echo "Usage: $0 <polaris-report.json>"
    exit 1
fi

echo "=== POLARIS COMPLIANCE ANALYSIS ==="
echo "Report generated on: $(date)"
echo

# Count issues by severity
DANGER_COUNT=$(cat "$REPORT_FILE" | jq '[.Results[] | .PodResult.Results[] | select(.Severity == "danger")] | length')
WARNING_COUNT=$(cat "$REPORT_FILE" | jq '[.Results[] | .PodResult.Results[] | select(.Severity == "warning")] | length')
SUCCESS_COUNT=$(cat "$REPORT_FILE" | jq '[.Results[] | .PodResult.Results[] | select(.Success == true)] | length')

echo "Issue Summary:"
echo "  Danger (Critical): $DANGER_COUNT"
echo "  Warning: $WARNING_COUNT"
echo "  Success: $SUCCESS_COUNT"
echo

echo "=== CRITICAL ISSUES (Danger Level) ==="
cat "$REPORT_FILE" | jq -r '.Results[] | .PodResult.Results[] | select(.Severity == "danger") | "- " + .Message + " (Category: " + .Category + ")"'

echo
echo "=== WARNINGS ==="
cat "$REPORT_FILE" | jq -r '.Results[] | .PodResult.Results[] | select(.Severity == "warning") | "- " + .Message + " (Category: " + .Category + ")"'

echo
echo "=== RECOMMENDATIONS ==="
echo "1. Address all critical (danger) issues immediately"
echo "2. Review and fix warning-level issues"
echo "3. Implement security policies to prevent future violations"
echo "4. Set up automated compliance monitoring"
EOF

chmod +x ~/compliance-tools/analyze-polaris.sh

# Run Polaris analysis
~/compliance-tools/analyze-polaris.sh ~/compliance-tools/polaris-report.json
Subtask 2.3: Create Comprehensive Compliance Dashboard
Create a unified compliance report script
cat > ~/compliance-tools/compliance-dashboard.sh << 'EOF'
#!/bin/bash

# Compliance Dashboard Generator
TIMESTAMP=$(date '+%Y-%m-%d_%H-%M-%S')
REPORT_DIR="~/compliance-tools/reports/$TIMESTAMP"

mkdir -p "$REPORT_DIR"

echo "=== KUBERNETES CLUSTER COMPLIANCE DASHBOARD ==="
echo "Generated on: $(date)"
echo "Cluster: $(kubectl config current-context)"
echo

# Generate fresh reports
echo "Generating kube-bench report..."
sudo kube-bench run --config-dir /etc/kube-bench/cfg --config /etc/kube-bench/cfg/config.yaml --json > "$REPORT_DIR/kube-bench.json" 2>/dev/null

echo "Generating Polaris report..."
polaris audit --output-file "$REPORT_DIR/polaris.json" --format json 2>/dev/null

# Analyze kube-bench results
if [ -f "$REPORT_DIR/kube-bench.json" ]; then
    echo
    echo "=== KUBE-BENCH RESULTS ==="
    KUBE_PASS=$(cat "$REPORT_DIR/kube-bench.json" | jq '.Totals.total_pass // 0')
    KUBE_FAIL=$(cat "$REPORT_DIR/kube-bench.json" | jq '.Totals.total_fail // 0')
    KUBE_WARN=$(cat "$REPORT_DIR/kube-bench.json" | jq '.Totals.total_warn // 0')
    
    echo "  Passed: $KUBE_PASS"
    echo "  Failed: $KUBE_FAIL"
    echo "  Warnings: $KUBE_WARN"
    
    if [ "$KUBE_FAIL" -gt 0 ]; then
        echo "  Status: ❌ COMPLIANCE ISSUES DETECTED"
    else
        echo "  Status: ✅ CIS BENCHMARK COMPLIANT"
    fi
fi

# Analyze Polaris results
if [ -f "$REPORT_DIR/polaris.json" ]; then
    echo
    echo "=== POLARIS RESULTS ==="
    POLARIS_DANGER=$(cat "$REPORT_DIR/polaris.json" | jq '[.Results[]? | .PodResult.Results[]? | select(.Severity == "danger")] | length')
    POLARIS_WARNING=$(cat "$REPORT_DIR/polaris.json" | jq '[.Results[]? | .PodResult.Results[]? | select(.Severity == "warning")] | length')
    
    echo "  Critical Issues: $POLARIS_DANGER"
    echo "  Warnings: $POLARIS_WARNING"
    
    if [ "$POLARIS_DANGER" -gt 0 ]; then
        echo "  Status: ❌ CRITICAL SECURITY ISSUES"
    elif [ "$POLARIS_WARNING" -gt 0 ]; then
        echo "  Status: ⚠️  WARNINGS PRESENT"
    else
        echo "  Status: ✅ BEST PRACTICES FOLLOWED"
    fi
fi

# Overall compliance status
echo
echo "=== OVERALL COMPLIANCE STATUS ==="
if [ "$KUBE_FAIL" -gt 0 ] || [ "$POLARIS_DANGER" -gt 0 ]; then
    echo "🔴 IMMEDIATE ACTION REQUIRED"
    echo "   Critical compliance issues detected that require immediate remediation."
elif [ "$KUBE_WARN" -gt 0 ] || [ "$POLARIS_WARNING" -gt 0 ]; then
    echo "🟡 REVIEW RECOMMENDED"
    echo "   Some issues detected that should be reviewed and addressed."
else
    echo "🟢 COMPLIANT"
    echo "   Cluster meets security and compliance standards."
fi

echo
echo "Detailed reports saved to: $REPORT_DIR"
EOF

chmod +x ~/compliance-tools/compliance-dashboard.sh

# Run the compliance dashboard
~/compliance-tools/compliance-dashboard.sh
Task 3: Automating Remediation of Detected Issues
Subtask 3.1: Create Automated Remediation Scripts
Create a script to fix common kube-bench issues
cat > ~/compliance-tools/remediate-kube-bench.sh << 'EOF'
#!/bin/bash

echo "=== KUBE-BENCH AUTOMATED REMEDIATION ==="
echo "This script addresses common CIS Kubernetes Benchmark failures"
echo

# Function to backup configuration files
backup_config() {
    local file="$1"
    if [ -f "$file" ]; then
        sudo cp "$file" "$file.backup.$(date +%Y%m%d_%H%M%S)"
        echo "Backed up $file"
    fi
}

# Remediation for kubelet configuration
remediate_kubelet() {
    echo "Remediating kubelet configuration..."
    
    # Common kubelet security settings
    KUBELET_CONFIG="/var/lib/kubelet/config.yaml"
    
    if [ -f "$KUBELET_CONFIG" ]; then
        backup_config "$KUBELET_CONFIG"
        
        # Create a secure kubelet configuration
        sudo tee "$KUBELET_CONFIG" > /dev/null << 'KUBELET_EOF'
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
serverTLSBootstrap: true
protectKernelDefaults: true
makeIPTablesUtilChains: true
eventRecordQPS: 0
streamingConnectionIdleTimeout: 4h0m0s
KUBELET_EOF
        
        echo "✅ Updated kubelet configuration for security"
    fi
}

# Remediation for API server
remediate_apiserver() {
    echo "Remediating API server configuration..."
    
    # Note: In a real cluster, you would modify the API server manifest
    # For this lab, we'll create a recommended configuration template
    
    cat > ~/compliance-tools/secure-apiserver-config.yaml << 'API_EOF'
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-log-maxage=30
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=100
    - --audit-log-path=/var/log/apiserver/audit.log
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver:v1.28.0
    name: kube-apiserver
API_EOF
    
    echo "✅ Created secure API server configuration template"
}

# Create Network Policies for security
create_network_policies() {
    echo "Creating default network policies..."
    
    # Default deny all ingress traffic
    kubectl apply -f - << 'NP_EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
NP_EOF
    
    echo "✅ Created default network policies"
}

# Create Pod Security Policy
create_pod_security_policy() {
    echo "Creating Pod Security Policy..."
    
    kubectl apply -f - << 'PSP_EOF'
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
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
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: restricted-psp-user
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames:
  - restricted
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: restricted-psp-all-serviceaccounts
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: restricted-psp-user
  apiGroup: rbac.authorization.k8s.io
PSP_EOF
    
    echo "✅ Created Pod Security Policy"
}

# Main remediation execution
echo "Starting automated remediation..."
echo

remediate_kubelet
remediate_apiserver
create_network_policies
create_pod_security_policy

echo
echo "=== REMEDIATION COMPLETE ==="
echo "⚠️  Note: Some changes require cluster restart to take effect"
echo "📋 Review the generated configuration files before applying in production"
echo "🔄 Run kube-bench again to verify improvements"
EOF

chmod +x ~/compliance-tools/remediate-kube-bench.sh
Create Polaris remediation script
cat > ~/compliance-tools/remediate-polaris.sh << 'EOF'
#!/bin/bash

echo "=== POLARIS AUTOMATED REMEDIATION ==="
echo "This script creates secure deployment templates and policies"
echo

# Create secure deployment template
create_secure_deployment_template() {
    echo "Creating secure deployment template..."
    
    cat > ~/compliance-tools/secure-deployment-template.yaml << 'DEPLOY_EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
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
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: app
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
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 250m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
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
DEPLOY_EOF
    
    echo "✅ Created secure deployment template"
}

# Fix existing deployments
fix_existing_deployments() {
    echo "Fixing existing deployments..."
    
    # Get all deployments
    DEPLOYMENTS=$(kubectl get deployments -o name)
    
    for deployment in $DEPLOYMENTS; do
        DEPLOY_NAME=$(echo $deployment | cut -d'/' -f2)
        echo "Fixing deployment: $DEPLOY_NAME"
        
        # Add security context and resource limits
        kubectl patch deployment "$DEPLOY_NAME" -p '{
            "spec": {
                "template": {
                    "spec": {
                        "securityContext": {
                            "runAsNonRoot": true,
                            "runAsUser": 1000
                        },
                        "containers": [{
                            "name": "'$DEPLOY_NAME'",
                            "securityContext": {
                                "allowPrivilegeEscalation": false,
                                "readOnlyRootFilesystem": false,
                                "runAsNonRoot": true,
                                "capabilities": {
                                    "drop": ["ALL"]
                                }
                            },
                            "resources": {
                                "limits": {
                                    "cpu": "500m",
                                    "memory": "512Mi"
                                },
                                "requests": {
                                    "cpu": "250m",
                                    "memory": "256Mi"
                                }
                            }
                        }]
                    }
                }
            }
        }' 2>/dev/null || echo "  ⚠️  Could not patch $DEPLOY_NAME (may require manual intervention)"
    done
}

# Create ValidatingAdmissionWebhook for Polaris
create_admission_webhook() {
    echo "Setting up Polaris ValidatingAdmissionWebhook..."
    
    kubectl apply -f - << 'WEBHOOK_EOF'
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: polaris-webhook
webhooks:
- name: polaris.fairwinds.com
  clientConfig:
    service:
      name: polaris-webhook
      namespace: polaris
      path: "/validate"
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments", "daemonsets", "statefulsets"]
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail
WEBHOOK_EOF
    
    echo "✅ Created Polaris admission webhook configuration"
}

# Create OPA Gatekeeper policies
create_gatekeeper_policies() {
    echo "Creating OPA Gatekeeper policies for compliance..."
    
    # Install Gatekeeper
    kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
    
    # Wait for Gatekeeper to be ready
    kubectl wait --for=condition=available --timeout=300s deployment/gatekeeper-controller-manager -n gatekeeper-system
    
    # Create constraint template for required security context
    kubectl apply -f - << 'CONSTRAINT_EOF'
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
      validation:
        openAPIV3Schema:
          type: object
          properties:
            runAsNonRoot:
              type: boolean
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredsecuritycontext
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.template.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := "Container must run as non-root user"
        }
        
        violation[{"msg": msg}] {
          not input.review.object.spec.template.spec.securityContext.runAsNonRoot
          msg := "Pod must run as non-root user"
        }
---
apiVersion: config.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: must-run-as-non-root
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    runAsNonRoot: true
CONSTRAINT_EOF
    
    echo "✅ Created Gatekeeper security policies"
}

# Main execution
echo "Starting Polaris remediation..."
echo

create_secure_deployment_template
fix_existing_deployments
create_admission_webhook
create_gatekeeper_policies

echo
echo "=== POLARIS REMEDIATION COMPLETE ==="
echo "📋 Secure deployment template created at ~/compliance-tools/secure-deployment-template.yaml"
echo "🔒 Admission webhook configured to prevent non-compliant deployments"
echo "⚖️  Gatekeeper policies installed for ongoing compliance enforcement"
echo "🔄 Run Polaris audit again to verify improvements"
EOF

chmod +x ~/compliance-tools/remediate-polaris.sh
Subtask 3.2: Create Policy-Based Automation
Set up automated compliance monitoring with cron jobs
# Create automated compliance checking script
cat > ~/compliance-tools/automated-compliance-check.sh << 'EOF'
#!/bin/bash

# Automated Compliance Monitoring Script
TIMESTAMP=$(date '+%Y-%m-%d_%H-%M-%S')
LOG_DIR="/var/log/compliance"
REPORT_DIR="$HOME/compliance-tools/reports"

# Create directories
sudo mkdir -p "$LOG_DIR"
mkdir -p "$REPORT_DIR"

# Function to send alerts (placeholder for real alerting system)
send_alert() {
    local severity="$1"
    local message="$2"
    
    echo "[$severity] $message" | sudo tee -a "$LOG_DIR/compliance-alerts.log"
    
    # In production, you would integrate with:
    # - Slack/Teams notifications
    # - Email alerts
    # - PagerDuty/OpsGenie
    # - Monitoring systems (Prometheus AlertManager)
}

# Run compliance checks
echo "Running automated compliance check at $(date)"

# Run kube-bench
echo "Executing kube-bench scan..."
sudo kube-bench run --config-dir /etc/kube-bench/cfg --config /etc/kube-bench/cfg/config.yaml --json > "$REPORT_DIR/kube-bench-$TIMESTAMP.json" 2>/dev/null

# Analyze kube-bench results
if [ -f "$REPORT_DIR/kube-bench-$TIMESTAMP.json" ]; then
    FAILED_CHECKS=$(cat "$REPORT_DIR/kube-bench-$TIMESTAMP.json" | jq '.Totals.total_fail // 0')
    
    if [ "$FAILED_CHECKS" -gt 0 ]; then
        send_alert "CRITICAL" "kube-bench detected $FAILED_CHECKS failed security checks"
        
        # Auto-remediate if configured
        if [ "$AUTO_REMEDIATE" = "true" ]; then
            echo "Auto-remediation enabled, running fixes..."
            ~/compliance-tools/remediate-kube-bench.sh >> "$LOG_DIR/auto-remediation.log" 2>&1
        fi
    else
        send_alert "INFO" "kube-bench scan completed successfully - no critical issues"
    fi
fi

# Run Polaris audit
echo "Executing Polaris audit..."
polaris audit --output-file "$REPORT_DIR/polaris-$TIMESTAMP.json" --format json 2>/dev/null

# Analyze Polaris results
if [ -f "$REPORT_DIR/polaris-$TIMESTAMP.json" ]; then
    CRITICAL_ISSUES=$(cat "$REPORT_DIR/polaris-$TIMESTAMP.json" | jq '[.Results[]? | .PodResult.Results[]? | select(.Severity == "danger")] | length')
    
    if [ "$CRITICAL_ISSUES" -gt 0 ]; then
        send_alert "HIGH
