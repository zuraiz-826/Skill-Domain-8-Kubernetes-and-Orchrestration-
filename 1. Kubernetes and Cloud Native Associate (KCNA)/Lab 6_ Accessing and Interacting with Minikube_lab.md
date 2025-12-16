Lab 6: Accessing and Interacting with Minikube
Objectives
By the end of this lab, you will be able to:

• Understand the basic architecture and components of a Kubernetes cluster using Minikube • Use kubectl command-line tool to interact with Kubernetes resources • List and examine nodes, namespaces, and pods in a Kubernetes cluster • Deploy a simple Pod and manage its lifecycle • Retrieve and analyze Pod logs for troubleshooting • Diagnose and resolve common connectivity issues within a Kubernetes cluster • Apply fundamental Kubernetes concepts in a hands-on environment

Prerequisites
Before starting this lab, you should have:

• Basic understanding of containerization concepts (Docker) • Familiarity with Linux command-line interface • Basic knowledge of YAML file structure • Understanding of networking fundamentals • No prior Kubernetes experience required - this lab will guide you through the basics

Lab Environment Setup
Good News! Al Nafi provides ready-to-use Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install software.

Your cloud machine comes with: • Minikube pre-installed and configured • kubectl command-line tool ready to use • Docker runtime environment • All necessary dependencies configured

Task 1: Understanding Your Kubernetes Environment
Subtask 1.1: Start Minikube and Verify Cluster Status
First, let's start your Minikube cluster and verify it's running properly.

Start Minikube cluster:
minikube start --driver=docker
Check Minikube status:
minikube status
Expected output should show:

minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
Verify kubectl is configured to communicate with your cluster:
kubectl cluster-info
Subtask 1.2: List and Examine Nodes
Now let's explore the nodes in your cluster.

List all nodes in the cluster:
kubectl get nodes
Get detailed information about nodes:
kubectl get nodes -o wide
Describe a specific node (replace 'minikube' with your node name if different):
kubectl describe node minikube
Key Concept: In Minikube, you typically have one node that acts as both master and worker node, making it perfect for learning and development.

Subtask 1.3: Explore Namespaces
Namespaces provide a way to organize resources in a cluster.

List all namespaces:
kubectl get namespaces
Get detailed information about namespaces:
kubectl get namespaces -o wide
Describe the default namespace:
kubectl describe namespace default
List namespaces with additional details:
kubectl get ns --show-labels
Key Concept: Namespaces are like folders that help organize your Kubernetes resources. Common namespaces include default, kube-system, and kube-public.

Subtask 1.4: List and Examine Pods
Let's explore the pods running in your cluster.

List pods in the default namespace:
kubectl get pods
List pods in all namespaces:
kubectl get pods --all-namespaces
List pods with additional information:
kubectl get pods -o wide --all-namespaces
Focus on system pods:
kubectl get pods -n kube-system
Task 2: Deploy a Simple Pod and Manage Its Lifecycle
Subtask 2.1: Create a Simple Pod
Let's deploy a simple nginx pod to practice pod management.

Create a simple pod using kubectl run command:
kubectl run my-nginx-pod --image=nginx:latest --port=80
Verify the pod was created:
kubectl get pods
Get detailed information about your pod:
kubectl get pod my-nginx-pod -o wide
Describe the pod to see detailed information:
kubectl describe pod my-nginx-pod
Subtask 2.2: Create a Pod Using YAML Manifest
Now let's create a more complex pod using a YAML file.

Create a YAML file for a pod:
cat > test-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-app-pod
  labels:
    app: test-app
    environment: lab
spec:
  containers:
  - name: test-container
    image: busybox:latest
    command: ['sh', '-c', 'echo "Hello from Kubernetes Pod!" && sleep 3600']
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
Apply the YAML file to create the pod:
kubectl apply -f test-pod.yaml
Verify both pods are running:
kubectl get pods
Subtask 2.3: Retrieve and Analyze Pod Logs
Understanding how to access logs is crucial for troubleshooting.

Get logs from the nginx pod:
kubectl logs my-nginx-pod
Get logs from the test-app-pod:
kubectl logs test-app-pod
Follow logs in real-time (use Ctrl+C to stop):
kubectl logs -f test-app-pod
Get logs with timestamps:
kubectl logs test-app-pod --timestamps
Get the last 10 lines of logs:
kubectl logs test-app-pod --tail=10
Subtask 2.4: Execute Commands Inside Pods
Sometimes you need to troubleshoot by executing commands inside running pods.

Execute a command in the busybox pod:
kubectl exec test-app-pod -- ls -la
Get an interactive shell in the pod:
kubectl exec -it test-app-pod -- sh
Inside the pod shell, try these commands:

# Check the hostname
hostname

# Check network configuration
ip addr

# Check running processes
ps aux

# Exit the pod shell
exit
Task 3: Diagnose and Resolve Connectivity Issues
Subtask 3.1: Create a Service for Pod Access
Let's create a service to expose our nginx pod and then diagnose connectivity.

Expose the nginx pod as a service:
kubectl expose pod my-nginx-pod --port=80 --target-port=80 --name=nginx-service
List services:
kubectl get services
Describe the service:
kubectl describe service nginx-service
Subtask 3.2: Test Connectivity Between Pods
Let's test network connectivity between pods.

Create a debug pod for network testing:
kubectl run debug-pod --image=busybox:latest --rm -it --restart=Never -- sh
Inside the debug pod, test connectivity:

# Test DNS resolution
nslookup nginx-service

# Test HTTP connectivity to the service
wget -qO- nginx-service

# Test connectivity to the pod directly (replace IP with actual pod IP)
# First, get the pod IP from another terminal: kubectl get pod my-nginx-pod -o wide
wget -qO- <POD_IP>

# Exit the debug pod
exit
Subtask 3.3: Simulate and Resolve a Connectivity Issue
Let's create a problematic pod and learn to diagnose issues.

Create a pod with a wrong image name:
kubectl run broken-pod --image=nginx:nonexistent-tag
Check the pod status:
kubectl get pods
Describe the broken pod to see the issue:
kubectl describe pod broken-pod
Check events to understand what went wrong:
kubectl get events --sort-by=.metadata.creationTimestamp
Fix the broken pod by deleting and recreating it:
kubectl delete pod broken-pod
kubectl run fixed-pod --image=nginx:latest
Verify the fix:
kubectl get pods
kubectl describe pod fixed-pod
Subtask 3.4: Advanced Troubleshooting Techniques
Let's explore more troubleshooting commands.

Check resource usage:
kubectl top nodes
kubectl top pods
Get detailed cluster information:
kubectl get all
Check pod resource specifications:
kubectl get pods -o yaml my-nginx-pod
Monitor pod status in real-time:
kubectl get pods -w
Press Ctrl+C to stop watching.

Subtask 3.5: Network Policy Testing (Optional Advanced Section)
For advanced users, let's test network policies.

Create a network policy that blocks traffic:
cat > deny-all-policy.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
Apply the network policy:
kubectl apply -f deny-all-policy.yaml
Test connectivity (should fail):
kubectl run test-connectivity --image=busybox:latest --rm -it --restart=Never -- wget -qO- nginx-service
Remove the network policy to restore connectivity:
kubectl delete networkpolicy deny-all
Test connectivity again (should work):
kubectl run test-connectivity --image=busybox:latest --rm -it --restart=Never -- wget -qO- nginx-service
Task 4: Clean Up Resources
Subtask 4.1: Remove Created Resources
Let's clean up the resources we created during this lab.

Delete pods:
kubectl delete pod my-nginx-pod
kubectl delete pod test-app-pod
kubectl delete pod fixed-pod
Delete services:
kubectl delete service nginx-service
Delete YAML files:
rm test-pod.yaml deny-all-policy.yaml
Verify cleanup:
kubectl get pods
kubectl get services
Subtask 4.2: Stop Minikube (Optional)
If you want to stop your Minikube cluster:

minikube stop
To start it again later:

minikube start
Troubleshooting Common Issues
Issue 1: Pod Stuck in Pending State
Symptoms: Pod shows status as "Pending"

Diagnosis:

kubectl describe pod <pod-name>
kubectl get events
Common Causes: • Insufficient resources • Image pull issues • Scheduling constraints

Issue 2: Cannot Connect to Service
Symptoms: Connection timeouts or refused connections

Diagnosis:

kubectl get endpoints <service-name>
kubectl describe service <service-name>
Common Causes: • Service selector doesn't match pod labels • Wrong port configuration • Pod not ready

Issue 3: Image Pull Errors
Symptoms: Pod shows "ImagePullBackOff" or "ErrImagePull"

Diagnosis:

kubectl describe pod <pod-name>
Common Causes: • Incorrect image name or tag • Network connectivity issues • Authentication problems with private registries

Key Commands Reference
Essential kubectl Commands
# Cluster information
kubectl cluster-info
kubectl get nodes
kubectl get namespaces

# Pod management
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- <command>

# Service management
kubectl get services
kubectl describe service <service-name>
kubectl expose pod <pod-name> --port=<port>

# Troubleshooting
kubectl get events
kubectl top nodes
kubectl top pods
Conclusion
Congratulations! You have successfully completed Lab 6: Accessing and Interacting with Minikube. In this lab, you have accomplished the following:

What You Learned: • How to start and manage a Minikube cluster • Essential kubectl commands for cluster interaction • How to list and examine nodes, namespaces, and pods • Pod deployment using both command-line and YAML manifests • Log retrieval and analysis techniques • Network connectivity testing and troubleshooting • Common issue diagnosis and resolution strategies

Why This Matters: These skills form the foundation of Kubernetes administration and are essential for the Kubernetes and Cloud Native Associate (KCNA) certification. Understanding how to interact with Kubernetes clusters, deploy applications, and troubleshoot issues is crucial for anyone working with containerized applications in production environments.

Next Steps: • Practice these commands regularly to build muscle memory • Experiment with different pod configurations • Explore more complex Kubernetes resources like Deployments and Services • Study networking concepts in Kubernetes • Practice troubleshooting scenarios to prepare for real-world situations

The hands-on experience you gained in this lab provides a solid foundation for more advanced Kubernetes topics and prepares you for cloud-native application development and deployment.
