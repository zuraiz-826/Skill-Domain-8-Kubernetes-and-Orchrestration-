Lab 3: Understanding Kubernetes Architecture
Objectives
By the end of this lab, you will be able to:

• Identify and understand the core components of a Kubernetes cluster • Use kubectl commands to inspect cluster architecture and component status • Examine control plane component logs including API Server and etcd • Create and deploy a Pod while understanding its interaction with control plane and worker nodes • Analyze the communication flow between Kubernetes components • Troubleshoot basic cluster component issues using command-line tools

Prerequisites
Before starting this lab, you should have:

• Basic understanding of containerization concepts (Docker) • Familiarity with Linux command line operations • Basic knowledge of YAML file structure • Understanding of client-server architecture concepts • Completion of previous Kubernetes fundamentals labs or equivalent knowledge

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes manually.

Your lab environment includes: • Ubuntu 22.04 LTS with kubectl pre-installed • A single-node Kubernetes cluster (minikube) ready for use • All necessary tools and permissions configured • Internet access for downloading container images

Task 1: Identifying Kubernetes Cluster Components
Subtask 1.1: Verify Cluster Status and Basic Information
First, let's confirm our cluster is running and gather basic information about its architecture.

Check cluster status:
kubectl cluster-info
Display detailed cluster information:
kubectl cluster-info dump | head -20
List all nodes in the cluster:
kubectl get nodes -o wide
Get detailed information about the node:
kubectl describe nodes
Subtask 1.2: Explore Control Plane Components
The control plane manages the cluster and makes decisions about scheduling and cluster state.

List all pods in the kube-system namespace (where control plane components run):
kubectl get pods -n kube-system
Get detailed information about control plane pods:
kubectl get pods -n kube-system -o wide
Identify specific control plane components:
kubectl get pods -n kube-system | grep -E "(apiserver|etcd|scheduler|controller)"
Subtask 1.3: Examine API Server Component
The API Server is the central component that exposes the Kubernetes API.

Find the API Server pod:
kubectl get pods -n kube-system | grep apiserver
Get detailed information about the API Server:
kubectl describe pod -n kube-system $(kubectl get pods -n kube-system | grep apiserver | awk '{print $1}')
Check API Server service endpoints:
kubectl get endpoints -n kube-system
Subtask 1.4: Examine etcd Component
etcd is the distributed key-value store that holds all cluster data.

Find the etcd pod:
kubectl get pods -n kube-system | grep etcd
Get detailed information about etcd:
kubectl describe pod -n kube-system $(kubectl get pods -n kube-system | grep etcd | awk '{print $1}')
Check etcd health (if accessible):
kubectl exec -n kube-system $(kubectl get pods -n kube-system | grep etcd | awk '{print $1}') -- etcdctl endpoint health
Task 2: Inspecting Control Plane Component Logs
Subtask 2.1: Examining API Server Logs
Understanding API Server logs helps troubleshoot cluster communication issues.

View recent API Server logs:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep apiserver | awk '{print $1}') --tail=50
Follow API Server logs in real-time (open a new terminal for this):
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep apiserver | awk '{print $1}') -f
Search for specific events in API Server logs:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep apiserver | awk '{print $1}') | grep -i "error\|warning" | tail -10
Subtask 2.2: Examining etcd Logs
etcd logs provide insights into cluster state storage and replication.

View recent etcd logs:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep etcd | awk '{print $1}') --tail=30
Check for etcd health-related messages:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep etcd | awk '{print $1}') | grep -i "health\|ready" | tail -5
Monitor etcd performance metrics in logs:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep etcd | awk '{print $1}') | grep -i "slow\|latency" | tail -5
Subtask 2.3: Examining Scheduler and Controller Manager Logs
View Scheduler logs:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep scheduler | awk '{print $1}') --tail=20
View Controller Manager logs:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep controller-manager | awk '{print $1}') --tail=20
Task 3: Creating a Pod and Understanding Component Interactions
Subtask 3.1: Create a Simple Pod
Let's create a Pod and observe how it interacts with various cluster components.

Create a Pod manifest file:
cat > nginx-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  containers:
  - name: nginx
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
Apply the Pod manifest:
kubectl apply -f nginx-pod.yaml
Verify Pod creation:
kubectl get pods
Get detailed Pod information:
kubectl describe pod nginx-demo
Subtask 3.2: Monitor Component Interactions During Pod Creation
Check scheduler logs for Pod scheduling decisions:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep scheduler | awk '{print $1}') | grep nginx-demo
Check API Server logs for Pod-related API calls:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep apiserver | awk '{print $1}') | grep nginx-demo | tail -10
Examine kubelet logs on the node (if accessible):
# Note: In minikube, you might need to access the node directly
minikube ssh "sudo journalctl -u kubelet | grep nginx-demo | tail -5"
Subtask 3.3: Analyze Pod Lifecycle and Component Communication
Check Pod events to understand the creation process:
kubectl get events --sort-by=.metadata.creationTimestamp | grep nginx-demo
Monitor Pod status changes:
kubectl get pod nginx-demo -o yaml | grep -A 10 "status:"
Verify container runtime interaction:
kubectl get pod nginx-demo -o jsonpath='{.status.containerStatuses[0].containerID}'
Subtask 3.4: Test Pod Functionality and Network Communication
Execute commands inside the Pod:
kubectl exec nginx-demo -- nginx -v
Check Pod IP and network configuration:
kubectl get pod nginx-demo -o wide
Test network connectivity to the Pod:
POD_IP=$(kubectl get pod nginx-demo -o jsonpath='{.status.podIP}')
curl -I http://$POD_IP
Port-forward to test application accessibility:
kubectl port-forward nginx-demo 8080:80 &
curl http://localhost:8080
# Stop port-forward
pkill -f "kubectl port-forward"
Task 4: Advanced Component Analysis
Subtask 4.1: Examine Resource Usage and Metrics
Check node resource usage:
kubectl top nodes
Check Pod resource usage:
kubectl top pods
Get detailed resource information:
kubectl describe node | grep -A 5 "Allocated resources"
Subtask 4.2: Understand Component Dependencies
Check service accounts and RBAC:
kubectl get serviceaccounts -n kube-system
Examine cluster roles and bindings:
kubectl get clusterroles | head -10
kubectl get clusterrolebindings | head -10
Check component health endpoints:
kubectl get componentstatuses
Troubleshooting Common Issues
Issue 1: Pod Stuck in Pending State
If your Pod remains in Pending state:

Check Pod events:
kubectl describe pod nginx-demo | grep -A 10 Events
Verify node resources:
kubectl describe nodes | grep -A 5 "Allocated resources"
Check scheduler logs:
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep scheduler | awk '{print $1}') | tail -20
Issue 2: Cannot Access Control Plane Components
If you cannot access control plane logs:

Verify cluster status:
kubectl cluster-info
Check if components are running:
kubectl get pods -n kube-system
Restart minikube if necessary:
minikube stop
minikube start
Issue 3: Network Connectivity Issues
If Pod networking doesn't work:

Check Pod network configuration:
kubectl get pod nginx-demo -o yaml | grep -A 5 "podIP"
Verify DNS resolution:
kubectl exec nginx-demo -- nslookup kubernetes.default
Cleanup
Remove the resources created during this lab:

kubectl delete pod nginx-demo
rm nginx-pod.yaml
Conclusion
In this lab, you have successfully:

• Identified Kubernetes cluster components using kubectl commands and learned how to inspect their status and configuration • Examined control plane component logs including API Server and etcd, understanding how to troubleshoot cluster issues through log analysis • Created a Pod and analyzed its interaction with control plane and worker nodes, observing the complete lifecycle from creation to running state • Understood component communication flow and how different parts of Kubernetes work together to manage containerized applications

Why This Matters: Understanding Kubernetes architecture is crucial for:

Troubleshooting cluster issues effectively by knowing which component logs to check
Optimizing cluster performance by understanding resource allocation and component interactions
Securing your cluster by knowing the role of each component and their communication patterns
Preparing for KCNA certification by demonstrating practical knowledge of Kubernetes internals
This knowledge forms the foundation for advanced Kubernetes operations, including cluster administration, performance tuning, and security hardening. The hands-on experience with kubectl commands and log analysis will be invaluable as you progress to more complex Kubernetes scenarios.
