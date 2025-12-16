Lab 3: Workloads and Scheduling Lab
Objectives
By the end of this lab, students will be able to:

• Deploy applications using Kubernetes Deployment resources • Configure horizontal scaling using kubectl scale command • Set up and configure Horizontal Pod Autoscaler (HPA) for automatic scaling based on CPU usage • Perform rolling updates on deployed applications • Simulate and execute rollback scenarios for application deployments • Understand the relationship between workloads, scheduling, and resource management in Kubernetes

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with command-line interface operations • Basic knowledge of YAML file structure • Understanding of containerization concepts • Previous experience with kubectl commands

Required Tools: • kubectl (Kubernetes command-line tool) • A running Kubernetes cluster • Metrics server (for HPA functionality)

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu Linux machine with kubectl configured • Single-node Kubernetes cluster (minikube) • Metrics server pre-installed • All necessary permissions configured

Task 1: Deploy an Application Using Deployment Resource and Configure Horizontal Scaling
Subtask 1.1: Create a Sample Application Deployment
First, we'll create a deployment for a simple web application that we can scale and monitor.

Create the deployment YAML file:
cat > nginx-deployment.yaml << EOF
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
EOF
Apply the deployment:
kubectl apply -f nginx-deployment.yaml
Verify the deployment:
kubectl get deployments
kubectl get pods -l app=nginx
Expected output should show 3 running pods with the nginx label.

Subtask 1.2: Create a Service to Expose the Application
Create a service YAML file:
cat > nginx-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
EOF
Apply the service:
kubectl apply -f nginx-service.yaml
Verify the service:
kubectl get services
kubectl describe service nginx-service
Subtask 1.3: Configure Manual Horizontal Scaling
Scale up the deployment to 5 replicas:
kubectl scale deployment nginx-deployment --replicas=5
Verify the scaling operation:
kubectl get deployments
kubectl get pods -l app=nginx
You should now see 5 pods running.

Scale down the deployment to 2 replicas:
kubectl scale deployment nginx-deployment --replicas=2
Monitor the scaling process:
kubectl get pods -l app=nginx -w
Press Ctrl+C to stop watching after observing the pods being terminated.

Task 2: Set Up Horizontal Pod Autoscaler (HPA) Based on CPU Usage
Subtask 2.1: Verify Metrics Server Installation
Check if metrics server is running:
kubectl get pods -n kube-system | grep metrics-server
If metrics server is not running, install it:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Wait for metrics server to be ready:
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=60s
Subtask 2.2: Create a CPU-Intensive Application for Testing
Create a CPU-intensive deployment:
cat > cpu-demo-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-demo
  template:
    metadata:
      labels:
        app: cpu-demo
    spec:
      containers:
      - name: cpu-demo
        image: busybox
        command: ["sh", "-c", "while true; do echo 'CPU intensive task'; dd if=/dev/zero of=/dev/null bs=1M count=100; sleep 1; done"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 128Mi
EOF
Apply the CPU demo deployment:
kubectl apply -f cpu-demo-deployment.yaml
Verify the deployment:
kubectl get pods -l app=cpu-demo
Subtask 2.3: Configure Horizontal Pod Autoscaler
Create HPA for the CPU demo application:
kubectl autoscale deployment cpu-demo --cpu-percent=50 --min=1 --max=10
Verify HPA creation:
kubectl get hpa
Create HPA using YAML for more control:
cat > cpu-demo-hpa.yaml << EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-demo-hpa-v2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-demo
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30
EOF
Apply the advanced HPA:
kubectl apply -f cpu-demo-hpa.yaml
Subtask 2.4: Test Autoscaling Behavior
Monitor HPA status:
kubectl get hpa -w
Keep this running in one terminal window.

In a new terminal, generate CPU load:
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh
Inside the load generator pod, run:
while true; do wget -q -O- http://cpu-demo-service/; done
Create a service for the CPU demo first (in another terminal):
kubectl expose deployment cpu-demo --port=80 --target-port=8080 --name=cpu-demo-service
Watch the scaling in action:
kubectl get pods -l app=cpu-demo -w
Stop the load generator by pressing Ctrl+C and exit the pod.

Observe scale-down behavior:

kubectl get hpa
kubectl get pods -l app=cpu-demo
Task 3: Perform Rolling Update and Simulate Rollback Scenario
Subtask 3.1: Prepare Application for Rolling Updates
Create a more comprehensive deployment for update testing:
cat > webapp-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
EOF
Apply the webapp deployment:
kubectl apply -f webapp-deployment.yaml
Create a service for the webapp:
kubectl expose deployment webapp --port=80 --type=ClusterIP
Verify initial deployment:
kubectl get deployments webapp
kubectl get pods -l app=webapp
kubectl rollout status deployment/webapp
Subtask 3.2: Perform Rolling Update
Check current image version:
kubectl describe deployment webapp | grep Image
Update the deployment to use a newer nginx version:
kubectl set image deployment/webapp webapp=nginx:1.21
Monitor the rolling update process:
kubectl rollout status deployment/webapp
Watch pods during the update:
kubectl get pods -l app=webapp -w
Press Ctrl+C after observing the rolling update process.

Verify the update completed successfully:
kubectl describe deployment webapp | grep Image
kubectl get pods -l app=webapp
Subtask 3.3: View Rollout History
Check rollout history:
kubectl rollout history deployment/webapp
Get detailed information about a specific revision:
kubectl rollout history deployment/webapp --revision=1
kubectl rollout history deployment/webapp --revision=2
Subtask 3.4: Simulate Rollback Scenario
Perform another update with a problematic image:
kubectl set image deployment/webapp webapp=nginx:1.22-invalid
Monitor the failed update:
kubectl rollout status deployment/webapp --timeout=60s
Check the status of pods:
kubectl get pods -l app=webapp
kubectl describe pods -l app=webapp | grep -A 5 Events
Rollback to the previous working version:
kubectl rollout undo deployment/webapp
Monitor the rollback process:
kubectl rollout status deployment/webapp
Verify rollback success:
kubectl get pods -l app=webapp
kubectl describe deployment webapp | grep Image
Rollback to a specific revision:
kubectl rollout undo deployment/webapp --to-revision=1
Verify the specific rollback:
kubectl rollout status deployment/webapp
kubectl describe deployment webapp | grep Image
Task 4: Advanced Workload Management
Subtask 4.1: Configure Resource Quotas and Limits
Create a namespace for resource management testing:
kubectl create namespace resource-demo
Create a resource quota:
cat > resource-quota.yaml << EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: resource-demo
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "10"
EOF
Apply the resource quota:
kubectl apply -f resource-quota.yaml
Verify the quota:
kubectl describe quota compute-quota -n resource-demo
Subtask 4.2: Test Resource Constraints
Create a deployment that respects resource limits:
cat > resource-limited-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-limited-app
  namespace: resource-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-limited-app
  template:
    metadata:
      labels:
        app: resource-limited-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
EOF
Apply the deployment:
kubectl apply -f resource-limited-deployment.yaml
Check resource usage against quota:
kubectl describe quota compute-quota -n resource-demo
Cleanup and Verification
Subtask 5.1: Clean Up Resources
Delete all created deployments:
kubectl delete deployment nginx-deployment
kubectl delete deployment cpu-demo
kubectl delete deployment webapp
kubectl delete deployment resource-limited-app -n resource-demo
Delete services:
kubectl delete service nginx-service
kubectl delete service cpu-demo-service
kubectl delete service webapp
Delete HPA:
kubectl delete hpa cpu-demo
kubectl delete hpa cpu-demo-hpa-v2
Delete namespace:
kubectl delete namespace resource-demo
Verify cleanup:
kubectl get deployments
kubectl get services
kubectl get hpa
kubectl get namespaces
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Metrics Server Not Working

Symptom: HPA shows "unknown" for CPU metrics
Solution:
kubectl top nodes
kubectl top pods
If these commands fail, restart metrics server:

kubectl delete pods -n kube-system -l k8s-app=metrics-server
Issue 2: Rolling Update Stuck

Symptom: Rolling update doesn't progress
Solution: Check pod events and resource availability:
kubectl describe pods -l app=webapp
kubectl get events --sort-by=.metadata.creationTimestamp
Issue 3: HPA Not Scaling

Symptom: HPA doesn't trigger scaling despite high CPU
Solution: Verify resource requests are set:
kubectl describe deployment cpu-demo
Issue 4: Rollback Fails

Symptom: Rollback command doesn't work
Solution: Check rollout history and use specific revision:
kubectl rollout history deployment/webapp
kubectl rollout undo deployment/webapp --to-revision=1
Conclusion
In this comprehensive lab, you have successfully:

• Deployed applications using Kubernetes Deployment resources with proper resource specifications • Configured manual horizontal scaling using kubectl scale commands to adjust replica counts • Implemented automatic scaling with Horizontal Pod Autoscaler based on CPU utilization metrics • Performed rolling updates to deploy new application versions without downtime • Executed rollback scenarios to recover from failed deployments • Managed resource constraints using quotas and limits to control cluster resource usage

Key Takeaways
Workload Management: You learned how Kubernetes manages application workloads through Deployments, which provide declarative updates and scaling capabilities.

Scheduling Intelligence: The lab demonstrated how Kubernetes scheduler works with resource requests and limits to place pods efficiently across cluster nodes.

Operational Excellence: Through rolling updates and rollbacks, you experienced how Kubernetes enables zero-downtime deployments and quick recovery from issues.

Resource Optimization: HPA and resource quotas showed how to automatically optimize resource usage while maintaining application performance.

Real-World Applications
These skills are essential for: • Production deployments where high availability is critical • Cost optimization through efficient resource utilization • DevOps practices enabling continuous deployment with safety mechanisms • Scalable architectures that respond automatically to demand changes

The knowledge gained in this lab directly applies to the Certified Kubernetes Administrator (CKA) certification and real-world Kubernetes operations in enterprise environments.
