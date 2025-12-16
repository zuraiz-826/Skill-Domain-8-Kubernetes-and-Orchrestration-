Lab 15: Implementing Autoscaling in Kubernetes
Objectives
By the end of this lab, you will be able to:

• Understand the concepts of Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA) • Configure and deploy a Horizontal Pod Autoscaler for automatic scaling based on CPU utilization • Generate synthetic traffic to trigger autoscaling behavior • Monitor and observe scaling events in real-time • Implement Vertical Pod Autoscaler to automatically adjust resource requests and limits • Analyze the differences between horizontal and vertical scaling strategies • Troubleshoot common autoscaling issues and optimize performance

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with kubectl command-line tool • Knowledge of YAML configuration files • Understanding of CPU and memory resource concepts • Basic Linux command-line skills

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes cluster already set up. Simply click Start Lab to access your environment. No need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • A running Kubernetes cluster with multiple nodes • kubectl configured and ready to use • Metrics server pre-installed for resource monitoring • All necessary tools for generating load and monitoring

Task 1: Configure Horizontal Pod Autoscaler (HPA)
Subtask 1.1: Create a Sample Application Deployment
First, we'll create a simple web application that we can scale automatically.

Create a deployment manifest file:
cat > php-apache-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
            memory: 128Mi
          requests:
            cpu: 200m
            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: php-apache
  type: ClusterIP
EOF
Apply the deployment:
kubectl apply -f php-apache-deployment.yaml
Verify the deployment is running:
kubectl get deployments
kubectl get pods -l app=php-apache
Subtask 1.2: Verify Metrics Server Installation
The HPA requires metrics server to function properly. Let's verify it's running:

Check if metrics server is installed:
kubectl get deployment metrics-server -n kube-system
If metrics server is not running, install it:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Wait for metrics server to be ready:
kubectl wait --for=condition=available --timeout=300s deployment/metrics-server -n kube-system
Verify metrics are available:
kubectl top nodes
kubectl top pods
Subtask 1.3: Create Horizontal Pod Autoscaler
Now we'll create an HPA that scales based on CPU utilization.

Create the HPA using kubectl command:
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
Alternatively, create an HPA using YAML manifest:
cat > hpa-config.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
EOF
Apply the HPA configuration:
kubectl apply -f hpa-config.yaml
Verify the HPA is created:
kubectl get hpa
kubectl describe hpa php-apache
Task 2: Generate Traffic to Observe Scaling Behavior
Subtask 2.1: Create Load Generator Pod
We'll create a pod that generates continuous traffic to trigger autoscaling.

Create a load generator pod:
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh
Inside the load generator pod, run the following command to generate load:
while true; do wget -q -O- http://php-apache; done
Subtask 2.2: Monitor Scaling in Real-Time
Open a new terminal window and monitor the scaling behavior:

Watch HPA status in real-time:
kubectl get hpa php-apache --watch
In another terminal, monitor pod scaling:
kubectl get pods -l app=php-apache --watch
Monitor resource usage:
watch kubectl top pods -l app=php-apache
Subtask 2.3: Observe Scaling Events
Check HPA events to see scaling decisions:
kubectl describe hpa php-apache
View deployment events:
kubectl describe deployment php-apache
Check cluster events:
kubectl get events --sort-by=.metadata.creationTimestamp
Subtask 2.4: Test Scale-Down Behavior
Stop the load generator by pressing Ctrl+C in the load generator terminal.

Monitor the scale-down process:

kubectl get hpa php-apache --watch
Observe how pods are terminated:
kubectl get pods -l app=php-apache --watch
Task 3: Experiment with Vertical Pod Autoscaler (VPA)
Subtask 3.1: Install Vertical Pod Autoscaler
VPA is not installed by default, so we need to install it first.

Clone the VPA repository:
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
Install VPA components:
./hack/vpa-install.sh
Verify VPA installation:
kubectl get pods -n kube-system | grep vpa
Subtask 3.2: Create a VPA Configuration
First, remove the existing HPA to avoid conflicts:
kubectl delete hpa php-apache
Create a VPA configuration:
cat > vpa-config.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: php-apache-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: php-apache
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1000m
        memory: 500Mi
      controlledResources: ["cpu", "memory"]
EOF
Apply the VPA configuration:
kubectl apply -f vpa-config.yaml
Verify VPA is created:
kubectl get vpa
kubectl describe vpa php-apache-vpa
Subtask 3.3: Generate Load for VPA Testing
Create a more intensive load generator:
cat > load-generator-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 3
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: load-generator
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true; do wget -q -O- http://php-apache; sleep 0.1; done"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
EOF
Deploy the load generator:
kubectl apply -f load-generator-deployment.yaml
Subtask 3.4: Monitor VPA Recommendations
Monitor VPA recommendations:
kubectl describe vpa php-apache-vpa
Check the current resource usage:
kubectl top pods -l app=php-apache
Watch for pod restarts as VPA applies new resource limits:
kubectl get pods -l app=php-apache --watch
Compare resource requests before and after VPA adjustment:
kubectl describe pod -l app=php-apache
Subtask 3.5: Test VPA in Recommendation Mode
Create a VPA in recommendation-only mode:
cat > vpa-recommend-only.yaml << 'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: php-apache-vpa-recommend
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: php-apache
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1000m
        memory: 500Mi
EOF
Apply the recommendation-only VPA:
kubectl delete vpa php-apache-vpa
kubectl apply -f vpa-recommend-only.yaml
View recommendations without automatic updates:
kubectl describe vpa php-apache-vpa-recommend
Task 4: Advanced Autoscaling Scenarios
Subtask 4.1: Multi-Metric HPA
Create an HPA that scales based on multiple metrics:

cat > multi-metric-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-multi-metric
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
EOF
Subtask 4.2: Custom Metrics HPA
For advanced scenarios, you can also scale based on custom metrics:

cat > custom-metric-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1k"
EOF
Troubleshooting Common Issues
Issue 1: HPA Shows "Unknown" Status
Problem: HPA status shows "Unknown" for current metrics.

Solution:

Check if metrics server is running:
kubectl get pods -n kube-system | grep metrics-server
Verify pod resource requests are set:
kubectl describe deployment php-apache
Check metrics server logs:
kubectl logs -n kube-system deployment/metrics-server
Issue 2: VPA Not Updating Resources
Problem: VPA recommendations are generated but pods are not updated.

Solution:

Ensure VPA is in "Auto" mode:
kubectl describe vpa php-apache-vpa
Check VPA admission controller:
kubectl get pods -n kube-system | grep vpa-admission-controller
Verify no resource quotas are blocking updates:
kubectl describe resourcequota
Issue 3: Scaling Too Aggressive or Too Slow
Problem: Autoscaling behavior is not optimal.

Solution:

Adjust scaling policies in HPA behavior section
Modify stabilization windows
Fine-tune target utilization percentages
Cleanup
Clean up the resources created in this lab:

# Delete deployments
kubectl delete deployment php-apache load-generator

# Delete services
kubectl delete service php-apache

# Delete HPA
kubectl delete hpa --all

# Delete VPA
kubectl delete vpa --all

# Delete configuration files
rm -f php-apache-deployment.yaml hpa-config.yaml vpa-config.yaml load-generator-deployment.yaml
rm -f multi-metric-hpa.yaml custom-metric-hpa.yaml vpa-recommend-only.yaml

# Clean up VPA installation (optional)
cd autoscaler/vertical-pod-autoscaler/
./hack/vpa-down.sh
cd ../../../
rm -rf autoscaler/
Conclusion
In this comprehensive lab, you have successfully:

• Implemented Horizontal Pod Autoscaler (HPA) to automatically scale applications based on CPU utilization, learning how Kubernetes can dynamically adjust the number of pod replicas to handle varying workloads

• Generated synthetic traffic and observed real-time scaling behavior, understanding how HPA responds to load changes and the importance of proper resource requests and limits

• Configured Vertical Pod Autoscaler (VPA) to automatically adjust resource requests and limits, learning the difference between horizontal scaling (more pods) and vertical scaling (bigger pods)

• Explored advanced autoscaling scenarios including multi-metric scaling and custom metrics, preparing you for complex production environments

• Gained hands-on experience with monitoring tools and troubleshooting techniques essential for managing autoscaling in production Kubernetes clusters

This knowledge is crucial for the Kubernetes and Cloud Native Associate (KCNA) certification and real-world Kubernetes operations. Autoscaling is a fundamental capability that enables applications to handle varying loads efficiently while optimizing resource utilization and costs. Understanding both HPA and VPA allows you to choose the right scaling strategy for different application patterns and requirements.

The skills you've developed in this lab will help you design resilient, cost-effective Kubernetes applications that can automatically adapt to changing demands, making you a more effective cloud-native engineer
