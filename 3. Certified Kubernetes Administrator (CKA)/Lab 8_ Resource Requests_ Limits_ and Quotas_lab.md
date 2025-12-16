Lab 8: Resource Requests, Limits, and Quotas
Objectives
By the end of this lab, you will be able to:

• Understand the difference between resource requests and limits in Kubernetes • Configure CPU and memory requests and limits for Pods • Create and apply namespace-level ResourceQuotas to control resource consumption • Deploy applications with resource constraints and verify quota compliance • Troubleshoot resource-related issues in Kubernetes environments • Monitor resource usage and quota utilization

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Namespaces, Deployments) • Familiarity with YAML syntax and Kubernetes manifest files • Experience with kubectl command-line tool • Knowledge of Linux command-line operations • Understanding of CPU and memory resource concepts

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes manually.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-configured • Single-node Kubernetes cluster (minikube or kind) • All necessary tools and permissions configured

Task 1: Understanding Resource Requests and Limits
Subtask 1.1: Explore Current Resource Usage
First, let's examine the current state of our cluster and understand resource concepts.

Check cluster nodes and their capacity:
kubectl get nodes
kubectl describe nodes
View current resource usage:
kubectl top nodes
kubectl top pods --all-namespaces
Note: If the metrics server is not available, you may see an error. This is normal in some lab environments.

Create a dedicated namespace for this lab:
kubectl create namespace resource-lab
kubectl config set-context --current --namespace=resource-lab
Subtask 1.2: Create a Pod with Resource Requests and Limits
Create a Pod manifest with resource specifications:
cat > pod-with-resources.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo-pod
  namespace: resource-lab
  labels:
    app: resource-demo
spec:
  containers:
  - name: demo-container
    image: nginx:1.21
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
    - containerPort: 80
EOF
Apply the Pod configuration:
kubectl apply -f pod-with-resources.yaml
Verify the Pod is running and check its resource allocation:
kubectl get pods -o wide
kubectl describe pod resource-demo-pod
Examine the resource section in the Pod description:
Look for the Requests and Limits sections in the output. Note how: • Requests: Guaranteed resources the Pod will receive • Limits: Maximum resources the Pod can consume

Task 2: Implementing Namespace-Level ResourceQuotas
Subtask 2.1: Create a ResourceQuota
Create a ResourceQuota manifest:
cat > resource-quota.yaml << 'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: resource-lab
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    persistentvolumeclaims: "4"
    pods: "10"
    services: "5"
EOF
Apply the ResourceQuota:
kubectl apply -f resource-quota.yaml
Verify the ResourceQuota is active:
kubectl get resourcequota -n resource-lab
kubectl describe resourcequota compute-quota -n resource-lab
Subtask 2.2: Test ResourceQuota Enforcement
Create a Pod that exceeds the quota:
cat > large-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: large-resource-pod
  namespace: resource-lab
spec:
  containers:
  - name: large-container
    image: nginx:1.21
    resources:
      requests:
        memory: "2Gi"
        cpu: "1500m"
      limits:
        memory: "3Gi"
        cpu: "2500m"
EOF
Attempt to create the Pod:
kubectl apply -f large-pod.yaml
Expected Result: You should see an error message indicating that the ResourceQuota would be exceeded.

Check the current quota usage:
kubectl describe resourcequota compute-quota -n resource-lab
Task 3: Deploy Applications with Resource Constraints
Subtask 3.1: Create a Deployment with Resource Specifications
Create a Deployment manifest with proper resource allocation:
cat > web-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: resource-lab
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-container
        image: nginx:1.21
        resources:
          requests:
            memory: "32Mi"
            cpu: "100m"
          limits:
            memory: "64Mi"
            cpu: "200m"
        ports:
        - containerPort: 80
EOF
Deploy the application:
kubectl apply -f web-deployment.yaml
Monitor the deployment progress:
kubectl get deployments -n resource-lab
kubectl get pods -n resource-lab
Check if all replicas are created successfully:
kubectl describe deployment web-app -n resource-lab
Subtask 3.2: Verify Quota Compliance
Check current resource usage against quotas:
kubectl describe resourcequota compute-quota -n resource-lab
Calculate total resource consumption:
The output should show: • Used vs Hard limits for each resource type • Current consumption by all Pods in the namespace

Attempt to scale the deployment beyond quota limits:
kubectl scale deployment web-app --replicas=8 -n resource-lab
Check the scaling result:
kubectl get deployments -n resource-lab
kubectl describe deployment web-app -n resource-lab
Note: Some replicas may not be created if they would exceed the ResourceQuota.

Task 4: Advanced Resource Management
Subtask 4.1: Create a LimitRange for Default Resource Values
Create a LimitRange to set default resource values:
cat > limit-range.yaml << 'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: resource-lab
spec:
  limits:
  - default:
      memory: "128Mi"
      cpu: "200m"
    defaultRequest:
      memory: "64Mi"
      cpu: "100m"
    type: Container
EOF
Apply the LimitRange:
kubectl apply -f limit-range.yaml
Verify the LimitRange is active:
kubectl get limitrange -n resource-lab
kubectl describe limitrange default-limits -n resource-lab
Subtask 4.2: Test Default Resource Assignment
Create a Pod without explicit resource specifications:
cat > pod-no-resources.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: default-resources-pod
  namespace: resource-lab
spec:
  containers:
  - name: default-container
    image: busybox:1.35
    command: ['sleep', '3600']
EOF
Deploy the Pod:
kubectl apply -f pod-no-resources.yaml
Check if default resources were applied:
kubectl describe pod default-resources-pod -n resource-lab
Expected Result: The Pod should have the default resource requests and limits applied automatically.

Task 5: Monitoring and Troubleshooting
Subtask 5.1: Monitor Resource Usage
Check overall namespace resource consumption:
kubectl top pods -n resource-lab
View detailed resource information for all Pods:
kubectl get pods -n resource-lab -o custom-columns=NAME:.metadata.name,CPU-REQUEST:.spec.containers[0].resources.requests.cpu,MEMORY-REQUEST:.spec.containers[0].resources.requests.memory,CPU-LIMIT:.spec.containers[0].resources.limits.cpu,MEMORY-LIMIT:.spec.containers[0].resources.limits.memory
Monitor quota usage over time:
watch -n 5 'kubectl describe resourcequota compute-quota -n resource-lab'
Press Ctrl+C to stop the watch command.

Subtask 5.2: Troubleshoot Resource Issues
Create a Pod that will be rejected due to resource constraints:
cat > problematic-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: problematic-pod
  namespace: resource-lab
spec:
  containers:
  - name: problem-container
    image: nginx:1.21
    resources:
      requests:
        memory: "1Gi"
        cpu: "800m"
      limits:
        memory: "1.5Gi"
        cpu: "1200m"
EOF
Attempt to create the Pod:
kubectl apply -f problematic-pod.yaml
Investigate the failure:
kubectl get events -n resource-lab --sort-by='.lastTimestamp'
kubectl describe resourcequota compute-quota -n resource-lab
Fix the issue by adjusting resources:
cat > fixed-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: fixed-pod
  namespace: resource-lab
spec:
  containers:
  - name: fixed-container
    image: nginx:1.21
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
Deploy the corrected Pod:
kubectl apply -f fixed-pod.yaml
kubectl get pods -n resource-lab
Task 6: Cleanup and Verification
Subtask 6.1: Review Final State
List all resources in the namespace:
kubectl get all -n resource-lab
kubectl get resourcequota,limitrange -n resource-lab
Check final quota usage:
kubectl describe resourcequota compute-quota -n resource-lab
Verify all Pods are running within constraints:
kubectl get pods -n resource-lab -o wide
Subtask 6.2: Clean Up Resources
Delete all Pods and Deployments:
kubectl delete deployment web-app -n resource-lab
kubectl delete pods --all -n resource-lab
Remove ResourceQuota and LimitRange:
kubectl delete resourcequota compute-quota -n resource-lab
kubectl delete limitrange default-limits -n resource-lab
Delete the namespace (optional):
kubectl delete namespace resource-lab
Common Troubleshooting Tips
Issue 1: Pod Creation Fails Due to ResourceQuota
Symptoms: Error message about exceeding ResourceQuota Solution: • Check current quota usage with kubectl describe resourcequota • Reduce resource requests/limits in Pod specification • Scale down existing deployments to free up quota

Issue 2: Pods Stuck in Pending State
Symptoms: Pods remain in Pending status Solution: • Check node resources with kubectl describe nodes • Verify resource requests don't exceed node capacity • Review events with kubectl get events

Issue 3: Metrics Server Not Available
Symptoms: kubectl top commands fail Solution: • This is normal in some lab environments • Use kubectl describe commands to view resource specifications • Focus on quota and limit configurations rather than real-time metrics

Key Concepts Summary
• Resource Requests: Guaranteed minimum resources for a container • Resource Limits: Maximum resources a container can consume • ResourceQuota: Namespace-level restrictions on resource usage • LimitRange: Default and maximum resource values for containers • CPU Units: Measured in millicores (m) where 1000m = 1 CPU core • Memory Units: Measured in bytes (Ki, Mi, Gi)

Conclusion
In this lab, you have successfully:

• Configured resource requests and limits for Pods to ensure proper resource allocation • Implemented namespace-level ResourceQuotas to control and limit resource consumption across multiple applications • Deployed applications with resource constraints and verified their compliance with established quotas • Created LimitRanges to provide default resource values for containers • Monitored and troubleshot resource-related issues in a Kubernetes environment

Understanding resource management is crucial for: • Cluster Stability: Preventing resource starvation and ensuring fair resource distribution • Cost Control: Optimizing resource usage and preventing over-provisioning • Application Performance: Guaranteeing sufficient resources for critical applications • Multi-tenancy: Safely running multiple applications in shared cluster environments

These skills are essential for the Certified Kubernetes Application Developer (CKAD) certification and real-world Kubernetes operations. Resource management ensures your applications run reliably while making efficient use of cluster resources, which is fundamental to successful Kubernetes deployments in production environments.
