Lab 19: Configuring and Using Service Mesh
Objectives
By the end of this lab, you will be able to:

• Deploy and configure Istio service mesh on a Kubernetes cluster • Understand service mesh architecture and components • Configure traffic routing and load balancing between microservices • Implement mutual TLS (mTLS) for secure service-to-service communication • Monitor and observe service mesh traffic using built-in tools • Apply traffic management policies including circuit breakers and retries

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments) • Familiarity with kubectl command-line tool • Knowledge of YAML configuration files • Understanding of microservices architecture • Basic networking concepts (HTTP, TLS, load balancing)

Lab Environment
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes:

Ubuntu 22.04 LTS with kubectl configured
Kubernetes cluster (single-node for lab purposes)
Internet connectivity for downloading Istio
All necessary permissions configured
Task 1: Deploy Istio Service Mesh
Subtask 1.1: Download and Install Istio
First, we'll download and install Istio, one of the most popular service mesh solutions.

Download Istio installation script:
curl -L https://istio.io/downloadIstio | sh -
Navigate to Istio directory and add istioctl to PATH:
cd istio-*
export PATH=$PWD/bin:$PATH
Verify istioctl installation:
istioctl version
Subtask 1.2: Install Istio on Kubernetes Cluster
Install Istio with default configuration profile:
istioctl install --set values.defaultRevision=default
Verify Istio installation:
kubectl get pods -n istio-system
You should see pods like istiod, istio-proxy, and others running.

Enable automatic sidecar injection for default namespace:
kubectl label namespace default istio-injection=enabled
Verify namespace labeling:
kubectl get namespace -L istio-injection
Subtask 1.3: Deploy Sample Application
We'll deploy a sample bookinfo application to demonstrate service mesh capabilities.

Deploy the Bookinfo application:
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
Verify all services and pods are running:
kubectl get services
kubectl get pods
Wait until all pods show status Running and Ready 2/2 (indicating both application and sidecar containers are running).

Create Istio Gateway and VirtualService:
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
Get the external IP of Istio ingress gateway:
kubectl get svc istio-ingressgateway -n istio-system
Task 2: Configure Traffic Routing and Load Balancing
Subtask 2.1: Create Multiple Versions of a Service
Deploy different versions of the reviews service:
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
Create destination rules for traffic management:
Create a file called destination-rule.yaml:

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
Apply the destination rule:
kubectl apply -f destination-rule.yaml
Subtask 2.2: Configure Traffic Splitting
Create a VirtualService for traffic splitting:
Create a file called reviews-virtual-service.yaml:

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
Apply the VirtualService:
kubectl apply -f reviews-virtual-service.yaml
Test traffic routing:
# Get the gateway URL
export GATEWAY_URL=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test the application multiple times
for i in {1..10}; do
  curl -s "http://$GATEWAY_URL/productpage" | grep -o "glyphicon-star\|color:red"
done
Subtask 2.3: Implement Load Balancing Policies
Create advanced destination rule with load balancing:
Create a file called advanced-destination-rule.yaml:

apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-lb
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    circuitBreaker:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
Apply the advanced destination rule:
kubectl apply -f advanced-destination-rule.yaml
Task 3: Implement Mutual TLS (mTLS)
Subtask 3.1: Enable Automatic mTLS
Check current mTLS status:
istioctl authn tls-check productpage.default.svc.cluster.local
Create PeerAuthentication policy for strict mTLS:
Create a file called peer-authentication.yaml:

apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
Apply the PeerAuthentication policy:
kubectl apply -f peer-authentication.yaml
Subtask 3.2: Verify mTLS Configuration
Check mTLS status after applying policy:
istioctl authn tls-check productpage.default.svc.cluster.local
Verify mTLS is working by checking proxy configuration:
istioctl proxy-config cluster productpage-v1-<pod-id>.default --fqdn reviews.default.svc.cluster.local
Replace <pod-id> with actual pod ID from kubectl get pods.

Test secure communication:
# Deploy a test pod without Istio sidecar
kubectl create namespace test
kubectl run test-pod --image=curlimages/curl -n test --rm -it --restart=Never -- sh

# Try to access the service (should fail due to mTLS)
curl http://productpage.default.svc.cluster.local:9080/productpage
Subtask 3.3: Configure Authorization Policies
Create AuthorizationPolicy for service access control:
Create a file called authorization-policy.yaml:

apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: productpage-viewer
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
  - to:
    - operation:
        methods: ["GET"]
Apply the authorization policy:
kubectl apply -f authorization-policy.yaml
Task 4: Monitor and Observe Service Mesh Traffic
Subtask 4.1: Install Observability Add-ons
Install Kiali, Prometheus, and Grafana:
kubectl apply -f samples/addons/kiali.yaml
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/jaeger.yaml
Wait for all observability pods to be ready:
kubectl get pods -n istio-system
Subtask 4.2: Generate Traffic and Monitor
Generate continuous traffic to the application:
# Run this in a separate terminal
while true; do
  curl -s "http://$GATEWAY_URL/productpage" > /dev/null
  sleep 1
done
Access Kiali dashboard:
kubectl port-forward -n istio-system svc/kiali 20001:20001
Open browser and navigate to http://localhost:20001 (username: admin, password: admin)

Access Grafana dashboard:
kubectl port-forward -n istio-system svc/grafana 3000:3000
Open browser and navigate to http://localhost:3000

Subtask 4.3: Analyze Service Mesh Metrics
View service topology in Kiali:

Navigate to Graph section
Select default namespace
Observe service communication patterns
Check Istio metrics in Grafana:

Go to Dashboards → Istio
Explore Istio Service Dashboard
Analyze request rates, error rates, and latencies
Use istioctl for proxy analysis:

# Check proxy configuration
istioctl proxy-config cluster productpage-v1-<pod-id>.default

# Check listeners
istioctl proxy-config listener productpage-v1-<pod-id>.default

# Check routes
istioctl proxy-config route productpage-v1-<pod-id>.default
Task 5: Advanced Traffic Management
Subtask 5.1: Implement Fault Injection
Create fault injection policy:
Create a file called fault-injection.yaml:

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-fault
spec:
  http:
  - fault:
      delay:
        percentage:
          value: 50
        fixedDelay: 5s
      abort:
        percentage:
          value: 10
        httpStatus: 500
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
Apply fault injection:
kubectl apply -f fault-injection.yaml
Test fault injection:
# Test with jason user (should experience delays and errors)
curl -H "end-user: jason" "http://$GATEWAY_URL/productpage"
Subtask 5.2: Configure Timeout and Retry Policies
Create timeout and retry configuration:
Create a file called timeout-retry.yaml:

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-timeout
spec:
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
    timeout: 3s
    retries:
      attempts: 3
      perTryTimeout: 1s
Apply timeout and retry policy:
kubectl apply -f timeout-retry.yaml
Troubleshooting Tips
Common Issues and Solutions
Pods not showing 2/2 ready status:

Check if namespace has istio-injection label
Restart pods after enabling injection: kubectl rollout restart deployment/productpage-v1
Cannot access application through gateway:

Verify gateway and virtual service configuration
Check if LoadBalancer service has external IP assigned
mTLS not working:

Ensure PeerAuthentication policy is applied correctly
Check proxy configuration with istioctl commands
Observability tools not accessible:

Verify all add-on pods are running
Check port-forward commands are correct
Verification Commands
# Check Istio installation
istioctl verify-install

# Check proxy status
istioctl proxy-status

# Analyze configuration
istioctl analyze

# Check mTLS status
istioctl authn tls-check <service-name>
Cleanup
To clean up the lab environment:

# Remove sample application
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Remove custom configurations
kubectl delete -f destination-rule.yaml
kubectl delete -f reviews-virtual-service.yaml
kubectl delete -f peer-authentication.yaml
kubectl delete -f authorization-policy.yaml

# Remove observability add-ons
kubectl delete -f samples/addons/

# Uninstall Istio
istioctl uninstall --purge
kubectl delete namespace istio-system
Conclusion
In this comprehensive lab, you have successfully:

• Deployed Istio service mesh on a Kubernetes cluster and understood its core components • Configured advanced traffic management including routing, load balancing, and traffic splitting • Implemented mutual TLS (mTLS) for secure service-to-service communication • Applied authorization policies to control access between services • Set up observability tools like Kiali, Prometheus, and Grafana for monitoring • Implemented fault injection and resilience patterns including timeouts and retries

Why This Matters: Service mesh technology is crucial for managing complex microservices architectures in production environments. The skills you've learned enable you to:

Secure communication between services without modifying application code
Implement sophisticated traffic management and deployment strategies
Gain deep visibility into service behavior and performance
Build resilient applications with automatic retry and circuit breaker patterns
These capabilities are essential for cloud-native applications and are highly valued in the industry, particularly for roles involving Kubernetes, DevOps, and cloud architecture. The knowledge gained in this lab directly supports preparation for the Kubernetes and Cloud Native Associate (KCNA) certification and real-world microservices deployments.
