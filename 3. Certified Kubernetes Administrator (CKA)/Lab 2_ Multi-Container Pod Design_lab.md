Lab 2: Multi-Container Pod Design
Objectives
By the end of this lab, students will be able to:

• Understand the concept of multi-container pods in Kubernetes • Create and configure sidecar containers for logging purposes • Implement init containers to perform setup tasks before main containers start • Configure shared volumes between containers within a pod • Deploy and troubleshoot multi-container pods • Verify container interactions and shared resource access • Apply best practices for multi-container pod design patterns

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (pods, containers, volumes) • Familiarity with YAML syntax and structure • Basic knowledge of Linux command line operations • Understanding of container fundamentals • Completion of basic Kubernetes pod creation exercises

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-configured • Minikube cluster ready for use • All necessary tools and dependencies installed

Task 1: Create a Pod with Sidecar Container for Logging
Subtask 1.1: Understanding the Architecture
Before creating the pod, let's understand what we're building:

• Main Container: Runs a simple web application • Sidecar Container: Monitors and logs requests from the main container • Shared Volume: Allows both containers to access the same log files

Subtask 1.2: Create the Multi-Container Pod Manifest
Create a new directory for your lab files:

mkdir ~/lab2-multicontainer
cd ~/lab2-multicontainer
Create the pod manifest file:

nano multi-container-pod.yaml
Add the following YAML configuration:

apiVersion: v1
kind: Pod
metadata:
  name: multi-container-app
  labels:
    app: multi-container-demo
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  - name: config-volume
    emptyDir: {}
  
  initContainers:
  - name: config-setup
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      echo "Initializing configuration..."
      echo "server_name=web-app" > /config/app.conf
      echo "log_level=info" >> /config/app.conf
      echo "port=8080" >> /config/app.conf
      echo "Configuration setup complete!"
    volumeMounts:
    - name: config-volume
      mountPath: /config
  
  containers:
  - name: web-app
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
    - name: config-volume
      mountPath: /etc/app-config
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "Starting web application..."
      echo "Configuration files available:"
      ls -la /etc/app-config/
      cat /etc/app-config/app.conf
      nginx -g 'daemon off;'
  
  - name: log-sidecar
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "Starting log sidecar container..."
      while true; do
        if [ -f /var/log/nginx/access.log ]; then
          echo "=== Access Log Entries $(date) ==="
          tail -n 5 /var/log/nginx/access.log
        else
          echo "Waiting for access log file... $(date)"
        fi
        sleep 10
      done
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
Subtask 1.3: Deploy the Multi-Container Pod
Deploy the pod to your Kubernetes cluster:

kubectl apply -f multi-container-pod.yaml
Verify the pod is running:

kubectl get pods
Check the status of all containers in the pod:

kubectl get pod multi-container-app -o jsonpath='{.status.containerStatuses[*].name}'
echo
kubectl get pod multi-container-app -o jsonpath='{.status.initContainerStatuses[*].name}'
Task 2: Verify Init Container Behavior
Subtask 2.1: Check Init Container Logs
View the init container logs to verify it completed successfully:

kubectl logs multi-container-app -c config-setup
You should see output similar to:

Initializing configuration...
Configuration setup complete!
Subtask 2.2: Verify Configuration Files
Check that the configuration files were created properly:

kubectl exec multi-container-app -c web-app -- ls -la /etc/app-config/
kubectl exec multi-container-app -c web-app -- cat /etc/app-config/app.conf
Task 3: Test Sidecar Container Functionality
Subtask 3.1: Generate Web Traffic
First, get the pod's IP address:

kubectl get pod multi-container-app -o wide
Create a test pod to generate HTTP requests:

kubectl run test-client --image=busybox:1.35 --rm -it --restart=Never -- /bin/sh
Inside the test client pod, generate some HTTP requests:

# Replace POD_IP with the actual IP from the previous command
wget -q -O- http://POD_IP
wget -q -O- http://POD_IP/nonexistent
wget -q -O- http://POD_IP
exit
Subtask 3.2: Monitor Sidecar Logs
View the sidecar container logs to see it processing the access logs:

kubectl logs multi-container-app -c log-sidecar --tail=20
You should see the sidecar container displaying access log entries from the nginx container.

Subtask 3.3: Verify Shared Volume Access
Check that both containers can access the shared volume:

# Check from the main container
kubectl exec multi-container-app -c web-app -- ls -la /var/log/nginx/

# Check from the sidecar container
kubectl exec multi-container-app -c log-sidecar -- ls -la /var/log/nginx/
Task 4: Advanced Multi-Container Scenarios
Subtask 4.1: Create a Pod with Multiple Sidecars
Create an advanced multi-container pod with multiple sidecar containers:

nano advanced-multi-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: advanced-multi-container
  labels:
    app: advanced-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  - name: shared-config
    emptyDir: {}
  
  initContainers:
  - name: data-initializer
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      echo "Initializing shared data..."
      mkdir -p /data/input /data/output
      echo "Sample data for processing" > /data/input/sample.txt
      echo "timestamp=$(date)" > /data/input/metadata.txt
      echo "Data initialization complete"
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  containers:
  - name: main-app
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "Main application starting..."
      while true; do
        if [ -f /data/input/sample.txt ]; then
          echo "Processing: $(cat /data/input/sample.txt)" > /data/output/processed.txt
          echo "Processed at: $(date)" >> /data/output/processed.txt
        fi
        sleep 30
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  - name: monitor-sidecar
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "Monitor sidecar starting..."
      while true; do
        echo "=== Monitoring shared data ==="
        echo "Input files:"
        ls -la /data/input/ 2>/dev/null || echo "No input directory"
        echo "Output files:"
        ls -la /data/output/ 2>/dev/null || echo "No output directory"
        if [ -f /data/output/processed.txt ]; then
          echo "Latest processed data:"
          cat /data/output/processed.txt
        fi
        echo "---"
        sleep 20
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  - name: cleanup-sidecar
    image: busybox:1.35
    command: ["/bin/sh"]
    args:
    - -c
    - |
      echo "Cleanup sidecar starting..."
      while true; do
        echo "Running cleanup tasks..."
        # Clean up old temporary files
        find /data -name "*.tmp" -mtime +1 -delete 2>/dev/null || true
        echo "Cleanup completed at $(date)"
        sleep 60
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data
Deploy the advanced pod:

kubectl apply -f advanced-multi-container.yaml
Subtask 4.2: Monitor All Containers
Check the status of all containers:

kubectl get pod advanced-multi-container
kubectl describe pod advanced-multi-container
View logs from different containers:

# Main application logs
kubectl logs advanced-multi-container -c main-app --tail=10

# Monitor sidecar logs
kubectl logs advanced-multi-container -c monitor-sidecar --tail=10

# Cleanup sidecar logs
kubectl logs advanced-multi-container -c cleanup-sidecar --tail=10
Task 5: Troubleshooting and Best Practices
Subtask 5.1: Common Troubleshooting Commands
Use these commands to troubleshoot multi-container pods:

# Get detailed pod information
kubectl describe pod multi-container-app

# Check events related to the pod
kubectl get events --field-selector involvedObject.name=multi-container-app

# Check resource usage
kubectl top pod multi-container-app --containers

# Execute commands in specific containers
kubectl exec multi-container-app -c web-app -- ps aux
kubectl exec multi-container-app -c log-sidecar -- ps aux
Subtask 5.2: Verify Container Communication
Test that containers can communicate through shared volumes:

# Write a file from the main container
kubectl exec multi-container-app -c web-app -- sh -c 'echo "Hello from web-app" > /var/log/nginx/test-message.txt'

# Read the file from the sidecar container
kubectl exec multi-container-app -c log-sidecar -- cat /var/log/nginx/test-message.txt
Subtask 5.3: Resource Monitoring
Monitor resource usage across containers:

# Check memory and CPU usage
kubectl top pod multi-container-app --containers

# Get detailed resource information
kubectl describe pod multi-container-app | grep -A 10 "Containers:"
Task 6: Cleanup and Resource Management
Subtask 6.1: Clean Up Resources
Remove the pods created during this lab:

kubectl delete pod multi-container-app
kubectl delete pod advanced-multi-container
Verify cleanup:

kubectl get pods
Subtask 6.2: Review Configuration Files
Keep your YAML files for future reference:

ls -la ~/lab2-multicontainer/
Troubleshooting Tips
Common Issues and Solutions
Issue: Init container fails to complete Solution: Check init container logs with kubectl logs pod-name -c init-container-name

Issue: Containers cannot access shared volumes Solution: Verify volume mounts in the pod specification and ensure paths are correct

Issue: Sidecar container not seeing main container logs Solution: Ensure both containers mount the same volume path and the main container is writing to the expected location

Issue: Pod stuck in Pending state Solution: Check node resources with kubectl describe nodes and pod events with kubectl describe pod pod-name

Debugging Commands Reference
# View all container logs
kubectl logs pod-name --all-containers=true

# Follow logs in real-time
kubectl logs -f pod-name -c container-name

# Get shell access to specific container
kubectl exec -it pod-name -c container-name -- /bin/sh

# Check volume mounts
kubectl describe pod pod-name | grep -A 5 "Mounts:"
Conclusion
In this lab, you have successfully:

• Created multi-container pods with main containers, sidecar containers, and init containers • Implemented shared volume communication between containers within the same pod • Configured init containers to perform setup tasks before main containers start • Deployed and monitored complex multi-container applications • Applied troubleshooting techniques for multi-container pod issues • Understood design patterns for sidecar and init container use cases

Why This Matters: Multi-container pods are essential for implementing microservices patterns in Kubernetes. The sidecar pattern allows you to separate concerns by having specialized containers handle logging, monitoring, or data processing alongside your main application. Init containers ensure proper initialization order and setup, making your applications more reliable and maintainable.

These skills are crucial for the Certified Kubernetes Application Developer (CKAD) certification and real-world Kubernetes deployments where complex applications require multiple cooperating containers to function effectively.

Key Takeaways: • Containers in the same pod share network and storage resources • Init containers run to completion before main containers start • Sidecar containers provide auxiliary functionality to main applications • Shared volumes enable data exchange between containers • Proper resource management and monitoring are essential for multi-container pods
