Lab 2: Troubleshooting Lab
Objectives
By the end of this lab, students will be able to:

• Analyze and interpret logs from kube-apiserver and kubelet components using kubectl logs and journalctl commands • Debug failing Pods systematically using kubectl describe and kubectl exec commands • Test and troubleshoot network connectivity issues between Pods using ping and traceroute utilities • Apply systematic troubleshooting methodologies to identify and resolve common Kubernetes issues • Understand the relationship between different Kubernetes components and their log outputs

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes architecture and components • Familiarity with Linux command-line interface • Knowledge of basic networking concepts (IP addresses, ports, DNS) • Understanding of Pod lifecycle and container concepts • Basic experience with kubectl commands

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed. Simply click Start Lab to access your environment - no need to build your own VM or install software.

Your lab environment includes: • A running Kubernetes cluster with master and worker nodes • kubectl command-line tool pre-configured • All necessary troubleshooting utilities installed • Sample applications and configurations for testing

Task 1: Analyzing Kubernetes Component Logs
Subtask 1.1: Examining kube-apiserver Logs
The kube-apiserver is the central management component of Kubernetes. Let's learn how to analyze its logs for troubleshooting.

Step 1: First, identify how kube-apiserver is running in your cluster

kubectl get pods -n kube-system | grep apiserver
Step 2: If kube-apiserver runs as a static pod, view its logs using kubectl

kubectl logs -n kube-system kube-apiserver-$(hostname) --tail=50
Step 3: If kube-apiserver runs as a system service, use journalctl

sudo journalctl -u kube-apiserver --no-pager --lines=50
Step 4: Filter logs for specific error patterns

kubectl logs -n kube-system kube-apiserver-$(hostname) | grep -i error
Step 5: Monitor real-time logs to observe API server activity

kubectl logs -n kube-system kube-apiserver-$(hostname) -f
Press Ctrl+C to stop the real-time monitoring.

Subtask 1.2: Examining kubelet Logs
The kubelet manages Pods on each node. Understanding its logs is crucial for node-level troubleshooting.

Step 1: Check kubelet service status

sudo systemctl status kubelet
Step 2: View recent kubelet logs using journalctl

sudo journalctl -u kubelet --no-pager --lines=50
Step 3: Filter kubelet logs for Pod-related issues

sudo journalctl -u kubelet | grep -i "failed\|error\|warning"
Step 4: Monitor kubelet logs in real-time

sudo journalctl -u kubelet -f
Step 5: Check kubelet logs for a specific time period

sudo journalctl -u kubelet --since "10 minutes ago"
Subtask 1.3: Creating a Log Analysis Script
Create a comprehensive script to gather logs from multiple sources.

Step 1: Create a log collection script

cat > collect_k8s_logs.sh << 'EOF'
#!/bin/bash

echo "=== Kubernetes Troubleshooting Log Collection ==="
echo "Timestamp: $(date)"
echo

echo "=== Cluster Info ==="
kubectl cluster-info
echo

echo "=== Node Status ==="
kubectl get nodes -o wide
echo

echo "=== System Pods Status ==="
kubectl get pods -n kube-system
echo

echo "=== Recent kubelet logs ==="
sudo journalctl -u kubelet --no-pager --lines=20
echo

echo "=== API Server logs (if available) ==="
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep apiserver | awk '{print $1}') --tail=20 2>/dev/null || echo "API server logs not accessible via kubectl"
echo

echo "=== Controller Manager logs ==="
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep controller-manager | awk '{print $1}') --tail=20 2>/dev/null || echo "Controller manager logs not accessible"
echo

echo "=== Scheduler logs ==="
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep scheduler | awk '{print $1}') --tail=20 2>/dev/null || echo "Scheduler logs not accessible"

EOF
Step 2: Make the script executable and run it

chmod +x collect_k8s_logs.sh
./collect_k8s_logs.sh
Task 2: Debugging a Failing Pod
Subtask 2.1: Creating a Problematic Pod for Testing
Let's create a Pod with intentional issues to practice debugging.

Step 1: Create a Pod with an incorrect image name

cat > failing-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: failing-app
  labels:
    app: debug-test
spec:
  containers:
  - name: web-server
    image: nginx:nonexistent-tag
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
Step 2: Apply the problematic Pod configuration

kubectl apply -f failing-pod.yaml
Step 3: Observe the Pod status

kubectl get pods failing-app
Subtask 2.2: Using kubectl describe for Debugging
The kubectl describe command provides detailed information about Kubernetes resources.

Step 1: Get detailed information about the failing Pod

kubectl describe pod failing-app
Step 2: Focus on the Events section to identify issues

kubectl describe pod failing-app | grep -A 10 "Events:"
Step 3: Check the Pod's current status and conditions

kubectl get pod failing-app -o yaml | grep -A 20 "status:"
Subtask 2.3: Analyzing Pod Logs
Step 1: Attempt to view Pod logs (this will likely fail due to the container not starting)

kubectl logs failing-app
Step 2: Check logs for previous container instances if available

kubectl logs failing-app --previous
Subtask 2.4: Creating a Working Pod for Comparison
Step 1: Create a working Pod configuration

cat > working-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: working-app
  labels:
    app: debug-test
spec:
  containers:
  - name: web-server
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
Step 2: Apply the working Pod configuration

kubectl apply -f working-pod.yaml
Step 3: Verify the working Pod is running

kubectl get pods working-app
Subtask 2.5: Using kubectl exec for Interactive Debugging
Step 1: Execute commands inside the working Pod

kubectl exec working-app -- ps aux
Step 2: Start an interactive shell session

kubectl exec -it working-app -- /bin/bash
Step 3: Inside the Pod, run diagnostic commands

# Run these commands inside the Pod shell
whoami
hostname
cat /etc/os-release
nginx -v
curl localhost
exit
Step 4: Execute specific diagnostic commands without interactive shell

kubectl exec working-app -- curl -s localhost
kubectl exec working-app -- cat /var/log/nginx/access.log
Subtask 2.6: Fixing the Failing Pod
Step 1: Delete the failing Pod

kubectl delete pod failing-app
Step 2: Create a corrected version

cat > fixed-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: fixed-app
  labels:
    app: debug-test
spec:
  containers:
  - name: web-server
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
Step 3: Apply the fixed configuration

kubectl apply -f fixed-pod.yaml
Step 4: Verify the fix worked

kubectl get pods fixed-app
kubectl describe pod fixed-app
Task 3: Testing and Troubleshooting Network Connectivity
Subtask 3.1: Setting Up Test Pods for Network Testing
Step 1: Create multiple Pods for network connectivity testing

cat > network-test-pods.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
  labels:
    app: network-test
    role: client
spec:
  containers:
  - name: client
    image: busybox:1.35
    command: ['sleep', '3600']
    resources:
      requests:
        memory: "32Mi"
        cpu: "100m"
---
apiVersion: v1
kind: Pod
metadata:
  name: server-pod
  labels:
    app: network-test
    role: server
spec:
  containers:
  - name: server
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
---
apiVersion: v1
kind: Service
metadata:
  name: server-service
spec:
  selector:
    role: server
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
Step 2: Apply the network test configuration

kubectl apply -f network-test-pods.yaml
Step 3: Verify all Pods are running

kubectl get pods -l app=network-test
kubectl get service server-service
Subtask 3.2: Testing Basic Network Connectivity with Ping
Step 1: Get the IP addresses of the test Pods

kubectl get pods -l app=network-test -o wide
Step 2: Store the server Pod IP for testing

SERVER_POD_IP=$(kubectl get pod server-pod -o jsonpath='{.status.podIP}')
echo "Server Pod IP: $SERVER_POD_IP"
Step 3: Test ping connectivity from client to server Pod

kubectl exec client-pod -- ping -c 4 $SERVER_POD_IP
Step 4: Test ping to the service IP

SERVICE_IP=$(kubectl get service server-service -o jsonpath='{.spec.clusterIP}')
echo "Service IP: $SERVICE_IP"
kubectl exec client-pod -- ping -c 4 $SERVICE_IP
Step 5: Test ping to external addresses

kubectl exec client-pod -- ping -c 4 8.8.8.8
kubectl exec client-pod -- ping -c 4 google.com
Subtask 3.3: Testing HTTP Connectivity
Step 1: Test HTTP connectivity to the server Pod directly

kubectl exec client-pod -- wget -qO- http://$SERVER_POD_IP
Step 2: Test HTTP connectivity via the service

kubectl exec client-pod -- wget -qO- http://server-service
Step 3: Test with curl for more detailed output

kubectl exec client-pod -- wget -O- --timeout=10 http://server-service 2>&1
Subtask 3.4: Using Traceroute for Network Path Analysis
Step 1: Install traceroute in the client Pod

kubectl exec client-pod -- sh -c 'which traceroute || echo "traceroute not available in busybox"'
Step 2: Use alternative network diagnostic tools available in busybox

kubectl exec client-pod -- netstat -rn
kubectl exec client-pod -- route -n
Step 3: Test network connectivity with telnet-like functionality

kubectl exec client-pod -- nc -zv $SERVER_POD_IP 80
kubectl exec client-pod -- nc -zv server-service 80
Subtask 3.5: Advanced Network Troubleshooting
Step 1: Create a more advanced debugging Pod with additional tools

cat > debug-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  labels:
    app: network-debug
spec:
  containers:
  - name: debug-tools
    image: nicolaka/netshoot:latest
    command: ['sleep', '3600']
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
EOF
Step 2: Apply the debug Pod configuration

kubectl apply -f debug-pod.yaml
Step 3: Wait for the debug Pod to be ready

kubectl wait --for=condition=Ready pod/debug-pod --timeout=60s
Step 4: Use advanced networking tools from the debug Pod

kubectl exec debug-pod -- nslookup server-service
kubectl exec debug-pod -- dig server-service
kubectl exec debug-pod -- traceroute $SERVER_POD_IP
Step 5: Analyze network interfaces and routing

kubectl exec debug-pod -- ip addr show
kubectl exec debug-pod -- ip route show
kubectl exec debug-pod -- ss -tuln
Subtask 3.6: Troubleshooting DNS Resolution
Step 1: Test DNS resolution within the cluster

kubectl exec debug-pod -- nslookup kubernetes.default.svc.cluster.local
kubectl exec debug-pod -- nslookup server-service.default.svc.cluster.local
Step 2: Check DNS configuration

kubectl exec debug-pod -- cat /etc/resolv.conf
Step 3: Test external DNS resolution

kubectl exec debug-pod -- nslookup google.com
kubectl exec debug-pod -- dig @8.8.8.8 google.com
Subtask 3.7: Creating a Network Troubleshooting Script
Step 1: Create a comprehensive network testing script

cat > network_troubleshoot.sh << 'EOF'
#!/bin/bash

echo "=== Kubernetes Network Troubleshooting Script ==="
echo "Timestamp: $(date)"
echo

# Get Pod and Service information
echo "=== Pod and Service Information ==="
kubectl get pods -o wide
echo
kubectl get services
echo

# Test basic connectivity
echo "=== Testing Basic Connectivity ==="
SERVER_POD_IP=$(kubectl get pod server-pod -o jsonpath='{.status.podIP}' 2>/dev/null)
SERVICE_IP=$(kubectl get service server-service -o jsonpath='{.spec.clusterIP}' 2>/dev/null)

if [ ! -z "$SERVER_POD_IP" ]; then
    echo "Testing ping to server Pod ($SERVER_POD_IP):"
    kubectl exec client-pod -- ping -c 2 $SERVER_POD_IP 2>/dev/null || echo "Ping to server Pod failed"
    echo
fi

if [ ! -z "$SERVICE_IP" ]; then
    echo "Testing ping to service ($SERVICE_IP):"
    kubectl exec client-pod -- ping -c 2 $SERVICE_IP 2>/dev/null || echo "Ping to service failed"
    echo
fi

# Test HTTP connectivity
echo "=== Testing HTTP Connectivity ==="
if [ ! -z "$SERVER_POD_IP" ]; then
    echo "Testing HTTP to server Pod:"
    kubectl exec client-pod -- wget -qO- --timeout=5 http://$SERVER_POD_IP 2>/dev/null | head -5 || echo "HTTP to server Pod failed"
    echo
fi

echo "Testing HTTP to service:"
kubectl exec client-pod -- wget -qO- --timeout=5 http://server-service 2>/dev/null | head -5 || echo "HTTP to service failed"
echo

# Test DNS resolution
echo "=== Testing DNS Resolution ==="
kubectl exec debug-pod -- nslookup server-service 2>/dev/null || echo "DNS resolution test failed"
echo

echo "=== Network Troubleshooting Complete ==="
EOF
Step 2: Make the script executable and run it

chmod +x network_troubleshoot.sh
./network_troubleshoot.sh
Troubleshooting Common Issues
Issue 1: Pod Stuck in Pending State
Symptoms: Pod remains in Pending status Debugging Steps:

kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
Common Causes: • Insufficient resources on nodes • Node selector constraints not met • Persistent volume claims not bound

Issue 2: Pod Stuck in ImagePullBackOff
Symptoms: Pod cannot pull container image Debugging Steps:

kubectl describe pod <pod-name>
kubectl logs <pod-name>
Common Causes: • Incorrect image name or tag • Private registry authentication issues • Network connectivity problems

Issue 3: Network Connectivity Issues
Symptoms: Pods cannot communicate with each other Debugging Steps:

kubectl exec <pod-name> -- ping <target-ip>
kubectl exec <pod-name> -- nslookup <service-name>
Common Causes: • Network policy restrictions • DNS configuration issues • Service misconfiguration

Cleanup
Clean up the resources created during this lab:

kubectl delete pod failing-app --ignore-not-found=true
kubectl delete pod working-app --ignore-not-found=true
kubectl delete pod fixed-app --ignore-not-found=true
kubectl delete -f network-test-pods.yaml --ignore-not-found=true
kubectl delete pod debug-pod --ignore-not-found=true
rm -f failing-pod.yaml working-pod.yaml fixed-pod.yaml network-test-pods.yaml debug-pod.yaml
rm -f collect_k8s_logs.sh network_troubleshoot.sh
Conclusion
In this comprehensive troubleshooting lab, you have accomplished several critical skills:

Key Achievements: • Log Analysis Mastery: You learned to analyze logs from essential Kubernetes components (kube-apiserver and kubelet) using both kubectl logs and journalctl commands, enabling you to identify system-level issues quickly.

• Pod Debugging Expertise: You gained hands-on experience debugging failing Pods using kubectl describe and kubectl exec, understanding how to interpret error messages and system events to resolve container issues.

• Network Troubleshooting Skills: You developed proficiency in testing and troubleshooting network connectivity between Pods using ping, wget, and advanced networking tools, essential for maintaining reliable cluster communication.

• Systematic Problem-Solving: You learned to approach Kubernetes issues methodically, using multiple diagnostic tools and techniques to identify root causes rather than just symptoms.

Why This Matters: These troubleshooting skills are fundamental for anyone working with Kubernetes in production environments. The ability to quickly diagnose and resolve issues minimizes downtime, improves system reliability, and builds confidence in managing complex containerized applications. These competencies are essential for the Certified Kubernetes Administrator (CKA) certification and real-world Kubernetes operations.

The systematic approach you've learned here - combining log analysis, resource inspection, and network testing - provides a solid foundation for handling the wide variety of issues you'll encounter in production Kubernetes environments.

