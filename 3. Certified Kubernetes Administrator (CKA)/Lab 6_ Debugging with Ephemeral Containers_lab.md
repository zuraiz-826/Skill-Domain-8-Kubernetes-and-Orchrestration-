Lab 6: Debugging with Ephemeral Containers
Objectives
By the end of this lab, you will be able to:

• Understand the concept and benefits of ephemeral containers in Kubernetes • Deploy a broken application Pod and identify common failure scenarios • Use kubectl exec to investigate container environments and troubleshoot issues • Create and attach ephemeral containers to running Pods for advanced debugging • Utilize debugging tools within ephemeral containers to diagnose application problems • Apply best practices for container debugging in production environments • Understand when to use ephemeral containers versus other debugging methods

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, containers, namespaces) • Familiarity with Linux command-line operations • Knowledge of kubectl basic commands • Understanding of container concepts and Docker basics • Experience with YAML configuration files • Basic troubleshooting skills in Linux environments

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes - everything is ready for you to begin immediately.

Your lab environment includes: • Kubernetes cluster (single-node setup) • kubectl command-line tool • All necessary debugging utilities • Pre-configured access permissions

Lab Tasks
Task 1: Deploy a Broken Application Pod
In this task, you will deploy a Pod that has intentional configuration issues to simulate real-world debugging scenarios.

Subtask 1.1: Create a Broken Pod Configuration
First, let's create a Pod configuration file that contains a common error - referencing a non-existent ConfigMap.

# Create the lab directory
mkdir -p ~/k8s-debug-lab
cd ~/k8s-debug-lab
Create a file named broken-app.yaml:

cat > broken-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: broken-app
  labels:
    app: debug-demo
spec:
  containers:
  - name: web-server
    image: nginx:1.21
    ports:
    - containerPort: 80
    env:
    - name: CONFIG_FILE
      value: "/etc/config/app.conf"
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: app-data
      mountPath: /var/app-data
  volumes:
  - name: config-volume
    configMap:
      name: app-config  # This ConfigMap doesn't exist - intentional error
  - name: app-data
    emptyDir: {}
  restartPolicy: Always
EOF
Subtask 1.2: Deploy the Broken Pod
Deploy the Pod and observe the failure:

# Apply the broken Pod configuration
kubectl apply -f broken-app.yaml

# Check the Pod status
kubectl get pods

# Get detailed information about the Pod
kubectl describe pod broken-app
You should see the Pod in a Pending state with an error message about the missing ConfigMap.

Subtask 1.3: Create a Working Pod for Comparison
Let's also create a working version to understand the difference:

cat > working-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: working-app
  labels:
    app: debug-demo-working
spec:
  containers:
  - name: web-server
    image: nginx:1.21
    ports:
    - containerPort: 80
    env:
    - name: SERVER_NAME
      value: "debug-server"
    volumeMounts:
    - name: app-data
      mountPath: /var/app-data
  volumes:
  - name: app-data
    emptyDir: {}
  restartPolicy: Always
EOF
Deploy the working Pod:

# Deploy the working Pod
kubectl apply -f working-app.yaml

# Verify it's running
kubectl get pods
Task 2: Use kubectl exec to Investigate Container Environment
Now let's use kubectl exec to investigate the working Pod and understand the container environment.

Subtask 2.1: Basic Container Investigation
Execute commands inside the working container:

# Get a shell inside the working Pod
kubectl exec -it working-app -- /bin/bash

# Once inside the container, explore the environment
# Check the current directory
pwd

# List environment variables
env | grep -E "(SERVER_NAME|PATH|NGINX)"

# Check mounted volumes
df -h

# Examine the nginx configuration
cat /etc/nginx/nginx.conf

# Check if nginx is running
ps aux | grep nginx

# Exit the container
exit
Subtask 2.2: Investigate File System and Processes
Let's examine the container's file system and running processes:

# Check the container's file system without entering interactive mode
kubectl exec working-app -- ls -la /

# Check mounted volumes
kubectl exec working-app -- ls -la /var/app-data

# Check running processes
kubectl exec working-app -- ps aux

# Check network configuration
kubectl exec working-app -- netstat -tulpn

# Check system resources
kubectl exec working-app -- free -h
kubectl exec working-app -- df -h
Subtask 2.3: Examine Logs and Troubleshoot
Check the application logs and system information:

# View Pod logs
kubectl logs working-app

# Check nginx access logs
kubectl exec working-app -- tail -f /var/log/nginx/access.log &

# In another terminal, make a request to test the server
kubectl exec working-app -- curl localhost:80

# Stop the tail command (Ctrl+C)

# Check nginx error logs
kubectl exec working-app -- cat /var/log/nginx/error.log
Task 3: Add an Ephemeral Container for Advanced Troubleshooting
Now let's use ephemeral containers to debug our broken Pod. First, we need to fix the broken Pod so it starts, then we'll simulate a runtime issue.

Subtask 3.1: Fix the Broken Pod and Create a Runtime Issue
First, let's create the missing ConfigMap and update our Pod:

# Create the missing ConfigMap
kubectl create configmap app-config --from-literal=app.conf="server_name=debug-app"

# Delete and recreate the broken Pod
kubectl delete pod broken-app
kubectl apply -f broken-app.yaml

# Verify the Pod is now running
kubectl get pods
Now let's simulate a runtime issue by creating a Pod that has problems during execution:

cat > problematic-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: problematic-app
  labels:
    app: debug-demo-problem
spec:
  containers:
  - name: app-container
    image: busybox:1.35
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo 'App running...'; sleep 30; done"]
    env:
    - name: DEBUG_MODE
      value: "true"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  restartPolicy: Always
EOF
Deploy the problematic Pod:

kubectl apply -f problematic-app.yaml
kubectl get pods
Subtask 3.2: Create an Ephemeral Container
Now let's add an ephemeral container to the problematic Pod for debugging:

# Add an ephemeral container with debugging tools
kubectl debug problematic-app -it --image=ubuntu:20.04 --target=app-container
If the above command doesn't work (older Kubernetes versions), use this alternative approach:

# Create an ephemeral container configuration
cat > ephemeral-debug.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: problematic-app
spec:
  ephemeralContainers:
  - name: debugger
    image: ubuntu:20.04
    command: ["/bin/bash"]
    stdin: true
    tty: true
    targetContainerName: app-container
EOF
Apply the ephemeral container:

# Patch the Pod to add the ephemeral container
kubectl patch pod problematic-app --subresource ephemeralcontainers -p '{"spec":{"ephemeralContainers":[{"name":"debugger","image":"ubuntu:20.04","command":["/bin/bash"],"stdin":true,"tty":true,"targetContainerName":"app-container"}]}}'

# Connect to the ephemeral container
kubectl attach problematic-app -c debugger -it
Subtask 3.3: Use Debugging Tools in the Ephemeral Container
Once inside the ephemeral container, install and use debugging tools:

# Update package manager and install debugging tools
apt-get update
apt-get install -y procps net-tools curl htop strace tcpdump

# Check processes from the target container
ps aux

# Check network connections
netstat -tulpn

# Monitor system resources
htop

# Check file system usage
df -h

# Examine the target container's file system
ls -la /proc/1/root/

# Check environment variables of the main process
cat /proc/1/environ | tr '\0' '\n'

# Monitor system calls (if needed)
# strace -p 1

# Exit the ephemeral container
exit
Subtask 3.4: Advanced Debugging Techniques
Let's create a more complex debugging scenario:

cat > complex-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: complex-app
  labels:
    app: complex-debug
spec:
  containers:
  - name: web-app
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: data-processor
    image: busybox:1.35
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo '<h1>Data processed at $(date)</h1>' > /shared/index.html; sleep 60; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
EOF
Deploy and debug the complex application:

# Deploy the complex app
kubectl apply -f complex-app.yaml

# Wait for it to be ready
kubectl wait --for=condition=Ready pod/complex-app --timeout=60s

# Add an ephemeral container for debugging
kubectl debug complex-app -it --image=nicolaka/netshoot --target=web-app

# Inside the ephemeral container, perform network debugging
# Check if the web server is responding
curl localhost:80

# Check network interfaces
ip addr show

# Check listening ports
ss -tulpn

# Check DNS resolution
nslookup kubernetes.default.svc.cluster.local

# Test connectivity to other services
# ping google.com

# Exit the debugging container
exit
Task 4: Compare Debugging Methods
Let's compare different debugging approaches and understand when to use each:

Subtask 4.1: Create a Comparison Table
Create a summary of debugging methods:

cat > debugging-comparison.md << 'EOF'
# Kubernetes Debugging Methods Comparison

## kubectl exec
- **Use Case**: Quick investigation of running containers
- **Pros**: Simple, direct access to container
- **Cons**: Limited to tools available in the container
- **Best For**: Basic troubleshooting, log checking

## Ephemeral Containers
- **Use Case**: Advanced debugging without modifying the original container
- **Pros**: Full debugging toolkit, doesn't affect main container
- **Cons**: Requires Kubernetes 1.18+, more complex setup
- **Best For**: Production debugging, complex issues

## kubectl logs
- **Use Case**: Checking application output and errors
- **Pros**: Non-intrusive, historical data available
- **Cons**: Limited to what application logs
- **Best For**: Initial problem identification

## kubectl describe
- **Use Case**: Understanding Pod configuration and events
- **Pros**: Shows configuration issues and Kubernetes events
- **Cons**: Doesn't show runtime issues
- **Best For**: Configuration problems, scheduling issues
EOF

cat debugging-comparison.md
Subtask 4.2: Practice Different Debugging Scenarios
Let's practice with different types of issues:

Scenario 1: Configuration Issue

# Create a Pod with wrong image
cat > wrong-image.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: wrong-image-pod
spec:
  containers:
  - name: app
    image: nonexistent-image:latest
EOF

kubectl apply -f wrong-image.yaml
kubectl describe pod wrong-image-pod
kubectl get events --sort-by=.metadata.creationTimestamp
Scenario 2: Resource Issue

# Create a Pod with insufficient resources
cat > resource-issue.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: resource-issue-pod
spec:
  containers:
  - name: app
    image: nginx:1.21
    resources:
      requests:
        memory: "10Gi"  # Requesting too much memory
        cpu: "8"        # Requesting too much CPU
EOF

kubectl apply -f resource-issue.yaml
kubectl describe pod resource-issue-pod
Scenario 3: Network Issue

# Create a Pod and test network connectivity
kubectl run network-test --image=busybox:1.35 --restart=Never -- sleep 3600

# Debug network issues using ephemeral container
kubectl debug network-test -it --image=nicolaka/netshoot

# Inside the debugging container:
# nslookup kubernetes.default.svc.cluster.local
# ping 8.8.8.8
# traceroute google.com
# exit
Task 5: Clean Up and Best Practices
Subtask 5.1: Clean Up Resources
Remove all the resources created during the lab:

# Delete all Pods created in this lab
kubectl delete pod broken-app working-app problematic-app complex-app wrong-image-pod resource-issue-pod network-test

# Delete the ConfigMap
kubectl delete configmap app-config

# Verify cleanup
kubectl get pods
kubectl get configmaps
Subtask 5.2: Document Best Practices
Create a best practices document:

cat > debugging-best-practices.md << 'EOF'
# Kubernetes Debugging Best Practices

## When to Use Each Method

### kubectl logs
- First step in debugging
- Check for application errors
- Use `-f` for real-time monitoring
- Use `--previous` for crashed containers

### kubectl describe
- Configuration and scheduling issues
- Check events section for problems
- Verify resource requests and limits
- Check volume mounts and secrets

### kubectl exec
- Quick investigation of running containers
- Check file system and environment
- Verify application configuration
- Limited to tools in the container

### Ephemeral Containers
- Production debugging without disruption
- When main container lacks debugging tools
- Complex network or performance issues
- Multi-container Pod debugging

## Security Considerations
- Limit debugging access in production
- Use minimal debugging images
- Remove ephemeral containers after debugging
- Audit debugging activities

## Performance Impact
- Ephemeral containers share resources
- Monitor resource usage during debugging
- Clean up debugging containers promptly
- Use resource limits for debugging containers
EOF

cat debugging-best-practices.md
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Ephemeral containers not supported

# Check Kubernetes version
kubectl version --short

# Ephemeral containers require Kubernetes 1.18+
# If not available, use kubectl exec or create debugging Pods
Issue 2: Permission denied when using kubectl debug

# Check your permissions
kubectl auth can-i create pods/ephemeralcontainers

# If denied, use alternative debugging methods
kubectl run debug-pod --image=ubuntu:20.04 --rm -it -- /bin/bash
Issue 3: Debugging container cannot access target container files

# Use process namespace sharing
kubectl debug pod-name -it --image=ubuntu:20.04 --share-processes --copy-to=debug-copy
Issue 4: Network debugging tools not available

# Use specialized debugging images
kubectl debug pod-name -it --image=nicolaka/netshoot
# or
kubectl debug pod-name -it --image=busybox:1.35
Conclusion
In this lab, you have successfully learned how to debug Kubernetes applications using various methods, with a focus on ephemeral containers. You have:

• Deployed and diagnosed broken applications - Understanding common failure patterns helps you quickly identify issues in real-world scenarios

• Mastered kubectl exec for basic debugging - This fundamental skill allows you to investigate running containers and gather essential troubleshooting information

• Implemented ephemeral containers for advanced debugging - This modern approach enables sophisticated debugging without disrupting production workloads

• Compared different debugging methodologies - Understanding when to use each method makes you a more effective troubleshooter

• Applied best practices for container debugging - Following established practices ensures secure and efficient debugging processes

Why This Matters:

Debugging skills are crucial for maintaining reliable Kubernetes applications in production environments. Ephemeral containers represent a significant advancement in debugging capabilities, allowing you to troubleshoot issues without modifying or restarting your applications. This approach minimizes downtime and provides powerful debugging capabilities that are essential for modern containerized applications.

The techniques you've learned in this lab will help you:

Quickly identify and resolve application issues
Debug production problems without service disruption
Use the right debugging approach for each situation
Maintain security and performance while troubleshooting
These skills are particularly valuable for the Certified Kubernetes Application Developer (CKAD) certification and real-world Kubernetes operations, where efficient debugging can mean the difference between minutes and hours of downtime.
