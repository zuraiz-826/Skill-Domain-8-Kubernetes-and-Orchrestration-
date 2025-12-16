Lab 9: Implementing Observability and Service Mesh Security
Objectives
By the end of this lab, you will be able to:

• Install and configure Istio service mesh for secure microservice communication • Implement comprehensive observability using metrics, logs, and distributed tracing • Enable and verify mutual TLS (mTLS) for inter-service communication • Monitor service mesh performance and security using Prometheus, Grafana, and Jaeger • Troubleshoot service mesh connectivity and security issues • Apply security policies and traffic management rules in a service mesh environment

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with YAML configuration files • Knowledge of microservices architecture principles • Basic understanding of TLS/SSL concepts • Experience with command-line interface operations • Understanding of observability concepts (metrics, logs, traces)

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 22.04 LTS with kubectl pre-configured • Kubernetes cluster (single-node for lab purposes) • Docker runtime environment • All necessary networking components configured

Task 1: Installing and Configuring Istio Service Mesh
Subtask 1.1: Download and Install Istio
First, we'll download and install the latest stable version of Istio.

# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Navigate to Istio directory
cd istio-*

# Add istioctl to PATH
export PATH=$PWD/bin:$PATH

# Verify installation
istioctl version
Subtask 1.2: Install Istio on Kubernetes Cluster
# Install Istio with default configuration profile
istioctl install --set values.defaultRevision=default -y

# Verify Istio installation
kubectl get pods -n istio-system

# Enable automatic sidecar injection for default namespace
kubectl label namespace default istio-injection=enabled

# Verify namespace labeling
kubectl get namespace default --show-labels
Subtask 1.3: Deploy Sample Application
We'll deploy the Bookinfo sample application to demonstrate service mesh capabilities.

# Deploy the Bookinfo application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Verify all services are running
kubectl get services
kubectl get pods

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pod --all --timeout=300s
Subtask 1.4: Configure Ingress Gateway
# Deploy Istio Gateway and VirtualService
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Get ingress gateway external IP
kubectl get svc istio-ingressgateway -n istio-system

# Set environment variables for gateway access
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

# Test application access
curl -s "http://${GATEWAY_URL}/productpage" | grep -o "<title>.*</title>"
Task 2: Implementing Comprehensive Observability
Subtask 2.1: Install Observability Add-ons
# Install Prometheus for metrics collection
kubectl apply -f samples/addons/prometheus.yaml

# Install Grafana for metrics visualization
kubectl apply -f samples/addons/grafana.yaml

# Install Jaeger for distributed tracing
kubectl apply -f samples/addons/jaeger.yaml

# Install Kiali for service mesh visualization
kubectl apply -f samples/addons/kiali.yaml

# Verify all add-ons are running
kubectl get pods -n istio-system
Subtask 2.2: Configure Telemetry Collection
Create a telemetry configuration to collect comprehensive metrics:

# Create telemetry configuration file
cat <<EOF > telemetry-config.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: default-metrics
  namespace: istio-system
spec:
  metrics:
  - providers:
    - name: prometheus
  - overrides:
    - match:
        metric: ALL_METRICS
      tagOverrides:
        request_id:
          operation: UPSERT
          value: "%{REQUEST_ID}"
---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: default-tracing
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: jaeger
EOF

# Apply telemetry configuration
kubectl apply -f telemetry-config.yaml
Subtask 2.3: Generate Traffic for Observability Data
# Create a script to generate continuous traffic
cat <<EOF > generate-traffic.sh
#!/bin/bash
for i in {1..100}; do
  curl -s "http://${GATEWAY_URL}/productpage" > /dev/null
  echo "Request $i completed"
  sleep 2
done
EOF

# Make script executable and run it
chmod +x generate-traffic.sh
./generate-traffic.sh &

# Store the process ID to stop it later
TRAFFIC_PID=$!
echo "Traffic generation started with PID: $TRAFFIC_PID"
Subtask 2.4: Access Observability Dashboards
# Access Grafana dashboard
kubectl port-forward -n istio-system svc/grafana 3000:3000 &
echo "Grafana available at: http://localhost:3000"

# Access Jaeger tracing UI
kubectl port-forward -n istio-system svc/jaeger 16686:16686 &
echo "Jaeger available at: http://localhost:16686"

# Access Kiali service mesh dashboard
kubectl port-forward -n istio-system svc/kiali 20001:20001 &
echo "Kiali available at: http://localhost:20001"

# Access Prometheus metrics
kubectl port-forward -n istio-system svc/prometheus 9090:9090 &
echo "Prometheus available at: http://localhost:9090"
Task 3: Enabling Mutual TLS (mTLS) for Inter-Service Communication
Subtask 3.1: Verify Current TLS Configuration
# Check current TLS configuration
istioctl authn tls-check

# Verify mTLS status for services
kubectl get peerauthentication --all-namespaces
kubectl get destinationrule --all-namespaces
Subtask 3.2: Enable Strict mTLS for Default Namespace
# Create PeerAuthentication policy for strict mTLS
cat <<EOF > strict-mtls.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
EOF

# Apply strict mTLS policy
kubectl apply -f strict-mtls.yaml

# Verify policy is applied
kubectl get peerauthentication
Subtask 3.3: Configure Destination Rules for mTLS
# Create DestinationRule for mTLS traffic
cat <<EOF > destination-rule-mtls.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: default
  namespace: default
spec:
  host: "*.default.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF

# Apply destination rule
kubectl apply -f destination-rule-mtls.yaml

# Verify destination rule
kubectl get destinationrule
Subtask 3.4: Test mTLS Configuration
# Test internal service communication with mTLS
kubectl exec -it $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') -- curl -s http://details:9080/details/0

# Verify TLS certificates are being used
istioctl proxy-config secret $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')

# Check mTLS status for all services
istioctl authn tls-check $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}').default
Subtask 3.5: Create Authorization Policies
# Create authorization policy to control access
cat <<EOF > authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: productpage-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
    - source:
        principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
  - to:
    - operation:
        methods: ["GET", "POST"]
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: details-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: details
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
EOF

# Apply authorization policies
kubectl apply -f authorization-policy.yaml

# Verify policies are applied
kubectl get authorizationpolicy
Task 4: Advanced Monitoring and Security Analysis
Subtask 4.1: Create Custom Metrics Dashboard
# Create custom ServiceMonitor for additional metrics
cat <<EOF > custom-metrics.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-dashboard
  namespace: istio-system
  labels:
    grafana_dashboard: "1"
data:
  custom-dashboard.json: |
    {
      "dashboard": {
        "title": "Service Mesh Security Dashboard",
        "panels": [
          {
            "title": "mTLS Success Rate",
            "type": "stat",
            "targets": [
              {
                "expr": "sum(rate(istio_requests_total{security_policy=\"mutual_tls\"}[5m])) / sum(rate(istio_requests_total[5m])) * 100"
              }
            ]
          }
        ]
      }
    }
EOF

# Apply custom metrics configuration
kubectl apply -f custom-metrics.yaml
Subtask 4.2: Monitor Security Events
# Create a script to monitor security-related logs
cat <<EOF > monitor-security.sh
#!/bin/bash
echo "Monitoring Istio security events..."

# Monitor authentication failures
kubectl logs -n istio-system -l app=istiod --tail=100 | grep -i "authentication\|authorization\|tls"

# Monitor proxy access logs
kubectl logs -l app=productpage -c istio-proxy --tail=50

# Check certificate rotation events
kubectl get events --field-selector reason=SecretRotated -n istio-system
EOF

# Make script executable and run
chmod +x monitor-security.sh
./monitor-security.sh
Subtask 4.3: Performance Impact Analysis
# Measure latency with and without service mesh
echo "Testing application performance..."

# Test direct service access (if possible)
kubectl exec -it $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') -- time curl -s http://details:9080/details/0

# Generate load test
cat <<EOF > load-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-test
spec:
  containers:
  - name: load-test
    image: busybox
    command: ['sh', '-c', 'while true; do wget -q -O- http://productpage:9080/productpage; sleep 1; done']
  restartPolicy: Never
EOF

# Apply load test
kubectl apply -f load-test.yaml

# Monitor resource usage
kubectl top pods
kubectl top nodes
Task 5: Troubleshooting and Validation
Subtask 5.1: Validate Service Mesh Configuration
# Comprehensive configuration validation
istioctl analyze

# Check proxy configuration
istioctl proxy-status

# Verify sidecar injection
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'

# Check for configuration conflicts
istioctl proxy-config cluster $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')
Subtask 5.2: Security Validation Tests
# Test unauthorized access (should fail)
kubectl run test-pod --image=curlimages/curl --rm -it --restart=Never -- curl -s http://details:9080/details/0

# Verify mTLS is enforced
kubectl exec -it $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- openssl s_client -connect details:9080 -servername details

# Check certificate validity
istioctl proxy-config secret $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') -o json | jq '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | base64 -d | openssl x509 -text -noout
Subtask 5.3: Common Issues and Solutions
# Check for common configuration issues
echo "Checking for common service mesh issues..."

# Verify namespace injection
kubectl get namespace default -o yaml | grep istio-injection

# Check for conflicting policies
kubectl get peerauthentication,destinationrule,authorizationpolicy --all-namespaces

# Validate gateway configuration
istioctl analyze samples/bookinfo/networking/bookinfo-gateway.yaml

# Check proxy logs for errors
kubectl logs -l app=productpage -c istio-proxy --tail=20 | grep -i error
Cleanup
# Stop traffic generation
kill $TRAFFIC_PID

# Stop port forwarding processes
pkill -f "kubectl port-forward"

# Remove sample application
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Remove security policies
kubectl delete -f strict-mtls.yaml
kubectl delete -f destination-rule-mtls.yaml
kubectl delete -f authorization-policy.yaml

# Remove observability add-ons
kubectl delete -f samples/addons/

# Remove Istio (optional)
istioctl uninstall --purge -y
kubectl delete namespace istio-system
Troubleshooting Tips
Common Issues and Solutions
Issue: Pods not starting after enabling Istio injection Solution: Check if the namespace has sufficient resources and verify sidecar injection status

kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
Issue: mTLS connection failures Solution: Verify PeerAuthentication and DestinationRule configurations match

istioctl authn tls-check <pod-name>.<namespace>
kubectl get peerauthentication,destinationrule -o yaml
Issue: Observability data not appearing Solution: Ensure telemetry configuration is correct and services are generating traffic

kubectl logs -n istio-system -l app=istiod
kubectl get telemetry -n istio-system
Issue: Authorization policies blocking legitimate traffic Solution: Review and adjust authorization policies, check service account configurations

kubectl logs -l app=<app-name> -c istio-proxy | grep RBAC
kubectl get serviceaccount
Conclusion
In this comprehensive lab, you have successfully:

• Implemented a complete service mesh solution using Istio, providing secure communication between microservices • Established comprehensive observability through metrics collection with Prometheus, visualization with Grafana, and distributed tracing with Jaeger • Secured inter-service communication by enabling strict mutual TLS (mTLS) and implementing authorization policies • Gained hands-on experience with service mesh security concepts that are essential for modern cloud-native applications • Learned troubleshooting techniques for common service mesh issues and configuration problems

This lab demonstrates critical skills for the Kubernetes and Cloud Native Security Associate (KCSA) certification, particularly in areas of:

Service mesh security implementation
Observability and monitoring in cloud-native environments
Zero-trust networking principles
Security policy enforcement in microservices architectures
The knowledge gained from this lab is directly applicable to real-world scenarios where organizations need to secure microservices communication, implement comprehensive monitoring, and maintain visibility into complex distributed systems. Service mesh technology like Istio is becoming increasingly important in enterprise environments for managing security, observability, and traffic management at scale.

Understanding these concepts will help you design and implement secure, observable, and manageable microservices architectures in production environments, making you a valuable asset in cloud-native security roles.
