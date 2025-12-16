Lab 18: Observability with Prometheus and Grafana
Objectives
By the end of this lab, you will be able to:

• Deploy Prometheus monitoring system to a Kubernetes cluster • Configure Prometheus to scrape metrics from cluster components • Install and configure Grafana for data visualization • Create custom dashboards in Grafana to monitor cluster health • Set up alerting rules for critical metrics like CPU and memory usage • Understand the fundamentals of observability in cloud-native environments • Implement monitoring best practices for Kubernetes workloads

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with YAML configuration files • Basic knowledge of Linux command line operations • Understanding of containerization concepts • Access to kubectl command-line tool • Basic understanding of monitoring and metrics concepts

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 20.04 LTS with Docker pre-installed • Kubernetes cluster (minikube) ready to use • kubectl configured and ready • Helm package manager installed • All necessary networking configured

Task 1: Deploy Prometheus to the Cluster
Subtask 1.1: Verify Cluster Status
First, let's ensure our Kubernetes cluster is running properly.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# Check if minikube is running (if using minikube)
minikube status
Subtask 1.2: Create Monitoring Namespace
Create a dedicated namespace for our monitoring stack.

# Create monitoring namespace
kubectl create namespace monitoring

# Verify namespace creation
kubectl get namespaces
Subtask 1.3: Deploy Prometheus Using Helm
We'll use Helm to deploy Prometheus, which simplifies the installation process.

# Add Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update Helm repositories
helm repo update

# Install Prometheus stack (includes Prometheus, Grafana, and AlertManager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
Subtask 1.4: Verify Prometheus Deployment
Check that all components are deployed successfully.

# Check all pods in monitoring namespace
kubectl get pods -n monitoring

# Check services
kubectl get services -n monitoring

# Wait for all pods to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=prometheus -n monitoring --timeout=300s
Subtask 1.5: Access Prometheus Web Interface
Set up port forwarding to access Prometheus web interface.

# Port forward Prometheus service
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# Note: The & runs the command in background
# You can now access Prometheus at http://localhost:9090
Open a web browser and navigate to http://localhost:9090 to verify Prometheus is running.

Task 2: Configure Prometheus to Scrape Metrics
Subtask 2.1: Understand Default Configuration
Prometheus is already configured to scrape basic Kubernetes metrics. Let's examine the configuration.

# View Prometheus configuration
kubectl get configmap -n monitoring prometheus-kube-prometheus-prometheus-rulefiles-0 -o yaml
Subtask 2.2: Deploy Sample Application for Monitoring
Let's deploy a sample application that exposes metrics.

# Create a sample application deployment
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
  labels:
    app: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: sample-app
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  namespace: default
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - port: 9100
    targetPort: 9100
    name: metrics
EOF
Subtask 2.3: Create ServiceMonitor for Custom Application
Create a ServiceMonitor to tell Prometheus to scrape our sample application.

# Create ServiceMonitor
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-monitor
  namespace: monitoring
  labels:
    app: sample-app
spec:
  selector:
    matchLabels:
      app: sample-app
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF
Subtask 2.4: Verify Metrics Collection
Check that Prometheus is collecting metrics from our application.

# Check if ServiceMonitor is created
kubectl get servicemonitor -n monitoring

# Verify targets in Prometheus web interface
# Go to http://localhost:9090/targets to see all monitored targets
Task 3: Install and Configure Grafana
Subtask 3.1: Access Grafana
Grafana was installed as part of the Prometheus stack. Let's access it.

# Get Grafana admin password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
echo

# Port forward Grafana service
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &
Subtask 3.2: Login to Grafana
Open web browser and go to http://localhost:3000
Login with:
Username: admin
Password: (use the password from previous step)
Subtask 3.3: Verify Prometheus Data Source
Grafana should already be configured with Prometheus as a data source.

In Grafana, go to Configuration → Data Sources
Verify that Prometheus is listed and connected
The URL should be: http://prometheus-kube-prometheus-prometheus:9090
Task 4: Create Custom Dashboards in Grafana
Subtask 4.1: Import Pre-built Kubernetes Dashboard
Let's import a comprehensive Kubernetes dashboard.

In Grafana, click the + icon → Import
Enter dashboard ID: 315 (Kubernetes cluster monitoring dashboard)
Click Load
Select Prometheus as the data source
Click Import
Subtask 4.2: Create Custom Dashboard for Node Metrics
Create a custom dashboard to monitor node-specific metrics.

Click + → Dashboard → Add new panel

Configure the first panel:

Title: CPU Usage by Node
Query: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
Visualization: Time series
Click Apply
Add another panel:

Title: Memory Usage by Node
Query: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
Visualization: Stat
Unit: Percent (0-100)
Click Apply
Add third panel:

Title: Disk Usage
Query: 100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)
Visualization: Gauge
Unit: Percent (0-100)
Click Apply
Save the dashboard:

Click Save (disk icon)
Name: Custom Node Monitoring
Click Save
Subtask 4.3: Create Pod Monitoring Dashboard
Create a dashboard specifically for pod metrics.

Create new dashboard: + → Dashboard

Add panel for Pod CPU Usage:

Title: Pod CPU Usage
Query: sum(rate(container_cpu_usage_seconds_total{container!="POD",container!=""}[5m])) by (pod)
Visualization: Time series
Add panel for Pod Memory Usage:

Title: Pod Memory Usage
Query: sum(container_memory_working_set_bytes{container!="POD",container!=""}) by (pod)
Visualization: Time series
Unit: Bytes
Add panel for Pod Count:

Title: Running Pods
Query: count(kube_pod_info)
Visualization: Stat
Save dashboard as Pod Monitoring

Task 5: Set Up Alerts for Critical Metrics
Subtask 5.1: Create CPU Usage Alert Rule
Create an alert rule for high CPU usage.

# Create PrometheusRule for CPU alerts
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cpu-usage-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  - name: cpu.rules
    rules:
    - alert: HighCPUUsage
      expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
        description: "CPU usage is above 80% for more than 2 minutes on {{ \$labels.instance }}"
    
    - alert: CriticalCPUUsage
      expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 95
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Critical CPU usage detected"
        description: "CPU usage is above 95% for more than 1 minute on {{ \$labels.instance }}"
EOF
Subtask 5.2: Create Memory Usage Alert Rule
Create alert rules for memory usage.

# Create PrometheusRule for Memory alerts
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: memory-usage-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  - name: memory.rules
    rules:
    - alert: HighMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"
        description: "Memory usage is above 85% for more than 2 minutes on {{ \$labels.instance }}"
    
    - alert: CriticalMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 95
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Critical memory usage detected"
        description: "Memory usage is above 95% for more than 1 minute on {{ \$labels.instance }}"
EOF
Subtask 5.3: Create Pod-specific Alert Rules
Create alerts for pod-related issues.

# Create PrometheusRule for Pod alerts
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  - name: pod.rules
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod is crash looping"
        description: "Pod {{ \$labels.pod }} in namespace {{ \$labels.namespace }} is restarting frequently"
    
    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="false"} == 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod not ready"
        description: "Pod {{ \$labels.pod }} in namespace {{ \$labels.namespace }} has been not ready for more than 5 minutes"
EOF
Subtask 5.4: Verify Alert Rules
Check that alert rules are loaded correctly.

# Verify PrometheusRules are created
kubectl get prometheusrules -n monitoring

# Check Prometheus web interface for alerts
# Go to http://localhost:9090/alerts to see all configured alerts
Subtask 5.5: Configure Grafana Alerting
Set up alerting in Grafana for dashboard panels.

In Grafana, go to your Custom Node Monitoring dashboard
Edit the CPU Usage by Node panel
Go to Alert tab
Click Create Alert
Configure alert condition:
Query: A (use existing query)
Condition: IS ABOVE 80
Evaluation: Every 10s for 1m
Add notification:
Name: High CPU Alert
Message: CPU usage is critically high
Click Save
Subtask 5.6: Test Alert Functionality
Create a high CPU load to test our alerts.

# Deploy a CPU stress test pod
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-stress-test
  namespace: default
spec:
  containers:
  - name: stress
    image: progrium/stress
    command: ["stress"]
    args: ["--cpu", "2", "--timeout", "300s"]
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
EOF
Monitor the alerts in both Prometheus (http://localhost:9090/alerts) and Grafana to see if they trigger.

Task 6: Advanced Monitoring Configuration
Subtask 6.1: Configure Custom Metrics Collection
Create a custom application that exposes business metrics.

# Deploy a sample application with custom metrics
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-metrics-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-metrics-app
  template:
    metadata:
      labels:
        app: custom-metrics-app
    spec:
      containers:
      - name: metrics-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        - containerPort: 9113
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: custom-metrics-service
  namespace: default
  labels:
    app: custom-metrics-app
spec:
  selector:
    app: custom-metrics-app
  ports:
  - port: 80
    name: http
  - port: 9113
    name: metrics
EOF
Subtask 6.2: Create ServiceMonitor for Custom Metrics
# Create ServiceMonitor for custom application
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: custom-metrics-monitor
  namespace: monitoring
  labels:
    app: custom-metrics-app
spec:
  selector:
    matchLabels:
      app: custom-metrics-app
  namespaceSelector:
    matchNames:
    - default
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
EOF
Troubleshooting Common Issues
Issue 1: Pods Not Starting
If pods are not starting properly:

# Check pod status and events
kubectl describe pod -n monitoring

# Check logs
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus

# Verify resource availability
kubectl top nodes
kubectl top pods -n monitoring
Issue 2: Metrics Not Appearing
If metrics are not showing up in Prometheus:

# Verify ServiceMonitor configuration
kubectl get servicemonitor -n monitoring -o yaml

# Check Prometheus targets
# Go to http://localhost:9090/targets

# Verify service endpoints
kubectl get endpoints -n default
Issue 3: Grafana Connection Issues
If Grafana cannot connect to Prometheus:

# Check Grafana logs
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana

# Verify Prometheus service
kubectl get svc -n monitoring prometheus-kube-prometheus-prometheus

# Test connectivity from Grafana pod
kubectl exec -n monitoring -it deployment/prometheus-grafana -- wget -qO- http://prometheus-kube-prometheus-prometheus:9090/api/v1/status/config
Cleanup
When you're finished with the lab, clean up the resources:

# Stop port forwarding processes
pkill -f "kubectl port-forward"

# Delete test applications
kubectl delete pod cpu-stress-test
kubectl delete deployment sample-app custom-metrics-app
kubectl delete service sample-app-service custom-metrics-service

# Delete ServiceMonitors
kubectl delete servicemonitor -n monitoring sample-app-monitor custom-metrics-monitor

# Delete PrometheusRules
kubectl delete prometheusrules -n monitoring cpu-usage-alerts memory-usage-alerts pod-alerts

# Uninstall Prometheus stack (optional)
helm uninstall prometheus -n monitoring

# Delete monitoring namespace (optional)
kubectl delete namespace monitoring
Conclusion
Congratulations! You have successfully completed Lab 18: Observability with Prometheus and Grafana. In this comprehensive lab, you have accomplished the following:

Key Achievements:

• Deployed a complete monitoring stack using Prometheus and Grafana in a Kubernetes environment • Configured metric collection from both system components and custom applications • Created custom dashboards in Grafana to visualize cluster health and performance metrics • Implemented alerting rules for critical metrics including CPU usage, memory consumption, and pod health • Set up automated monitoring for Kubernetes workloads using ServiceMonitors • Learned troubleshooting techniques for common monitoring issues

Why This Matters:

Observability is crucial in modern cloud-native environments because it provides the visibility needed to:

Maintain system reliability by detecting issues before they impact users
Optimize resource utilization and reduce costs through data-driven decisions
Meet SLA requirements by monitoring performance metrics continuously
Enable proactive maintenance through predictive alerting
Support incident response with detailed metrics and historical data
Real-World Applications:

The skills you've developed in this lab are directly applicable to:

Production Kubernetes clusters in enterprise environments
DevOps practices for continuous monitoring and improvement
Site Reliability Engineering (SRE) responsibilities
Cloud-native application development with built-in observability
Compliance and audit requirements for system monitoring
Next Steps:

To further enhance your observability skills, consider exploring:

Advanced Prometheus query language (PromQL) for complex metrics analysis
Integration with external alerting systems like PagerDuty or Slack
Log aggregation with tools like Fluentd and Elasticsearch
Distributed tracing with Jaeger or Zipkin
Custom metric exporters for specific applications
This lab has provided you with a solid foundation in Kubernetes observability that will serve you well in your cloud-native journey and preparation for the Kubernetes and Cloud Native Associate (KCNA) certification.
