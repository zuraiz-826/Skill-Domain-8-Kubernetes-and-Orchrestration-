Lab 19: Service Mesh Security Enhancements
Objectives
By the end of this lab, students will be able to:

• Deploy and configure Istio service mesh with mutual TLS (mTLS) authentication • Implement traffic policies to control and secure service-to-service communication • Configure authorization policies to prevent unauthorized access between services • Monitor and observe service mesh traffic using Istio's built-in observability features • Understand the security benefits of service mesh architecture in microservices environments • Troubleshoot common service mesh security configurations

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with YAML configuration files • Basic knowledge of microservices architecture • Understanding of TLS/SSL concepts • Experience with command-line interface (CLI) tools • Knowledge of networking fundamentals

Lab Environment
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your cloud machine includes: • Kubernetes cluster (minikube or kind) • kubectl CLI tool • Istio service mesh • Sample applications for testing • All required dependencies

Task 1: Deploy Istio Service Mesh with mTLS
Subtask 1.1: Verify Kubernetes Cluster
First, let's ensure our Kubernetes cluster is running and accessible.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Check if any pods are running
kubectl get pods --all-namespaces
Subtask 1.2: Install Istio
Download and install Istio on your cluster.

# Download Istio (latest stable version)
curl -L https://istio.io/downloadIstio | sh -

# Move to Istio directory
cd istio-*

# Add istioctl to PATH
export PATH=$PWD/bin:$PATH

# Verify installation
istioctl version
Subtask 1.3: Install Istio Control Plane
Install Istio with the demo configuration profile, which includes all core components.

# Install Istio with demo profile
istioctl install --set values.defaultRevision=default -y

# Verify installation
kubectl get pods -n istio-system

# Wait for all pods to be ready (this may take a few minutes)
kubectl wait --for=condition=ready pod --all -n istio-system --timeout=300s
Subtask 1.4: Enable Automatic Sidecar Injection
Configure automatic sidecar injection for the default namespace.

# Label the default namespace for automatic sidecar injection
kubectl label namespace default istio-injection=enabled

# Verify the label
kubectl get namespace default --show-labels
Subtask 1.5: Verify mTLS is Enabled by Default
Check that Istio has mTLS enabled by default in STRICT mode.

# Check the default mesh policy
kubectl get peerauthentication --all-namespaces

# Create a mesh-wide mTLS policy if not present
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF
Task 2: Deploy Sample Applications
Subtask 2.1: Deploy Bookinfo Sample Application
Deploy the Bookinfo application to test our service mesh configuration.

# Deploy the Bookinfo application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Verify all services are deployed
kubectl get services

# Verify all pods are running with sidecars
kubectl get pods

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod --all --timeout=300s
Subtask 2.2: Deploy Gateway and Virtual Service
Configure ingress gateway to access the application.

# Deploy the Bookinfo gateway
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Verify gateway and virtual service
kubectl get gateway
kubectl get virtualservice
Subtask 2.3: Get Ingress Gateway URL
Determine the ingress gateway URL for accessing the application.

# Get the ingress gateway service
kubectl get svc istio-ingressgateway -n istio-system

# For minikube, get the URL
export INGRESS_HOST=$(minikube ip)
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

# Test the application
curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
Task 3: Configure Traffic Policies and Security
Subtask 3.1: Create Destination Rules
Configure destination rules to define service subsets and traffic policies.

# Apply destination rules
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
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
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
EOF

# Verify destination rules
kubectl get destinationrules
Subtask 3.2: Implement Authorization Policies
Create authorization policies to control access between services.

# Create a deny-all policy first
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}
EOF

# Allow access to productpage from ingress gateway
cat <<EOF | kubectl apply -f -
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
        principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
EOF

# Allow productpage to access details and reviews
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: details-viewer
  namespace: default
spec:
  selector:
    matchLabels:
      app: details
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: reviews-viewer
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
EOF

# Allow reviews to access ratings
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ratings-viewer
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
EOF
Subtask 3.3: Test Authorization Policies
Verify that the authorization policies are working correctly.

# Test access to the application
curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"

# Check if the page loads correctly (should work)
echo "Testing authorized access..."
curl -s -o /dev/null -w "%{http_code}" http://${GATEWAY_URL}/productpage
Subtask 3.4: Create a Test Pod to Verify Unauthorized Access
Create a test pod to demonstrate that unauthorized access is blocked.

# Create a test pod without proper service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  labels:
    app: test
spec:
  containers:
  - name: test
    image: curlimages/curl:latest
    command: ["/bin/sh"]
    args: ["-c", "sleep 3600"]
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod test-pod --timeout=60s

# Try to access services from unauthorized pod (should fail)
echo "Testing unauthorized access to productpage..."
kubectl exec test-pod -- curl -s -o /dev/null -w "%{http_code}" http://productpage:9080/productpage

echo "Testing unauthorized access to details..."
kubectl exec test-pod -- curl -s -o /dev/null -w "%{http_code}" http://details:9080/details/1
Task 4: Configure Observability and Monitoring
Subtask 4.1: Install Observability Add-ons
Install Kiali, Prometheus, Grafana, and Jaeger for observability.

# Install observability add-ons
kubectl apply -f samples/addons/

# Wait for all observability pods to be ready
kubectl wait --for=condition=ready pod --all -n istio-system --timeout=300s

# Verify all add-ons are running
kubectl get pods -n istio-system
Subtask 4.2: Generate Traffic for Monitoring
Generate some traffic to create data for monitoring.

# Generate traffic to the application
for i in {1..100}; do
  curl -s http://${GATEWAY_URL}/productpage > /dev/null
  echo "Request $i completed"
  sleep 1
done
Subtask 4.3: Access Kiali Dashboard
Open Kiali dashboard to visualize service mesh traffic.

# Start Kiali dashboard (this will open in background)
istioctl dashboard kiali --browser=false &

# Get the Kiali URL
echo "Kiali dashboard available at: http://localhost:20001"

# Alternative: Port forward to access Kiali
kubectl port-forward -n istio-system svc/kiali 20001:20001 &
Subtask 4.4: Access Grafana Dashboard
Open Grafana dashboard to view metrics.

# Start Grafana dashboard
istioctl dashboard grafana --browser=false &

# Get the Grafana URL
echo "Grafana dashboard available at: http://localhost:3000"

# Alternative: Port forward to access Grafana
kubectl port-forward -n istio-system svc/grafana 3000:3000 &
Subtask 4.5: View Service Mesh Metrics
Check various metrics and logs to understand service communication.

# Check mTLS status for all services
istioctl authn tls-check

# View proxy configuration for a specific pod
PRODUCTPAGE_POD=$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config cluster $PRODUCTPAGE_POD

# Check certificates
istioctl proxy-config secret $PRODUCTPAGE_POD

# View access logs
kubectl logs $PRODUCTPAGE_POD -c istio-proxy --tail=10
Task 5: Test mTLS Communication
Subtask 5.1: Verify mTLS is Working
Test that mTLS is properly configured and working between services.

# Check mTLS configuration
kubectl get peerauthentication --all-namespaces

# Verify TLS mode in destination rules
kubectl get destinationrules -o yaml | grep -A 5 -B 5 "mode:"

# Test mTLS connectivity
istioctl authn tls-check productpage.default.svc.cluster.local
istioctl authn tls-check reviews.default.svc.cluster.local
istioctl authn tls-check ratings.default.svc.cluster.local
istioctl authn tls-check details.default.svc.cluster.local
Subtask 5.2: Analyze Traffic Encryption
Examine the traffic to ensure it's encrypted.

# Check proxy configuration for TLS
REVIEWS_POD=$(kubectl get pod -l app=reviews,version=v1 -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config cluster $REVIEWS_POD --fqdn ratings.default.svc.cluster.local

# View certificate details
istioctl proxy-config secret $REVIEWS_POD -o json | jq '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | base64 -d | openssl x509 -text -noout | head -20
Task 6: Advanced Security Configuration
Subtask 6.1: Configure Request Authentication
Set up JWT authentication for additional security.

# Create a request authentication policy
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-example
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.20/security/tools/jwt/samples/jwks.json"
EOF
Subtask 6.2: Create Network Policies
Implement additional network-level security.

# Create a network policy to restrict traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-istio-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: istio-system
  - from:
    - podSelector: {}
EOF
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Pods not getting sidecar injected

# Solution: Check namespace labeling
kubectl get namespace default --show-labels
kubectl label namespace default istio-injection=enabled --overwrite
Issue 2: mTLS connection failures

# Solution: Check peer authentication and destination rules
kubectl get peerauthentication --all-namespaces
kubectl get destinationrules -o yaml
Issue 3: Authorization policies blocking legitimate traffic

# Solution: Check service accounts and principals
kubectl get serviceaccounts
istioctl authn tls-check <service-name>
Issue 4: Observability dashboards not accessible

# Solution: Check if add-ons are running
kubectl get pods -n istio-system
kubectl port-forward -n istio-system svc/kiali 20001:20001
Verification and Testing
Final Verification Steps
Verify mTLS is enforced:
istioctl authn tls-check productpage.default.svc.cluster.local
Test authorization policies:
curl -s -o /dev/null -w "%{http_code}" http://${GATEWAY_URL}/productpage
Check observability data:
kubectl logs -n istio-system deployment/istiod | grep -i "certificate"
Validate traffic encryption:
istioctl proxy-config secret $PRODUCTPAGE_POD
Cleanup
When you're finished with the lab, clean up the resources:

# Remove sample applications
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Remove authorization policies
kubectl delete authorizationpolicy --all

# Remove destination rules
kubectl delete destinationrules --all

# Remove test pod
kubectl delete pod test-pod

# Remove observability add-ons
kubectl delete -f samples/addons/

# Uninstall Istio
istioctl uninstall --purge -y

# Remove namespace label
kubectl label namespace default istio-injection-
Conclusion
In this lab, you have successfully:

• Deployed Istio service mesh with automatic sidecar injection and mutual TLS enabled, providing a secure communication layer for microservices • Configured traffic policies using destination rules to enforce TLS encryption and define service subsets for traffic management • Implemented authorization policies to create a zero-trust security model, preventing unauthorized access between services • Set up comprehensive observability using Kiali, Grafana, Prometheus, and Jaeger to monitor and visualize service mesh traffic • Tested security configurations to verify that mTLS is working correctly and unauthorized access is properly blocked • Explored advanced security features including request authentication and network policies for defense-in-depth

Why This Matters: Service mesh security is crucial in modern microservices architectures because it provides:

Zero-trust networking where every service-to-service communication is authenticated and encrypted
Fine-grained access control allowing you to define exactly which services can communicate with each other
Comprehensive observability giving you visibility into all service communications for security monitoring and compliance
Simplified security management by moving security concerns from application code to infrastructure layer
This knowledge is essential for the Kubernetes and Cloud Native Security Associate (KCSA) certification and real-world cloud-native security implementations. Service mesh security is becoming a standard practice in enterprise Kubernetes deployments, making these skills highly valuable for cloud security professionals.

The hands-on experience you've gained with Istio's security features, including mTLS, authorization policies, and observability tools, provides a solid foundation for implementing production-grade service mesh security in any organization.
