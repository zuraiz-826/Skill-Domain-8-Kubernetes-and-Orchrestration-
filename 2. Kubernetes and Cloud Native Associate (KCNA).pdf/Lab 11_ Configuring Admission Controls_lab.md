Lab 11: Configuring Admission Controls
Objectives
By the end of this lab, you will be able to:

• Understand the concept of admission controllers in Kubernetes • Deploy and configure Open Policy Agent (OPA) Gatekeeper for policy enforcement • Create security policies to prevent containers from running as root • Test policy enforcement by attempting to deploy non-compliant workloads • Troubleshoot common admission control issues • Implement best practices for container security using admission controls

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, deployments, services) • Familiarity with YAML configuration files • Knowledge of Linux command line operations • Understanding of container security fundamentals • Experience with kubectl commands

Lab Environment
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl configured • Kubernetes cluster (single-node for lab purposes) • Internet connectivity for downloading required components • Pre-configured user with sudo privileges

Task 1: Deploy Open Policy Agent (OPA) Gatekeeper
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure our Kubernetes cluster is running properly.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
Subtask 1.2: Install OPA Gatekeeper
OPA Gatekeeper is a validating admission webhook that enforces Custom Resource Definition (CRD)-based policies.

# Apply the Gatekeeper manifest
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml

# Wait for Gatekeeper pods to be ready
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n gatekeeper-system --timeout=300s

# Verify Gatekeeper installation
kubectl get pods -n gatekeeper-system
Expected output should show pods in Running status:

NAME                                             READY   STATUS    RESTARTS   AGE
gatekeeper-audit-7d9c4c5b5d-xyz12               1/1     Running   0          2m
gatekeeper-controller-manager-6b8c9d4f7b-abc34  1/1     Running   0          2m
gatekeeper-controller-manager-6b8c9d4f7b-def56  1/1     Running   0          2m
Subtask 1.3: Verify Gatekeeper Components
# Check Gatekeeper CRDs
kubectl get crd | grep gatekeeper

# Verify admission webhook configuration
kubectl get validatingadmissionwebhooks | grep gatekeeper
Task 2: Create Policy to Block Root Containers
Subtask 2.1: Create Constraint Template
A ConstraintTemplate defines the logic for policy enforcement. We'll create one to block containers running as root.

# Create the constraint template file
cat << 'EOF' > block-root-user-template.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
  annotations:
    description: "Requires containers to run with a non-root user"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
      validation:
        type: object
        properties:
          runAsNonRoot:
            type: boolean
            description: "Requires containers to run as non-root user"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredsecuritycontext

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container <%v> must set runAsNonRoot to true", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.runAsUser == 0
          msg := sprintf("Container <%v> cannot run as root user (UID 0)", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not has_security_context(container)
          msg := sprintf("Container <%v> must define securityContext", [container.name])
        }

        has_security_context(container) {
          container.securityContext
        }
EOF

# Apply the constraint template
kubectl apply -f block-root-user-template.yaml

# Verify the template was created
kubectl get constrainttemplates
Subtask 2.2: Create the Constraint
Now we'll create a Constraint that uses our template to enforce the policy.

# Create the constraint file
cat << 'EOF' > block-root-user-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: must-run-as-nonroot
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "gatekeeper-system", "kube-public", "kube-node-lease"]
  parameters:
    runAsNonRoot: true
EOF

# Apply the constraint
kubectl apply -f block-root-user-constraint.yaml

# Verify the constraint was created
kubectl get k8srequiredsecuritycontext
Subtask 2.3: Check Constraint Status
# Check constraint details and status
kubectl describe k8srequiredsecuritycontext must-run-as-nonroot

# Wait for constraint to be ready
kubectl wait --for=condition=ready k8srequiredsecuritycontext must-run-as-nonroot --timeout=60s
Task 3: Test Policy Enforcement During Deployment
Subtask 3.1: Test with Non-Compliant Deployment
Let's try to deploy a pod that runs as root - this should be blocked by our policy.

# Create a non-compliant pod manifest
cat << 'EOF' > bad-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  labels:
    app: test
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    securityContext:
      runAsUser: 0  # This runs as root - should be blocked
EOF

# Try to apply the non-compliant pod
kubectl apply -f bad-pod.yaml
Expected Result: The deployment should fail with an error message similar to:

Error from server: error validating data: ValidationError(Pod.spec.containers[0].securityContext): 
Container <nginx> cannot run as root user (UID 0)
Subtask 3.2: Test with Another Non-Compliant Pod
# Create another non-compliant pod without securityContext
cat << 'EOF' > bad-pod-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod-2
  labels:
    app: test
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    # No securityContext defined - should be blocked
EOF

# Try to apply this pod
kubectl apply -f bad-pod-2.yaml
Expected Result: This should also fail with an error about missing securityContext.

Subtask 3.3: Test with Compliant Deployment
Now let's create a pod that follows our security policy.

# Create a compliant pod manifest
cat << 'EOF' > good-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  labels:
    app: test
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000  # Non-root user
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
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

# Apply the compliant pod
kubectl apply -f good-pod.yaml

# Verify the pod was created successfully
kubectl get pods good-pod

# Check pod details
kubectl describe pod good-pod
Expected Result: The pod should be created successfully and show Running status.

Subtask 3.4: Test with Deployment Resource
Let's test our policy with a Deployment resource.

# Create a compliant deployment
cat << 'EOF' > secure-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-nginx
  labels:
    app: secure-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-nginx
  template:
    metadata:
      labels:
        app: secure-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
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

# Apply the deployment
kubectl apply -f secure-deployment.yaml

# Check deployment status
kubectl get deployment secure-nginx

# Check pods created by deployment
kubectl get pods -l app=secure-nginx
Task 4: Monitor and Validate Policy Enforcement
Subtask 4.1: Check Gatekeeper Audit Results
Gatekeeper continuously audits existing resources for policy violations.

# Check for any violations in existing resources
kubectl get k8srequiredsecuritycontext must-run-as-nonroot -o yaml

# Look for violations in the status section
kubectl get k8srequiredsecuritycontext must-run-as-nonroot -o jsonpath='{.status.violations}'
Subtask 4.2: View Gatekeeper Logs
# Check controller logs for policy enforcement
kubectl logs -n gatekeeper-system -l control-plane=controller-manager --tail=50

# Check audit logs
kubectl logs -n gatekeeper-system -l control-plane=audit-controller --tail=50
Subtask 4.3: Test Policy with Different Namespaces
# Create a test namespace
kubectl create namespace policy-test

# Try to deploy non-compliant pod in the new namespace
cat << 'EOF' > test-namespace-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: policy-test
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    securityContext:
      runAsUser: 0  # Should be blocked
EOF

# Apply to test namespace
kubectl apply -f test-namespace-pod.yaml
This should also be blocked by our policy since we didn't exclude the policy-test namespace.

Task 5: Advanced Policy Configuration
Subtask 5.1: Create Exception for Specific Applications
Sometimes you need to allow exceptions for specific applications. Let's modify our constraint.

# Update constraint to allow exceptions
cat << 'EOF' > block-root-user-constraint-updated.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: must-run-as-nonroot
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces: ["kube-system", "gatekeeper-system", "kube-public", "kube-node-lease"]
    labelSelector:
      matchExpressions:
      - key: "security-policy"
        operator: NotIn
        values: ["exempt"]
  parameters:
    runAsNonRoot: true
EOF

# Apply the updated constraint
kubectl apply -f block-root-user-constraint-updated.yaml
Subtask 5.2: Test Exception
# Create a pod with exemption label
cat << 'EOF' > exempt-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: exempt-pod
  labels:
    app: test
    security-policy: exempt
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    securityContext:
      runAsUser: 0  # This should now be allowed due to exemption
EOF

# Apply the exempt pod
kubectl apply -f exempt-pod.yaml

# Verify it was created
kubectl get pod exempt-pod
Troubleshooting Common Issues
Issue 1: Gatekeeper Pods Not Starting
# Check pod status and events
kubectl get pods -n gatekeeper-system
kubectl describe pods -n gatekeeper-system

# Check for resource constraints
kubectl top nodes
kubectl describe nodes
Issue 2: Policies Not Enforcing
# Verify constraint template is valid
kubectl get constrainttemplates k8srequiredsecuritycontext -o yaml

# Check constraint status
kubectl describe k8srequiredsecuritycontext must-run-as-nonroot

# Verify webhook is active
kubectl get validatingadmissionwebhooks gatekeeper-validating-webhook-configuration
Issue 3: False Positives
# Check exact error messages
kubectl apply -f bad-pod.yaml --dry-run=server

# Review constraint logic
kubectl get constrainttemplates k8srequiredsecuritycontext -o jsonpath='{.spec.targets[0].rego}'
Cleanup
Remove Test Resources
# Remove test pods and deployments
kubectl delete pod good-pod --ignore-not-found
kubectl delete pod exempt-pod --ignore-not-found
kubectl delete deployment secure-nginx --ignore-not-found
kubectl delete namespace policy-test --ignore-not-found

# Remove constraint and template
kubectl delete k8srequiredsecuritycontext must-run-as-nonroot
kubectl delete constrainttemplate k8srequiredsecuritycontext

# Remove Gatekeeper (optional)
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
Clean Up Files
# Remove created YAML files
rm -f block-root-user-template.yaml
rm -f block-root-user-constraint.yaml
rm -f block-root-user-constraint-updated.yaml
rm -f bad-pod.yaml
rm -f bad-pod-2.yaml
rm -f good-pod.yaml
rm -f secure-deployment.yaml
rm -f test-namespace-pod.yaml
rm -f exempt-pod.yaml
Conclusion
In this lab, you have successfully:

• Deployed OPA Gatekeeper as an admission controller in your Kubernetes cluster • Created security policies using ConstraintTemplates and Constraints to prevent containers from running as root • Tested policy enforcement by attempting to deploy both compliant and non-compliant workloads • Implemented advanced policy features including namespace exclusions and label-based exemptions • Learned troubleshooting techniques for common admission control issues

Why This Matters: Admission controllers are a critical security layer in Kubernetes that help enforce organizational policies and security standards before resources are created. By preventing containers from running as root, you significantly reduce the attack surface and potential impact of container compromises. This proactive approach to security is essential for production Kubernetes environments and is a key requirement for many compliance frameworks.

The skills you've learned in this lab are directly applicable to the Kubernetes and Cloud Native Security Associate (KCSA) certification and real-world Kubernetes security implementations. Understanding admission controls is fundamental to implementing a comprehensive Kubernetes security strategy.
