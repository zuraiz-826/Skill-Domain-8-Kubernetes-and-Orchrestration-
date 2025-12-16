Lab 8: Compliance Automation in Kubernetes
Objectives
By the end of this lab, you will be able to:

• Install and configure Open Policy Agent (OPA) Gatekeeper in a Kubernetes cluster • Create and implement security policies to restrict container privileges • Understand the difference between OPA Gatekeeper and Kyverno policy engines • Automate compliance checks using policy enforcement • Test policy enforcement during application deployments • Troubleshoot policy violations and understand compliance reporting • Implement best practices for Kubernetes security governance

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, deployments, services) • Familiarity with YAML configuration files • Basic knowledge of Linux command line operations • Understanding of container security concepts • Knowledge of kubectl commands and Kubernetes API objects

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-configured • A running Kubernetes cluster (single-node for lab purposes) • All necessary tools and dependencies pre-installed • Internet access for downloading policy engine components

Task 1: Install Open Policy Agent (OPA) Gatekeeper
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure your Kubernetes cluster is running properly.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
Subtask 1.2: Install OPA Gatekeeper
OPA Gatekeeper is the policy engine that enforces Open Policy Agent policies in Kubernetes.

# Apply the Gatekeeper manifest
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml

# Wait for Gatekeeper pods to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n gatekeeper-system --timeout=300s

# Verify Gatekeeper installation
kubectl get pods -n gatekeeper-system
Expected output should show three pods running:

gatekeeper-audit (1 pod)
gatekeeper-controller-manager (3 pods)
Subtask 1.3: Verify Gatekeeper Components
# Check Gatekeeper CRDs (Custom Resource Definitions)
kubectl get crd | grep gatekeeper

# Verify webhook configuration
kubectl get validatingadmissionwebhooks | grep gatekeeper

# Check Gatekeeper logs
kubectl logs -n gatekeeper-system deployment/gatekeeper-controller-manager
Task 2: Create Security Policies to Restrict Container Privileges
Subtask 2.1: Create a Constraint Template for Root User Restriction
A Constraint Template defines the logic for a policy, while a Constraint applies that template with specific parameters.

Create a file called no-root-user-template.yaml:

cat > no-root-user-template.yaml << 'EOF'
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequirenonroot
  annotations:
    description: "Requires containers to run as non-root user"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireNonRoot
      validation:
        type: object
        properties:
          exemptImages:
            description: "List of exempt container images"
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirenonroot

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          not is_exempt(container.image)
          msg := sprintf("Container <%v> must run as non-root user", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.runAsUser == 0
          not is_exempt(container.image)
          msg := sprintf("Container <%v> cannot run as root (UID 0)", [container.name])
        }

        is_exempt(image) {
          exempt_image := input.parameters.exemptImages[_]
          startswith(image, exempt_image)
        }
EOF
Apply the constraint template:

kubectl apply -f no-root-user-template.yaml

# Verify the template was created
kubectl get constrainttemplates
Subtask 2.2: Create a Constraint to Enforce the Policy
Create a file called no-root-user-constraint.yaml:

cat > no-root-user-constraint.yaml << 'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireNonRoot
metadata:
  name: must-run-as-non-root
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "gatekeeper-system", "kube-public"]
  parameters:
    exemptImages:
      - "gcr.io/google-containers/"
      - "k8s.gcr.io/"
EOF
Apply the constraint:

kubectl apply -f no-root-user-constraint.yaml

# Verify the constraint was created
kubectl get k8srequirenonroot

# Check constraint status
kubectl describe k8srequirenonroot must-run-as-non-root
Subtask 2.3: Create Additional Security Policies
Let's create another policy to restrict privileged containers.

Create no-privileged-template.yaml:

cat > no-privileged-template.yaml << 'EOF'
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequirenonprivileged
  annotations:
    description: "Prohibits privileged containers"
spec:
  crd:
    spec:
      names:
        kind: K8sRequireNonPrivileged
      validation:
        type: object
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirenonprivileged

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged
          msg := sprintf("Container <%v> cannot run in privileged mode", [container.name])
        }
EOF
Create no-privileged-constraint.yaml:

cat > no-privileged-constraint.yaml << 'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireNonPrivileged
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "gatekeeper-system"]
EOF
Apply both files:

kubectl apply -f no-privileged-template.yaml
kubectl apply -f no-privileged-constraint.yaml

# Verify both constraints are active
kubectl get constraints
Task 3: Automate Compliance Checks and Test Enforcement
Subtask 3.1: Create Test Applications
Let's create applications that violate our policies to test enforcement.

Create a non-compliant application that runs as root:

cat > bad-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: bad-app
  labels:
    app: bad-app
spec:
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      runAsUser: 0  # This violates our policy
      privileged: true  # This also violates our policy
EOF
Try to deploy the non-compliant application:

kubectl apply -f bad-app.yaml
You should see an error message indicating policy violations.

Subtask 3.2: Create a Compliant Application
Now create a compliant application:

cat > good-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: good-app
  labels:
    app: good-app
spec:
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      privileged: false
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
Deploy the compliant application:

kubectl apply -f good-app.yaml

# Verify the pod is running
kubectl get pods
kubectl describe pod good-app
Subtask 3.3: Test Policy Enforcement with Deployments
Create a deployment that initially violates policies:

cat > test-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  labels:
    app: test-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: busybox:latest
        command: ["sleep", "3600"]
        securityContext:
          runAsUser: 0  # This will be blocked
EOF
Try to deploy:

kubectl apply -f test-deployment.yaml
Now fix the deployment to be compliant:

cat > test-deployment-fixed.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app-fixed
  labels:
    app: test-app-fixed
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app-fixed
  template:
    metadata:
      labels:
        app: test-app-fixed
    spec:
      containers:
      - name: app
        image: busybox:latest
        command: ["sleep", "3600"]
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          privileged: false
          allowPrivilegeEscalation: false
EOF
Deploy the fixed version:

kubectl apply -f test-deployment-fixed.yaml

# Verify deployment success
kubectl get deployments
kubectl get pods -l app=test-app-fixed
Subtask 3.4: Monitor Compliance with Gatekeeper Audit
Gatekeeper continuously audits existing resources for policy violations.

# Check audit results
kubectl get k8srequirenonroot must-run-as-non-root -o yaml

# View violations in the constraint status
kubectl describe k8srequirenonroot must-run-as-non-root

# Check privileged container violations
kubectl describe k8srequirenonprivileged no-privileged-containers
Subtask 3.5: Create a Compliance Dashboard Script
Create a script to check overall compliance status:

cat > compliance-check.sh << 'EOF'
#!/bin/bash

echo "=== Kubernetes Compliance Status ==="
echo ""

echo "1. Gatekeeper System Status:"
kubectl get pods -n gatekeeper-system --no-headers | awk '{print $1 ": " $3}'
echo ""

echo "2. Active Constraint Templates:"
kubectl get constrainttemplates --no-headers | awk '{print "- " $1}'
echo ""

echo "3. Active Constraints:"
kubectl get constraints --no-headers | awk '{print "- " $1 " (" $2 ")"}'
echo ""

echo "4. Policy Violations Summary:"
for constraint in $(kubectl get constraints -o name); do
    violations=$(kubectl get $constraint -o jsonpath='{.status.totalViolations}' 2>/dev/null)
    name=$(echo $constraint | cut -d'/' -f2)
    echo "- $name: ${violations:-0} violations"
done
echo ""

echo "5. Non-compliant Pods (if any):"
kubectl get pods --all-namespaces -o json | jq -r '.items[] | select(.spec.containers[]?.securityContext.runAsUser == 0 or .spec.containers[]?.securityContext.privileged == true) | "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null || echo "No non-compliant pods found or jq not available"
EOF

chmod +x compliance-check.sh
./compliance-check.sh
Task 4: Advanced Policy Testing and Troubleshooting
Subtask 4.1: Test Policy Exemptions
Create a pod in an exempted namespace to verify exclusions work:

# Create a test namespace
kubectl create namespace test-exempt

# Add the namespace to our constraint exemptions
cat > updated-constraint.yaml << 'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireNonRoot
metadata:
  name: must-run-as-non-root
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "gatekeeper-system", "kube-public", "test-exempt"]
  parameters:
    exemptImages:
      - "gcr.io/google-containers/"
      - "k8s.gcr.io/"
EOF

kubectl apply -f updated-constraint.yaml

# Test deploying a root container in the exempt namespace
kubectl run test-root --image=nginx --restart=Never -n test-exempt --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","securityContext":{"runAsUser":0}}]}}'

# Verify it was allowed
kubectl get pods -n test-exempt
Subtask 4.2: Policy Dry-Run Testing
Enable dry-run mode to test policies without enforcement:

cat > dry-run-constraint.yaml << 'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireNonRoot
metadata:
  name: must-run-as-non-root-dryrun
spec:
  enforcementAction: dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "gatekeeper-system", "kube-public"]
  parameters:
    exemptImages:
      - "gcr.io/google-containers/"
EOF

kubectl apply -f dry-run-constraint.yaml

# Test with a violating pod - it should be created but logged as violation
kubectl run test-dryrun --image=nginx --restart=Never --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","securityContext":{"runAsUser":0}}]}}'

# Check if pod was created (it should be)
kubectl get pod test-dryrun

# Check violations in constraint status
kubectl describe k8srequirenonroot must-run-as-non-root-dryrun
Subtask 4.3: Troubleshooting Common Issues
Create a troubleshooting guide:

cat > troubleshoot.sh << 'EOF'
#!/bin/bash

echo "=== Gatekeeper Troubleshooting Guide ==="
echo ""

echo "1. Check Gatekeeper Controller Logs:"
echo "kubectl logs -n gatekeeper-system deployment/gatekeeper-controller-manager"
echo ""

echo "2. Check Webhook Configuration:"
echo "kubectl get validatingadmissionwebhooks"
echo ""

echo "3. Verify Constraint Template Syntax:"
echo "kubectl describe constrainttemplate <template-name>"
echo ""

echo "4. Check Constraint Status:"
echo "kubectl describe <constraint-kind> <constraint-name>"
echo ""

echo "5. Test Policy Logic:"
echo "Use 'kubectl apply --dry-run=server' to test without creating resources"
echo ""

echo "6. Common Issues and Solutions:"
echo "- Policy not enforcing: Check constraint enforcementAction"
echo "- Template errors: Validate Rego syntax"
echo "- Webhook failures: Check network policies and DNS"
echo "- Performance issues: Review constraint scope and complexity"
EOF

chmod +x troubleshoot.sh
Task 5: Comparison with Kyverno (Optional Advanced Section)
Subtask 5.1: Understanding Kyverno vs OPA Gatekeeper
Create a comparison document:

cat > policy-engines-comparison.md << 'EOF'
# Policy Engine Comparison: OPA Gatekeeper vs Kyverno

## OPA Gatekeeper
**Pros:**
- Uses Rego language (powerful and flexible)
- Part of CNCF graduated project
- Strong community and enterprise support
- Excellent for complex policy logic

**Cons:**
- Steeper learning curve (Rego language)
- More complex setup for simple policies
- Requires understanding of OPA concepts

## Kyverno
**Pros:**
- Uses YAML (familiar to Kubernetes users)
- Easier to learn and implement
- Built-in policy library
- Good for simple to moderate complexity policies

**Cons:**
- Less flexible for complex logic
- Smaller community compared to OPA
- Limited advanced features

## When to Choose Which:
- **OPA Gatekeeper**: Complex compliance requirements, existing OPA knowledge, enterprise environments
- **Kyverno**: Simple policies, YAML preference, quick implementation needs
EOF
Subtask 5.2: Quick Kyverno Installation (Demo Only)
Note: This is for demonstration purposes. In production, choose one policy engine.

# Install Kyverno (for comparison)
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml

# Wait for Kyverno to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kyverno -n kyverno --timeout=300s

# Create a simple Kyverno policy
cat > kyverno-policy.yaml << 'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root-user
spec:
  validationFailureAction: enforce
  background: true
  rules:
  - name: check-non-root
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Containers must run as non-root user"
      pattern:
        spec:
          containers:
          - securityContext:
              runAsNonRoot: true
EOF

kubectl apply -f kyverno-policy.yaml
Cleanup and Resource Management
Subtask 6.1: Clean Up Test Resources
# Remove test pods and deployments
kubectl delete pod good-app --ignore-not-found
kubectl delete pod test-dryrun --ignore-not-found
kubectl delete pod test-root -n test-exempt --ignore-not-found
kubectl delete deployment test-app-fixed --ignore-not-found

# Remove test namespace
kubectl delete namespace test-exempt --ignore-not-found

# If you installed Kyverno for demo, remove it
kubectl delete -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml --ignore-not-found
Subtask 6.2: Keep Gatekeeper for Future Use
# List all Gatekeeper resources (keep these for future labs)
echo "Gatekeeper resources to keep:"
kubectl get constrainttemplates
kubectl get constraints
kubectl get pods -n gatekeeper-system
Conclusion
Congratulations! You have successfully completed Lab 8: Compliance Automation in Kubernetes. Here's what you accomplished:

Key Achievements
Policy Engine Mastery: You installed and configured OPA Gatekeeper, learning how to implement policy-as-code in Kubernetes environments.

Security Policy Implementation: You created comprehensive security policies that prevent containers from running as root users and prohibit privileged containers, significantly improving your cluster's security posture.

Automated Compliance: You implemented automated compliance checking that continuously monitors your cluster and prevents non-compliant workloads from being deployed.

Testing and Validation: You learned how to test policy enforcement, handle exemptions, and troubleshoot common policy-related issues.

Real-World Application: These skills directly apply to enterprise Kubernetes environments where security compliance is critical for regulatory requirements like SOC 2, PCI DSS, and HIPAA.

Why This Matters
In production Kubernetes environments, manual security reviews are insufficient and error-prone. Automated compliance ensures that:

Security policies are consistently enforced across all deployments
Compliance violations are caught early in the deployment pipeline
Audit requirements are automatically satisfied through continuous monitoring
Development teams receive immediate feedback on security issues
Next Steps
To further enhance your Kubernetes security skills:

Explore advanced Rego policies for more complex compliance scenarios
Integrate policy enforcement with CI/CD pipelines
Implement policy testing frameworks for validation before deployment
Study compliance frameworks like CIS Kubernetes Benchmark
Practice with real-world scenarios involving multiple policy engines
This lab has provided you with essential skills for the Kubernetes and Cloud Native Security Associate (KCSA) certification and prepared you for implementing enterprise-grade security governance in Kubernetes clusters.

The automated compliance capabilities you've learned are fundamental to modern DevSecOps practices and will serve you well in securing cloud-native applications at scale.
