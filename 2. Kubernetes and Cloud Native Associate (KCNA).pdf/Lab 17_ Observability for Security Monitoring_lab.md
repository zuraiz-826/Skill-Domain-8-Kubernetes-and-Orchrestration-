Lab 17: Observability for Security Monitoring
Objectives
By the end of this lab, you will be able to:

• Deploy and configure Prometheus for collecting security-related metrics • Set up Grafana dashboards to visualize security monitoring data • Configure alerting rules for detecting unusual security activities • Implement log aggregation and analysis for security incident investigation • Create custom metrics for API request monitoring and rate limiting detection • Investigate potential security incidents using observability tools • Understand the relationship between observability and security monitoring

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Linux command line operations • Familiarity with Docker and containerization concepts • Knowledge of YAML configuration files • Understanding of basic networking concepts (ports, HTTP requests) • Basic knowledge of Kubernetes concepts (pods, services, deployments) • Familiarity with log file formats and analysis

Lab Environment
Al Nafi provides you with a pre-configured Linux-based cloud machine. Simply click Start Lab to access your environment. No need to build your own VM or install additional software - everything you need is ready to use.

Your lab environment includes: • Ubuntu 20.04 LTS with Docker pre-installed • kubectl configured for local Kubernetes cluster • All necessary networking configurations • Sample applications for testing

Task 1: Deploy Prometheus for Security Metrics Collection
Subtask 1.1: Set Up the Lab Environment
First, let's prepare our working directory and verify our environment.

# Create a working directory for our lab
mkdir -p ~/security-monitoring-lab
cd ~/security-monitoring-lab

# Verify Docker is running
sudo systemctl status docker

# Check if kubectl is available
kubectl version --client
Subtask 1.2: Create Prometheus Configuration
Create a Prometheus configuration file that focuses on security-related metrics.

# Create prometheus directory
mkdir -p prometheus

# Create prometheus.yml configuration file
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "security_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'api-security-metrics'
    static_configs:
      - targets: ['api-app:8080']
    metrics_path: '/metrics'
    scrape_interval: 5s

  - job_name: 'nginx-exporter'
    static_configs:
      - targets: ['nginx-exporter:9113']
EOF
Subtask 1.3: Create Security Alert Rules
Create alerting rules for detecting security anomalies.

# Create security alert rules
cat > prometheus/security_rules.yml << 'EOF'
groups:
- name: security_alerts
  rules:
  - alert: HighAPIRequestRate
    expr: rate(http_requests_total[5m]) > 10
    for: 2m
    labels:
      severity: warning
      category: security
    annotations:
      summary: "High API request rate detected"
      description: "API request rate is {{ $value }} requests/second, which exceeds the threshold of 10 req/sec"

  - alert: UnauthorizedAccessAttempts
    expr: rate(http_requests_total{status=~"401|403"}[5m]) > 2
    for: 1m
    labels:
      severity: critical
      category: security
    annotations:
      summary: "Multiple unauthorized access attempts"
      description: "{{ $value }} unauthorized requests per second detected"

  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 3m
    labels:
      severity: warning
      category: performance
    annotations:
      summary: "High CPU usage detected"
      description: "CPU usage is above 80% for more than 3 minutes"

  - alert: SuspiciousNetworkActivity
    expr: rate(node_network_receive_bytes_total[5m]) > 10000000
    for: 2m
    labels:
      severity: warning
      category: security
    annotations:
      summary: "Suspicious network activity detected"
      description: "High network traffic: {{ $value }} bytes/second"
EOF
Subtask 1.4: Deploy Prometheus with Docker Compose
Create a Docker Compose file to deploy our monitoring stack.

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager:/etc/alertmanager
    networks:
      - monitoring

  api-app:
    image: nginx:alpine
    container_name: api-app
    ports:
      - "8080:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/html:/usr/share/nginx/html
    networks:
      - monitoring

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx-exporter
    ports:
      - "9113:9113"
    command:
      - '-nginx.scrape-uri=http://api-app/nginx_status'
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
EOF
Subtask 1.5: Configure Supporting Services
Create configuration files for our supporting services.

# Create alertmanager directory and configuration
mkdir -p alertmanager
cat > alertmanager/alertmanager.yml << 'EOF'
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alertmanager@example.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://localhost:5001/'
    send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
EOF

# Create nginx configuration for our test API
mkdir -p nginx/html
cat > nginx/nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        
        location /nginx_status {
            stub_status on;
            access_log off;
            allow all;
        }
        
        location /api/v1/users {
            return 200 '{"users": ["user1", "user2"]}';
            add_header Content-Type application/json;
        }
        
        location /api/v1/login {
            return 401 '{"error": "Unauthorized"}';
            add_header Content-Type application/json;
        }
        
        location /metrics {
            return 200 'http_requests_total{method="GET",status="200"} 1';
            add_header Content-Type text/plain;
        }
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
EOF

# Create a simple HTML page
cat > nginx/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Security Monitoring Test API</title>
</head>
<body>
    <h1>Security Monitoring Test API</h1>
    <p>Available endpoints:</p>
    <ul>
        <li><a href="/api/v1/users">/api/v1/users</a> - Returns user list</li>
        <li><a href="/api/v1/login">/api/v1/login</a> - Returns 401 (for testing)</li>
        <li><a href="/metrics">/metrics</a> - Prometheus metrics</li>
        <li><a href="/nginx_status">/nginx_status</a> - Nginx status</li>
    </ul>
</body>
</html>
EOF
Subtask 1.6: Start the Monitoring Stack
Deploy all services using Docker Compose.

# Start all services
sudo docker-compose up -d

# Verify all containers are running
sudo docker-compose ps

# Check logs if needed
sudo docker-compose logs prometheus
Task 2: Set Up Grafana Dashboards for Security Visualization
Subtask 2.1: Configure Grafana Data Sources
First, let's set up Grafana provisioning for automatic configuration.

# Create Grafana provisioning directories
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards

# Create datasource configuration
cat > grafana/provisioning/datasources/prometheus.yml << 'EOF'
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
EOF

# Create dashboard provider configuration
cat > grafana/provisioning/dashboards/dashboard.yml << 'EOF'
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
EOF
Subtask 2.2: Create Security Monitoring Dashboard
Create a comprehensive security monitoring dashboard.

# Create security dashboard JSON
cat > grafana/provisioning/dashboards/security-monitoring.json << 'EOF'
{
  "dashboard": {
    "id": null,
    "title": "Security Monitoring Dashboard",
    "tags": ["security", "monitoring"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "API Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec"
          }
        ],
        "xAxis": {
          "show": true
        },
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "Failed Authentication Attempts",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"401|403\"}[5m]))",
            "legendFormat": "Failed Attempts/sec"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        }
      },
      {
        "id": 3,
        "title": "System CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by(instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU Usage %"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 8
        }
      },
      {
        "id": 4,
        "title": "Network Traffic",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(node_network_receive_bytes_total[5m])",
            "legendFormat": "Received {{device}}"
          },
          {
            "expr": "rate(node_network_transmit_bytes_total[5m])",
            "legendFormat": "Transmitted {{device}}"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 8
        }
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
EOF
Subtask 2.3: Restart Services and Access Grafana
# Restart docker-compose to apply new configurations
sudo docker-compose down
sudo docker-compose up -d

# Wait for services to start
sleep 30

# Check service status
sudo docker-compose ps
Now access Grafana:

Open your web browser
Navigate to http://localhost:3000
Login with username: admin and password: admin123
You should see the Security Monitoring Dashboard automatically loaded
Task 3: Configure Security Alerts and Notifications
Subtask 3.1: Test Alert Generation
Let's generate some traffic to trigger our security alerts.

# Create a script to generate API traffic
cat > generate_traffic.sh << 'EOF'
#!/bin/bash

echo "Generating normal API traffic..."
for i in {1..5}; do
    curl -s http://localhost:8080/api/v1/users > /dev/null
    sleep 1
done

echo "Generating high-rate traffic to trigger alerts..."
for i in {1..50}; do
    curl -s http://localhost:8080/api/v1/users > /dev/null &
done
wait

echo "Generating unauthorized access attempts..."
for i in {1..10}; do
    curl -s http://localhost:8080/api/v1/login > /dev/null
    sleep 0.5
done

echo "Traffic generation complete!"
EOF

# Make script executable and run it
chmod +x generate_traffic.sh
./generate_traffic.sh
Subtask 3.2: Monitor Alerts in Prometheus
Check if alerts are firing in Prometheus:

# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# Check active alerts
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {alertname: .labels.alertname, state: .state}'
Access Prometheus web interface:

Open http://localhost:9090 in your browser
Go to Status > Targets to verify all targets are up
Go to Alerts to see active alerts
Try querying: rate(http_requests_total[5m]) in the query interface
Subtask 3.3: Create Custom Metrics Exporter
Create a simple Python script that exports custom security metrics.

# Install Python and pip if not available
sudo apt update
sudo apt install -y python3 python3-pip

# Install required Python packages
pip3 install prometheus_client flask

# Create custom metrics exporter
cat > custom_metrics_exporter.py << 'EOF'
#!/usr/bin/env python3

from prometheus_client import start_http_server, Counter, Histogram, Gauge
import time
import random
import threading

# Define custom metrics
login_attempts = Counter('security_login_attempts_total', 'Total login attempts', ['status'])
api_response_time = Histogram('security_api_response_seconds', 'API response time')
active_sessions = Gauge('security_active_sessions', 'Number of active user sessions')
failed_logins_per_ip = Counter('security_failed_logins_per_ip_total', 'Failed logins per IP', ['ip_address'])

def simulate_security_events():
    """Simulate various security events for monitoring"""
    while True:
        # Simulate login attempts
        if random.random() < 0.3:  # 30% chance of failed login
            login_attempts.labels(status='failed').inc()
            # Simulate failed login from random IP
            fake_ip = f"192.168.1.{random.randint(1, 254)}"
            failed_logins_per_ip.labels(ip_address=fake_ip).inc()
        else:
            login_attempts.labels(status='success').inc()
        
        # Simulate API response times
        response_time = random.uniform(0.1, 2.0)
        api_response_time.observe(response_time)
        
        # Simulate active sessions
        active_sessions.set(random.randint(10, 100))
        
        time.sleep(random.uniform(1, 5))

if __name__ == '__main__':
    # Start metrics server on port 8000
    start_http_server(8000)
    print("Custom security metrics exporter started on port 8000")
    
    # Start background thread for simulating events
    event_thread = threading.Thread(target=simulate_security_events)
    event_thread.daemon = True
    event_thread.start()
    
    # Keep the main thread alive
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("Shutting down...")
EOF

# Make the script executable
chmod +x custom_metrics_exporter.py

# Run the custom metrics exporter in background
python3 custom_metrics_exporter.py &
EXPORTER_PID=$!
echo "Custom metrics exporter started with PID: $EXPORTER_PID"
Subtask 3.4: Update Prometheus Configuration
Add the custom metrics exporter to Prometheus configuration.

# Update prometheus.yml to include custom exporter
cat >> prometheus/prometheus.yml << 'EOF'

  - job_name: 'custom-security-metrics'
    static_configs:
      - targets: ['host.docker.internal:8000']
    scrape_interval: 10s
EOF

# Reload Prometheus configuration
curl -X POST http://localhost:9090/-/reload
Task 4: Implement Log Aggregation and Analysis
Subtask 4.1: Set Up Log Collection
Create a simple log aggregation system using Fluentd and Elasticsearch (lightweight version).

# Create logs directory
mkdir -p logs

# Create a sample application that generates security logs
cat > log_generator.py << 'EOF'
#!/usr/bin/env python3

import json
import time
import random
import datetime
from pathlib import Path

# Ensure logs directory exists
Path("logs").mkdir(exist_ok=True)

def generate_security_log():
    """Generate realistic security log entries"""
    
    log_types = [
        {
            "event_type": "login_attempt",
            "severity": "info",
            "success": random.choice([True, False]),
            "user": f"user{random.randint(1, 100)}",
            "ip_address": f"192.168.1.{random.randint(1, 254)}"
        },
        {
            "event_type": "api_access",
            "severity": "info",
            "endpoint": random.choice(["/api/v1/users", "/api/v1/data", "/api/v1/admin"]),
            "method": random.choice(["GET", "POST", "PUT", "DELETE"]),
            "status_code": random.choice([200, 401, 403, 404, 500]),
            "ip_address": f"10.0.0.{random.randint(1, 254)}"
        },
        {
            "event_type": "security_alert",
            "severity": random.choice(["warning", "critical"]),
            "alert_type": random.choice(["brute_force", "suspicious_activity", "privilege_escalation"]),
            "details": "Automated security alert triggered"
        }
    ]
    
    log_entry = random.choice(log_types)
    log_entry.update({
        "timestamp": datetime.datetime.now().isoformat(),
        "hostname": "security-lab-host",
        "service": "security-monitor"
    })
    
    return log_entry

def main():
    print("Starting security log generator...")
    
    while True:
        # Generate log entry
        log_entry = generate_security_log()
        
        # Write to log file
        with open("logs/security.log", "a") as f:
            f.write(json.dumps(log_entry) + "\n")
        
        # Print to console for immediate feedback
        print(f"[{log_entry['timestamp']}] {log_entry['event_type']}: {log_entry.get('severity', 'info')}")
        
        # Wait before next log entry
        time.sleep(random.uniform(1, 3))

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nLog generator stopped.")
EOF

# Start log generator in background
python3 log_generator.py &
LOG_GEN_PID=$!
echo "Log generator started with PID: $LOG_GEN_PID"
Subtask 4.2: Create Log Analysis Scripts
Create scripts to analyze security logs for potential incidents.

# Create log analysis script
cat > analyze_logs.py << 'EOF'
#!/usr/bin/env python3

import json
import sys
from collections import defaultdict, Counter
from datetime import datetime, timedelta

def analyze_security_logs(log_file):
    """Analyze security logs for potential threats"""
    
    print("=== Security Log Analysis Report ===\n")
    
    # Counters for analysis
    event_counts = Counter()
    failed_logins = defaultdict(list)
    api_errors = defaultdict(list)
    security_alerts = []
    
    try:
        with open(log_file, 'r') as f:
            for line in f:
                try:
                    log_entry = json.loads(line.strip())
                    event_type = log_entry.get('event_type', 'unknown')
                    event_counts[event_type] += 1
                    
                    # Analyze login attempts
                    if event_type == 'login_attempt' and not log_entry.get('success', True):
                        ip = log_entry.get('ip_address', 'unknown')
                        failed_logins[ip].append(log_entry)
                    
                    # Analyze API access
                    elif event_type == 'api_access' and log_entry.get('status_code', 200) >= 400:
                        ip = log_entry.get('ip_address', 'unknown')
                        api_errors[ip].append(log_entry)
                    
                    # Collect security alerts
                    elif event_type == 'security_alert':
                        security_alerts.append(log_entry)
                        
                except json.JSONDecodeError:
                    continue
                    
    except FileNotFoundError:
        print(f"Log file {log_file} not found!")
        return
    
    # Print analysis results
    print("1. Event Summary:")
    for event_type, count in event_counts.most_common():
        print(f"   {event_type}: {count}")
    
    print("\n2. Failed Login Analysis:")
    suspicious_ips = []
    for ip, attempts in failed_logins.items():
        if len(attempts) >= 3:  # 3 or more failed attempts
            print(f"   SUSPICIOUS: {ip} - {len(attempts)} failed login attempts")
            suspicious_ips.append(ip)
        else:
            print(f"   {ip} - {len(attempts)} failed login attempts")
    
    print("\n3. API Error Analysis:")
    for ip, errors in api_errors.items():
        error_codes = Counter(e.get('status_code') for e in errors)
        print(f"   {ip}: {dict(error_codes)}")
    
    print("\n4. Security Alerts:")
    if security_alerts:
        for alert in security_alerts[-5:]:  # Show last 5 alerts
            print(f"   [{alert['timestamp']}] {alert['severity'].upper()}: {alert['alert_type']}")
    else:
        print("   No security alerts found")
    
    print("\n5. Recommendations:")
    if suspicious_ips:
        print("   - Consider blocking or monitoring these IPs:", ", ".join(suspicious_ips))
    if len(api_errors) > 5:
        print("   - High number of API errors detected - investigate application issues")
    if any(alert['severity'] == 'critical' for alert in security_alerts):
        print("   - CRITICAL alerts detected - immediate investigation required")
    
    print("\n=== End of Analysis ===")

if __name__ == "__main__":
    log_file = sys.argv[1] if len(sys.argv) > 1 else "logs/security.log"
    analyze_security_logs(log_file)
EOF

# Make analysis script executable
chmod +x analyze_logs.py
Subtask 4.3: Create Real-time Log Monitoring
Create a real-time log monitoring script.

# Create real-time monitoring script
cat > monitor_logs.py << 'EOF'
#!/usr/bin/env python3

import json
import time
import subprocess
from collections import defaultdict

class SecurityLogMonitor:
    def __init__(self, log_file):
        self.log_file = log_file
        self.failed_login_count = defaultdict(int)
        self.alert_threshold = 5  # Alert after 5 failed attempts
        
    def process_log_line(self, line):
        """Process a single log line and check for security issues"""
        try:
            log_entry = json.loads(line.strip())
            
            # Monitor failed logins
            if (log_entry.get('event_type') == 'login_attempt' and 
                not log_entry.get('success', True)):
                
                ip = log_entry.get('ip_address', 'unknown')
                self.failed_login_count[ip] += 1
                
                print(f"[MONITOR] Failed login from {ip} (count: {self.failed_login_count[ip]})")
                
                if self.failed_login_count[ip] >= self.alert_threshold:
                    self.trigger_alert(f"BRUTE FORCE DETECTED: {ip} has {self.failed_login_count[ip]} failed attempts")
            
            # Monitor critical security alerts
            elif (log_entry.get('event_type') == 'security_alert' and 
                  log_entry.get('severity') == 'critical'):
                
                self.trigger_alert(f"CRITICAL SECURITY ALERT: {log_entry.get('alert_type', 'unknown')}")
            
            # Monitor suspicious API access
            elif (log_entry.get('event_type') == 'api_access' and 
                  log_entry.get('status_code') == 403):
                
                ip = log_entry.get('ip_address', 'unknown')
                endpoint = log_entry.get('endpoint', 'unknown')
                print(f"[MONITOR] Forbidden access attempt: {ip} -> {endpoint}")
                
        except json.JSONDecodeError:
            pass
    
    def trigger_alert(self, message):
        """Trigger a security alert"""
        timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
        alert_msg = f"[{timestamp}] SECURITY ALERT: {message}"
        print(f"\033[91m{alert_msg}\033[0m")  # Red color for alerts
        
        # Log alert to separate file
        with open("logs/security_alerts.log", "a") as f:
            f.write(alert_msg + "\n")
    
    def start_monitoring(self):
        """Start real-time log monitoring"""
        print(f"Starting real-time monitoring of {self.log_file}")
        print("Press Ctrl+C to stop monitoring\n")
        
        try:
            # Use tail -f to follow log file
            process = subprocess.Popen(['tail', '-f', self.log_file], 
                                     stdout=subprocess.PIPE, 
                                     stderr=subprocess.PIPE,
                                     universal_newlines=True)
            
            for line in iter(process.stdout.readline, ''):
                if line:
                    self.process_log_line(line)
                    
        except KeyboardInterrupt:
            print("\nMonitoring stopped.")
            process.terminate()
        except FileNotFoundError:
            print(f"Log file {self.log_file} not found. Make sure log generator is running.")

if __name__ == "__main__":
    monitor = SecurityLogMonitor("logs/security.log")
    monitor.start_monitoring()
EOF

# Make monitoring script executable
chmod +x monitor_logs.py
Task 5: Investigate Security Incidents Using Observability Tools
Subtask 5.1: Simulate a Security Incident
Let's create a realistic security incident to investigate.

# Create incident simulation script
cat > simulate_incident.py << 'EOF'
#!/usr/bin/env python3

import json
import time
import requests
import threading
from datetime import datetime

def simulate_brute_force_attack():
    """Simulate a brute force attack"""
    print("Simulating brute force attack...")
    
    target_ip = "192.168.1.100"
    
    # Generate multiple failed login attempts
    for i in range(15):
        log_entry = {
            "timestamp": datetime.now().isoformat(),
            "event_type": "login_attempt",
            "severity": "warning",
            "success": False,
            "user": f"admin",
            "ip_address": target_ip,
            "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        }
        
        with open("logs/security.log", "a") as f:
            f.write(json.dumps(log_entry) + "\n")
        
        time.sleep(0.5)
    
    print(f"Brute force simulation complete - {target_ip}")

def simulate_api_abuse():
    """Simulate API abuse/DDoS"""
    print("Simulating API abuse
