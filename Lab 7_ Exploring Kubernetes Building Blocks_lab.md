Lab 7: Exploring Kubernetes Building Blocks
Objectives
By the end of this lab, you will be able to:

• Deploy a Pod using a YAML manifest file • Create and manage Deployments to scale applications • Understand the difference between Pods and Deployments • Create ConfigMaps to manage application configuration • Mount ConfigMaps into Pods as environment variables • Verify and troubleshoot Kubernetes resources using kubectl commands • Apply best practices for container orchestration with Kubernetes

Prerequisites
Before starting this lab, you should have:

• Basic understanding of containerization concepts (Docker) • Familiarity with YAML file structure and syntax • Basic knowledge of Linux command line operations • Understanding of environment variables and configuration management • Completion of previous Kubernetes fundamentals labs or equivalent knowledge

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed and configured. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes manually.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-installed • Minikube cluster ready for use • Text editor (nano/vim) for creating YAML files • All necessary permissions configured

Task 1: Deploy a Pod Using a YAML Manifest
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure your Kubernetes cluster is running properly.

Open your terminal in the lab environment

Check the cluster status:

kubectl cluster-info
Verify that nodes are ready:
kubectl get nodes
You should see output similar to:

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1d    v1.28.0
Subtask 1.2: Create a Pod YAML Manifest
Now we'll create a simple Pod using a YAML manifest file.

Create a new directory for your lab files:
mkdir ~/k8s-lab7
cd ~/k8s-lab7
Create a Pod manifest file:
nano simple-pod.yaml
Add the following content to the file:
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: lab
spec:
  containers:
  - name: nginx-container
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
Save and exit the file (Ctrl+X, then Y, then Enter in nano)
Subtask 1.3: Deploy the Pod
Apply the Pod manifest:
kubectl apply -f simple-pod.yaml
Verify the Pod was created:
kubectl get pods
Get detailed information about the Pod:
kubectl describe pod nginx-pod
Check the Pod logs:
kubectl logs nginx-pod
Subtask 1.4: Test Pod Connectivity
Get the Pod's IP address:
kubectl get pod nginx-pod -o wide
Test connectivity to the Pod (from within the cluster):
kubectl exec -it nginx-pod -- curl localhost:80
You should see the default nginx welcome page HTML.

Task 2: Scale the Application Using a Deployment
Subtask 2.1: Create a Deployment YAML Manifest
Deployments provide better management capabilities than standalone Pods, including scaling and rolling updates.

Create a Deployment manifest file:
nano nginx-deployment.yaml
Add the following content:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
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
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
Save and exit the file
Subtask 2.2: Deploy the Application
Apply the Deployment manifest:
kubectl apply -f nginx-deployment.yaml
Verify the Deployment was created:
kubectl get deployments
Check the Pods created by the Deployment:
kubectl get pods -l app=nginx
You should see 2 nginx Pods running.

Get detailed information about the Deployment:
kubectl describe deployment nginx-deployment
Subtask 2.3: Scale the Deployment
Now let's demonstrate scaling capabilities.

Scale the Deployment to 4 replicas using kubectl:
kubectl scale deployment nginx-deployment --replicas=4
Verify the scaling operation:
kubectl get pods -l app=nginx
You should now see 4 nginx Pods.

Check the Deployment status:
kubectl get deployment nginx-deployment
Subtask 2.4: Scale Using YAML Manifest
You can also scale by modifying the YAML file.

Edit the Deployment file:
nano nginx-deployment.yaml
Change the replicas value from 2 to 3:
spec:
  replicas: 3
Apply the updated manifest:
kubectl apply -f nginx-deployment.yaml
Verify the change:
kubectl get pods -l app=nginx
The number of Pods should now be 3.

Task 3: Create a ConfigMap and Mount it into a Pod
Subtask 3.1: Create a ConfigMap YAML Manifest
ConfigMaps allow you to separate configuration from your application code.

Create a ConfigMap manifest file:
nano app-config.yaml
Add the following content:
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_host: "mysql.example.com"
  database_port: "3306"
  database_name: "myapp"
  log_level: "INFO"
  max_connections: "100"
  app_version: "1.2.3"
Save and exit the file
Subtask 3.2: Create the ConfigMap
Apply the ConfigMap manifest:
kubectl apply -f app-config.yaml
Verify the ConfigMap was created:
kubectl get configmaps
View the ConfigMap details:
kubectl describe configmap app-config
Subtask 3.3: Create a Pod that Uses the ConfigMap
Now we'll create a Pod that uses the ConfigMap as environment variables.

Create a new Pod manifest that uses the ConfigMap:
nano pod-with-config.yaml
Add the following content:
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  labels:
    app: myapp
spec:
  containers:
  - name: app-container
    image: busybox:1.35
    command: ['sh', '-c', 'echo "Starting application..." && env | grep -E "(DATABASE|LOG|MAX|APP)" && sleep 3600']
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
Save and exit the file
Subtask 3.4: Deploy and Test the Pod with ConfigMap
Apply the Pod manifest:
kubectl apply -f pod-with-config.yaml
Verify the Pod is running:
kubectl get pod app-pod
Check the Pod logs to see the environment variables:
kubectl logs app-pod
You should see the configuration values from the ConfigMap displayed as environment variables.

Execute into the Pod to explore the environment:
kubectl exec -it app-pod -- sh
Inside the Pod, check all environment variables:
env | sort
Check specific configuration variables:
echo "Database Host: $database_host"
echo "Database Port: $database_port"
echo "Log Level: $log_level"
Exit the Pod:
exit
Subtask 3.5: Update ConfigMap and Observe Changes
Update the ConfigMap:
kubectl patch configmap app-config --patch '{"data":{"log_level":"DEBUG","app_version":"1.2.4"}}'
Verify the ConfigMap was updated:
kubectl get configmap app-config -o yaml
Delete and recreate the Pod to see the updated configuration:
kubectl delete pod app-pod
kubectl apply -f pod-with-config.yaml
Check the logs of the new Pod:
kubectl logs app-pod
You should see the updated log_level as "DEBUG" and app_version as "1.2.4".

Verification and Cleanup
Verification Steps
List all resources created in this lab:
kubectl get pods,deployments,configmaps
Verify the Deployment is managing the correct number of replicas:
kubectl get deployment nginx-deployment -o wide
Confirm the ConfigMap is properly mounted:
kubectl exec app-pod -- env | grep database_host
Cleanup Resources
When you're finished with the lab, clean up the resources:

Delete the standalone Pod:
kubectl delete pod nginx-pod
Delete the Deployment:
kubectl delete deployment nginx-deployment
Delete the Pod with ConfigMap:
kubectl delete pod app-pod
Delete the ConfigMap:
kubectl delete configmap app-config
Verify all resources are deleted:
kubectl get pods,deployments,configmaps
Troubleshooting Tips
Common Issues and Solutions
Pod Stuck in Pending State:

Check node resources: kubectl describe nodes
Verify image availability: kubectl describe pod <pod-name>
ConfigMap Not Loading:

Ensure ConfigMap exists: kubectl get configmap
Check Pod specification for correct ConfigMap name
Verify YAML indentation and syntax
Deployment Not Scaling:

Check Deployment status: kubectl get deployment <deployment-name> -o wide
Review events: kubectl get events --sort-by=.metadata.creationTimestamp
Environment Variables Not Appearing:

Restart the Pod after ConfigMap changes
Use kubectl exec to check environment inside the container
Verify ConfigMap reference in Pod specification
Conclusion
In this lab, you have successfully explored the fundamental building blocks of Kubernetes:

Key Accomplishments: • Pod Management: You deployed a Pod using a YAML manifest, learning how to define container specifications, resource limits, and labels • Application Scaling: You created and managed a Deployment, demonstrating how Kubernetes can automatically maintain desired replica counts and enable easy scaling • Configuration Management: You implemented ConfigMaps to externalize application configuration and mounted them as environment variables in Pods

Why This Matters: These building blocks form the foundation of container orchestration in Kubernetes. Pods represent the smallest deployable units, Deployments provide reliability and scaling capabilities, and ConfigMaps enable configuration management best practices. Understanding these concepts is essential for:

• Building resilient, scalable applications in Kubernetes • Implementing proper separation of concerns between code and configuration • Preparing for production deployments and cloud-native application development • Advancing toward Kubernetes certification goals (KCNA)

Next Steps: With this foundation, you're ready to explore more advanced Kubernetes concepts such as Services for networking, Persistent Volumes for storage, and Ingress controllers for external access. These building blocks will serve as the basis for more complex orchestration scenarios in your Kubernetes journey.

