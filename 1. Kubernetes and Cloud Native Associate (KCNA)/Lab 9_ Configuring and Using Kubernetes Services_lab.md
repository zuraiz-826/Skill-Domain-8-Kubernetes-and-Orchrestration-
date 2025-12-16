Lab 9: Configuring and Using Kubernetes Services
Objectives
By the end of this lab, you will be able to:

• Understand the different types of Kubernetes services and their use cases • Deploy applications and expose them using ClusterIP services • Configure and test NodePort services for external access • Set up LoadBalancer services in cloud environments • Verify service connectivity and troubleshoot common issues • Understand service discovery and DNS resolution in Kubernetes

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, deployments, namespaces) • Familiarity with command-line interface operations • Basic knowledge of networking concepts (IP addresses, ports) • Understanding of YAML file structure • Previous experience with kubectl commands

Note: Al Nafi provides ready-to-use Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to begin - no need to build your own VM or install Kubernetes manually.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with: • Kubernetes cluster (single-node or multi-node) • kubectl command-line tool • Docker runtime • All necessary networking components

Task 1: Deploy an Application and Expose it Using ClusterIP Service
Subtask 1.1: Create a Sample Application Deployment
First, let's create a simple web application deployment that we'll use throughout this lab.

Create a new directory for your lab files:
mkdir ~/k8s-services-lab
cd ~/k8s-services-lab
Create a deployment YAML file for a sample nginx application:
cat > nginx-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
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
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
EOF
Deploy the application to your Kubernetes cluster:
kubectl apply -f nginx-deployment.yaml
Verify that the deployment is running:
kubectl get deployments
kubectl get pods -l app=nginx
You should see 3 nginx pods in the Running state.

Subtask 1.2: Create a ClusterIP Service
A ClusterIP service is the default service type that exposes the service on an internal IP within the cluster. It's only accessible from within the cluster.

Create a ClusterIP service YAML file:
cat > nginx-clusterip-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip-service
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
EOF
Apply the ClusterIP service:
kubectl apply -f nginx-clusterip-service.yaml
Verify the service creation:
kubectl get services
kubectl describe service nginx-clusterip-service
Subtask 1.3: Test ClusterIP Service Connectivity
Get the ClusterIP address of your service:
kubectl get service nginx-clusterip-service -o wide
Create a temporary pod to test internal connectivity:
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh
From within the test pod, test the service connectivity:
# Test using the service IP (replace with your actual ClusterIP)
wget -qO- http://10.96.xxx.xxx

# Test using service name (DNS resolution)
wget -qO- http://nginx-clusterip-service

# Test using FQDN
wget -qO- http://nginx-clusterip-service.default.svc.cluster.local

# Exit the test pod
exit
You should see the nginx welcome page HTML content, confirming that the ClusterIP service is working correctly.

Task 2: Change Service Type to NodePort and Verify External Access
Subtask 2.1: Convert ClusterIP to NodePort Service
A NodePort service exposes the service on each node's IP at a static port, making it accessible from outside the cluster.

Create a NodePort service YAML file:
cat > nginx-nodeport-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-service
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
EOF
Apply the NodePort service:
kubectl apply -f nginx-nodeport-service.yaml
Verify the NodePort service:
kubectl get services
kubectl describe service nginx-nodeport-service
Subtask 2.2: Test External Access via NodePort
Get your node's external IP address:
kubectl get nodes -o wide
Test external access using curl (replace with your node's IP):
# Test from within the cluster
curl http://localhost:30080

# If you have external access to the node
curl http://NODE_EXTERNAL_IP:30080
Verify that the service is accessible on all nodes:
# List all nodes
kubectl get nodes

# Check if the service is accessible on each node
for node in $(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'); do
  echo "Testing node: $node"
  curl -s http://$node:30080 | grep -o "<title>.*</title>" || echo "Failed to connect"
done
Subtask 2.3: Understanding NodePort Range and Limitations
Check the default NodePort range:
kubectl cluster-info dump | grep service-node-port-range
Create another NodePort service without specifying a port:
cat > nginx-nodeport-auto.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-auto
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
EOF
Apply and observe the automatically assigned port:
kubectl apply -f nginx-nodeport-auto.yaml
kubectl get service nginx-nodeport-auto
Task 3: Configure LoadBalancer Service in Cloud Environment
Subtask 3.1: Understanding LoadBalancer Services
A LoadBalancer service exposes the service externally using a cloud provider's load balancer. This is the most production-ready way to expose services externally.

Note: LoadBalancer services require a cloud provider that supports external load balancers. If you're running on a local cluster or a cloud provider without load balancer support, the service will remain in Pending state.

Create a LoadBalancer service YAML file:
cat > nginx-loadbalancer-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer-service
  labels:
    app: nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
EOF
Apply the LoadBalancer service:
kubectl apply -f nginx-loadbalancer-service.yaml
Monitor the LoadBalancer service creation:
kubectl get service nginx-loadbalancer-service --watch
Subtask 3.2: Test LoadBalancer Service (Cloud Environment)
If you're in a supported cloud environment:

Wait for the external IP to be assigned:
kubectl get service nginx-loadbalancer-service
Once the external IP is available, test the service:
# Replace EXTERNAL-IP with the actual external IP
curl http://EXTERNAL-IP
Test load balancing by making multiple requests:
for i in {1..10}; do
  curl -s http://EXTERNAL-IP | grep -o "Server: .*" || echo "Request $i failed"
  sleep 1
done
Subtask 3.3: Simulate LoadBalancer with MetalLB (Local Environment)
If you're in a local environment, you can simulate LoadBalancer functionality using MetalLB:

Install MetalLB:
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
Wait for MetalLB to be ready:
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
Configure MetalLB with an IP address pool:
cat > metallb-config.yaml << EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
Apply the MetalLB configuration:
kubectl apply -f metallb-config.yaml
Check if your LoadBalancer service now has an external IP:
kubectl get service nginx-loadbalancer-service
Task 4: Service Discovery and DNS Testing
Subtask 4.1: Test Service DNS Resolution
Create a test pod for DNS testing:
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh
From within the test pod, test different DNS resolution methods:
# Test short name resolution
nslookup nginx-clusterip-service

# Test FQDN resolution
nslookup nginx-clusterip-service.default.svc.cluster.local

# Test service discovery
nslookup nginx-nodeport-service

# Exit the test pod
exit
Subtask 4.2: Explore Service Endpoints
Check the endpoints created by your services:
kubectl get endpoints
kubectl describe endpoints nginx-clusterip-service
Verify that endpoints match your pod IPs:
kubectl get pods -l app=nginx -o wide
Test what happens when you scale your deployment:
# Scale up the deployment
kubectl scale deployment nginx-app --replicas=5

# Check updated endpoints
kubectl get endpoints nginx-clusterip-service

# Scale back down
kubectl scale deployment nginx-app --replicas=3
Task 5: Service Troubleshooting and Best Practices
Subtask 5.1: Common Service Issues and Solutions
Create a service with incorrect selector to demonstrate troubleshooting:
cat > nginx-broken-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-broken-service
spec:
  type: ClusterIP
  selector:
    app: wrong-label
  ports:
  - port: 80
    targetPort: 80
EOF
Apply the broken service:
kubectl apply -f nginx-broken-service.yaml
Troubleshoot the service:
# Check service details
kubectl describe service nginx-broken-service

# Check endpoints (should be empty)
kubectl get endpoints nginx-broken-service

# Compare with working service
kubectl get endpoints nginx-clusterip-service
Fix the service by updating the selector:
kubectl patch service nginx-broken-service -p '{"spec":{"selector":{"app":"nginx"}}}'
Verify the fix:
kubectl get endpoints nginx-broken-service
Subtask 5.2: Service Performance and Monitoring
Check service resource usage:
kubectl top pods -l app=nginx
Monitor service connections:
# Create a load testing pod
kubectl run load-test --image=busybox --rm -it --restart=Never -- sh

# From within the load test pod, generate some traffic
for i in $(seq 1 100); do
  wget -qO- http://nginx-clusterip-service > /dev/null
  echo "Request $i completed"
done

exit
Check service logs:
kubectl logs -l app=nginx --tail=20
Task 6: Cleanup and Service Management
Subtask 6.1: Clean Up Resources
List all services created in this lab:
kubectl get services
Delete the services:
kubectl delete service nginx-clusterip-service
kubectl delete service nginx-nodeport-service
kubectl delete service nginx-loadbalancer-service
kubectl delete service nginx-nodeport-auto
kubectl delete service nginx-broken-service
Delete the deployment:
kubectl delete deployment nginx-app
Verify cleanup:
kubectl get all
Subtask 6.2: Service Configuration Best Practices
Create a production-ready service configuration:
cat > nginx-production-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-production
  labels:
    app: nginx
    environment: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: /health
spec:
  type: LoadBalancer
  selector:
    app: nginx
    environment: production
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
EOF
This configuration includes:

Proper labeling for organization
Annotations for cloud provider specific settings
Named ports for clarity
Multiple ports for HTTP and HTTPS
Session affinity for sticky sessions
Troubleshooting Common Issues
Service Not Accessible
Problem: Service is not responding to requests Solutions:

Check if pods are running: kubectl get pods -l app=nginx
Verify service selector matches pod labels: kubectl describe service SERVICE_NAME
Check endpoints: kubectl get endpoints SERVICE_NAME
Test from within cluster first before external access
LoadBalancer Stuck in Pending
Problem: LoadBalancer service shows <pending> for external IP Solutions:

Verify cloud provider supports LoadBalancer services
Check cloud provider quotas and limits
Review service annotations for cloud-specific requirements
Consider using NodePort or Ingress as alternatives
DNS Resolution Issues
Problem: Services not accessible by name Solutions:

Check CoreDNS pods: kubectl get pods -n kube-system -l k8s-app=kube-dns
Verify service exists: kubectl get services
Test with FQDN: service-name.namespace.svc.cluster.local
Check network policies that might block DNS
Conclusion
In this lab, you have successfully:

• Deployed applications and exposed them using different Kubernetes service types • Configured ClusterIP services for internal cluster communication • Set up NodePort services to enable external access through node IPs • Implemented LoadBalancer services for production-grade external access • Tested service discovery and DNS resolution within the cluster • Troubleshot common service issues and applied best practices

Why This Matters: Kubernetes services are fundamental to application networking and service discovery. Understanding how to properly configure and use different service types is crucial for:

Microservices Architecture: Services enable communication between different application components
High Availability: Load balancing across multiple pod replicas ensures application resilience
Security: ClusterIP services provide internal-only access while LoadBalancers offer controlled external access
Scalability: Services abstract away individual pod IPs, allowing seamless scaling
Production Readiness: Proper service configuration is essential for deploying applications in production environments
The skills you've learned in this lab form the foundation for more advanced Kubernetes networking concepts like Ingress controllers, service meshes, and network policies. These service types and configurations are commonly tested in the Kubernetes and Cloud Native Associate (KCNA) certification and are essential for any Kubernetes practitioner.
