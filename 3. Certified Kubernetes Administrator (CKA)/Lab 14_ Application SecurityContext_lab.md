Lab 14: Application SecurityContext
Objectives
By the end of this lab, you will be able to:

• Understand the importance of SecurityContext in Kubernetes Pod security • Configure a Pod with a restricted SecurityContext including read-only root filesystem • Test and analyze application behavior under security constraints • Examine the impact of Linux capabilities like NET_ADMIN on Pod functionality • Implement security best practices for containerized applications • Troubleshoot common issues related to SecurityContext configurations

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, containers, YAML manifests) • Familiarity with Linux file systems and permissions • Knowledge of container security fundamentals • Experience with kubectl command-line tool • Understanding of Linux capabilities and their security implications

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Kubernetes cluster (single-node or multi-node) • kubectl configured and ready to use • Text editor (nano, vim, or code editor) • All necessary permissions to create and manage Kubernetes resources

Task 1: Understanding SecurityContext Fundamentals
Subtask 1.1: Explore Current Security Settings
First, let's examine the default security settings of a basic Pod.

Create a basic Pod without any SecurityContext:
# Create file: basic-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-pod
  labels:
    app: security-demo
spec:
  containers:
  - name: demo-container
    image: nginx:1.21
    command: ["/bin/sh"]
    args: ["-c", "sleep 3600"]
Apply the Pod configuration:
kubectl apply -f basic-pod.yaml
Wait for the Pod to be running:
kubectl wait --for=condition=Ready pod/basic-pod --timeout=60s
Examine the security context of the running container:
kubectl exec basic-pod -- id
kubectl exec basic-pod -- ps aux
kubectl exec basic-pod -- ls -la /
Subtask 1.2: Analyze Default Security Posture
Check file system permissions:
kubectl exec basic-pod -- ls -la /tmp
kubectl exec basic-pod -- touch /tmp/test-file
kubectl exec basic-pod -- ls -la /tmp/test-file
Examine process capabilities:
kubectl exec basic-pod -- cat /proc/1/status | grep Cap
Test write access to root filesystem:
kubectl exec basic-pod -- touch /test-root-write
kubectl exec basic-pod -- ls -la /test-root-write
Task 2: Configure Pod with Restricted SecurityContext
Subtask 2.1: Create Pod with Read-Only Root Filesystem
Now we'll create a Pod with enhanced security settings including a read-only root filesystem.

Create a restricted SecurityContext Pod:
# Create file: restricted-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
  labels:
    app: security-demo-restricted
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: demo-container
    image: nginx:1.21
    command: ["/bin/sh"]
    args: ["-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
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
Apply the restricted Pod configuration:
kubectl apply -f restricted-pod.yaml
Wait for the Pod to be ready:
kubectl wait --for=condition=Ready pod/restricted-pod --timeout=60s
Subtask 2.2: Verify Security Restrictions
Check the user and group IDs:
kubectl exec restricted-pod -- id
Test read-only root filesystem:
kubectl exec restricted-pod -- touch /test-root-write
Expected Result: This should fail with a "Read-only file system" error.

Verify writable mounted volumes:
kubectl exec restricted-pod -- touch /tmp/test-tmp-write
kubectl exec restricted-pod -- ls -la /tmp/test-tmp-write
Check dropped capabilities:
kubectl exec restricted-pod -- cat /proc/1/status | grep Cap
Task 3: Test Application Behavior with SecurityContext
Subtask 3.1: Deploy Application with Security Constraints
Let's deploy a more realistic application to see how SecurityContext affects functionality.

Create a web application with restricted security:
# Create file: secure-webapp.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-webapp
  labels:
    app: secure-web
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
  containers:
  - name: webapp
    image: nginx:1.21
    ports:
    - containerPort: 8080
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1001
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: tmp-volume
      mountPath: /tmp
    - name: var-cache-nginx
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
  - name: tmp-volume
    emptyDir: {}
  - name: var-cache-nginx
    emptyDir: {}
  - name: var-run
    emptyDir: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 8080;
            location / {
                return 200 'Hello from Secure Web App!\n';
                add_header Content-Type text/plain;
            }
        }
    }
Apply the secure web application:
kubectl apply -f secure-webapp.yaml
Wait for the Pod to be ready:
kubectl wait --for=condition=Ready pod/secure-webapp --timeout=60s
Subtask 3.2: Test Application Functionality
Test the web application:
kubectl port-forward pod/secure-webapp 8080:8080 &
curl http://localhost:8080
Stop the port-forward process:
pkill -f "kubectl port-forward"
Examine the application's security posture:
kubectl exec secure-webapp -- id
kubectl exec secure-webapp -- ps aux
kubectl exec secure-webapp -- ls -la /etc/nginx/
Task 4: Analyze Impact of Linux Capabilities
Subtask 4.1: Create Pod with NET_ADMIN Capability
Now let's explore how specific Linux capabilities affect Pod behavior.

Create a Pod with NET_ADMIN capability:
# Create file: netadmin-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: netadmin-pod
  labels:
    app: network-admin-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
  containers:
  - name: network-tools
    image: nicolaka/netshoot:latest
    command: ["/bin/sh"]
    args: ["-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - NET_ADMIN
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
  volumes:
  - name: tmp-volume
    emptyDir: {}
Apply the NET_ADMIN Pod:
kubectl apply -f netadmin-pod.yaml
Wait for the Pod to be ready:
kubectl wait --for=condition=Ready pod/netadmin-pod --timeout=60s
Subtask 4.2: Test Network Administration Capabilities
Check current network interfaces:
kubectl exec netadmin-pod -- ip addr show
Test NET_ADMIN capability by creating a dummy network interface:
kubectl exec netadmin-pod -- ip link add dummy0 type dummy
kubectl exec netadmin-pod -- ip addr show dummy0
Clean up the dummy interface:
kubectl exec netadmin-pod -- ip link delete dummy0
Subtask 4.3: Compare with Pod Without NET_ADMIN
Create a Pod without NET_ADMIN capability:
# Create file: no-netadmin-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-netadmin-pod
  labels:
    app: no-network-admin-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
  containers:
  - name: network-tools
    image: nicolaka/netshoot:latest
    command: ["/bin/sh"]
    args: ["-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
  volumes:
  - name: tmp-volume
    emptyDir: {}
Apply the Pod without NET_ADMIN:
kubectl apply -f no-netadmin-pod.yaml
Wait for the Pod to be ready:
kubectl wait --for=condition=Ready pod/no-netadmin-pod --timeout=60s
Try to create a dummy interface (this should fail):
kubectl exec no-netadmin-pod -- ip link add dummy0 type dummy
Expected Result: This should fail with "Operation not permitted" error.

Task 5: Security Analysis and Best Practices
Subtask 5.1: Compare Security Postures
Create a comparison script to analyze all Pods:
# Create file: security-analysis.sh
#!/bin/bash

echo "=== Security Analysis Report ==="
echo

for pod in basic-pod restricted-pod secure-webapp netadmin-pod no-netadmin-pod; do
    echo "--- Pod: $pod ---"
    if kubectl get pod $pod &>/dev/null; then
        echo "User ID: $(kubectl exec $pod -- id -u 2>/dev/null || echo 'N/A')"
        echo "Group ID: $(kubectl exec $pod -- id -g 2>/dev/null || echo 'N/A')"
        echo "Root filesystem writable: $(kubectl exec $pod -- touch /test-write 2>/dev/null && echo 'Yes' || echo 'No')"
        kubectl exec $pod -- rm -f /test-write 2>/dev/null
        echo "Capabilities: $(kubectl exec $pod -- cat /proc/1/status 2>/dev/null | grep CapEff || echo 'N/A')"
    else
        echo "Pod not found or not ready"
    fi
    echo
done
Make the script executable and run it:
chmod +x security-analysis.sh
./security-analysis.sh
Subtask 5.2: Implement Pod Security Standards
Create a Pod that follows Pod Security Standards (restricted level):
# Create file: pss-restricted-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pss-restricted-pod
  labels:
    app: pod-security-standards
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: secure-app
    image: nginx:1.21
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
      seccompProfile:
        type: RuntimeDefault
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: tmp-volume
      mountPath: /tmp
    - name: var-cache-nginx
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
  - name: tmp-volume
    emptyDir: {}
  - name: var-cache-nginx
    emptyDir: {}
  - name: var-run
    emptyDir: {}
Apply the Pod Security Standards compliant Pod:
kubectl apply -f pss-restricted-pod.yaml
Verify the Pod is running with enhanced security:
kubectl wait --for=condition=Ready pod/pss-restricted-pod --timeout=60s
kubectl exec pss-restricted-pod -- id
kubectl exec pss-restricted-pod -- cat /proc/1/status | grep -E "(Seccomp|Cap)"
Task 6: Troubleshooting Common SecurityContext Issues
Subtask 6.1: Identify and Fix Common Problems
Create a Pod with intentional security misconfigurations:
# Create file: problematic-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: problematic-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 0  # This conflicts with runAsNonRoot
  containers:
  - name: problem-container
    image: nginx:1.21
    securityContext:
      readOnlyRootFilesystem: true
      # Missing volume mounts for nginx to work
Try to apply the problematic Pod:
kubectl apply -f problematic-pod.yaml
Check the Pod status and events:
kubectl get pod problematic-pod
kubectl describe pod problematic-pod
Fix the configuration:
# Create file: fixed-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fixed-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000  # Fixed: non-root user ID
    runAsGroup: 1000
  containers:
  - name: fixed-container
    image: nginx:1.21
    ports:
    - containerPort: 8080
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
    - name: tmp-volume
      mountPath: /tmp
    - name: var-cache-nginx
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
  - name: tmp-volume
    emptyDir: {}
  - name: var-cache-nginx
    emptyDir: {}
  - name: var-run
    emptyDir: {}
Apply the fixed Pod:
kubectl apply -f fixed-pod.yaml
kubectl wait --for=condition=Ready pod/fixed-pod --timeout=60s
Subtask 6.2: Create Security Validation Checklist
Create a security validation script:
# Create file: security-validation.sh
#!/bin/bash

POD_NAME=$1

if [ -z "$POD_NAME" ]; then
    echo "Usage: $0 <pod-name>"
    exit 1
fi

echo "=== Security Validation for Pod: $POD_NAME ==="
echo

# Check if pod exists
if ! kubectl get pod $POD_NAME &>/dev/null; then
    echo "❌ Pod $POD_NAME not found"
    exit 1
fi

# Check if running as non-root
USER_ID=$(kubectl exec $POD_NAME -- id -u 2>/dev/null)
if [ "$USER_ID" = "0" ]; then
    echo "❌ Running as root user (UID: $USER_ID)"
else
    echo "✅ Running as non-root user (UID: $USER_ID)"
fi

# Check read-only root filesystem
if kubectl exec $POD_NAME -- touch /test-readonly 2>/dev/null; then
    kubectl exec $POD_NAME -- rm -f /test-readonly 2>/dev/null
    echo "❌ Root filesystem is writable"
else
    echo "✅ Root filesystem is read-only"
fi

# Check capabilities
CAPS=$(kubectl exec $POD_NAME -- cat /proc/1/status 2>/dev/null | grep CapEff | awk '{print $2}')
if [ "$CAPS" = "0000000000000000" ]; then
    echo "✅ All capabilities dropped"
else
    echo "⚠️  Some capabilities present: $CAPS"
fi

# Check privilege escalation
YAML=$(kubectl get pod $POD_NAME -o yaml)
if echo "$YAML" | grep -q "allowPrivilegeEscalation: false"; then
    echo "✅ Privilege escalation disabled"
else
    echo "❌ Privilege escalation not explicitly disabled"
fi

echo
echo "=== Validation Complete ==="
Make the script executable:
chmod +x security-validation.sh
Test the validation script on different Pods:
./security-validation.sh basic-pod
./security-validation.sh restricted-pod
./security-validation.sh pss-restricted-pod
Task 7: Cleanup and Resource Management
Subtask 7.1: Clean Up Lab Resources
Remove all created Pods:
kubectl delete pod basic-pod restricted-pod secure-webapp netadmin-pod no-netadmin-pod pss-restricted-pod problematic-pod fixed-pod --ignore-not-found=true
Remove ConfigMaps:
kubectl delete configmap nginx-config --ignore-not-found=true
Verify cleanup:
kubectl get pods -l app=security-demo
kubectl get pods -l app=security-demo-restricted
kubectl get pods -l app=secure-web
kubectl get pods -l app=network-admin-demo
kubectl get pods -l app=no-network-admin-demo
kubectl get pods -l app=pod-security-standards
Subtask 7.2: Review Security Best Practices
Create a summary document of security best practices learned:

# Create file: security-best-practices.md
cat << 'EOF' > security-best-practices.md
# Kubernetes SecurityContext Best Practices

## Essential Security Settings

1. **Always run as non-root**
   - Set `runAsNonRoot: true`
   - Specify explicit `runAsUser` and `runAsGroup`

2. **Use read-only root filesystem**
   - Set `readOnlyRootFilesystem: true`
   - Mount writable volumes for application data

3. **Drop all capabilities by default**
   - Use `capabilities.drop: [ALL]`
   - Only add specific capabilities when needed

4. **Disable privilege escalation**
   - Set `allowPrivilegeEscalation: false`

5. **Use security profiles**
   - Enable seccomp: `seccompProfile.type: RuntimeDefault`
   - Consider AppArmor or SELinux profiles

## Common Pitfalls to Avoid

- Running containers as root (UID 0)
- Allowing writable root filesystem
- Granting unnecessary Linux capabilities
- Not setting explicit user/group IDs
- Ignoring Pod Security Standards

## Validation Checklist

- [ ] Non-root user configured
- [ ] Read-only root filesystem enabled
- [ ] All capabilities dropped (unless specifically needed)
- [ ] Privilege escalation disabled
- [ ] Security profiles configured
- [ ] Writable volumes mounted only where needed
EOF

echo "Security best practices documented in security-best-practices.md"
Troubleshooting Common Issues
Issue 1: Pod Fails to Start with "container has runAsNonRoot and image will run as root"
Solution: Ensure the container image supports running as non-root, or build a custom image with proper user configuration.

Issue 2: Application Fails with "Permission denied" on read-only filesystem
Solution: Mount emptyDir volumes for directories that need write access (e.g., /tmp, /var/cache, /var/run).

Issue 3: Network operations fail even with NET_ADMIN capability
Solution: Verify the capability is added at the container level, not just the Pod level, and ensure the user has sufficient permissions.

Issue 4: SecurityContext settings are ignored
Solution: Check that settings are applied at the correct level (Pod vs Container) and that there are no conflicting policies.

Conclusion
In this comprehensive lab, you have successfully:

• Configured secure Pods with restricted SecurityContext settings including read-only root filesystems, non-root users, and dropped capabilities • Tested real-world applications under security constraints and learned how to properly configure volumes for applications that need write access • Analyzed the impact of Linux capabilities like NET_ADMIN and understood when and how to grant specific privileges safely • Implemented Pod Security Standards following Kubernetes security best practices for production environments • Developed troubleshooting skills for common SecurityContext configuration issues and created validation tools

Why This Matters: SecurityContext is a critical component of Kubernetes security that helps implement the principle of least privilege. By properly configuring SecurityContext, you significantly reduce the attack surface of your applications and protect against container escape vulnerabilities. These skills are essential for the CKAD certification and for building secure, production-ready Kubernetes applications.

The security practices you've learned in this lab form the foundation of container security in Kubernetes and are directly applicable to real-world scenarios where security compliance and risk mitigation are paramount.
