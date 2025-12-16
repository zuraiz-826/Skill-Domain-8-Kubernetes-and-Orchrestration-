Lab 7: Supply Chain Security
Objectives
By the end of this lab, students will be able to:

• Understand the importance of supply chain security in containerized environments • Integrate container image vulnerability scanning using Trivy into a CI/CD pipeline • Sign container images using Cosign for authenticity verification • Configure Kubernetes admission controllers to enforce image signature verification • Deploy only signed and verified container images to a Kubernetes cluster • Implement security policies that prevent deployment of unsigned or vulnerable images

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Docker containers and container images • Familiarity with Kubernetes concepts (pods, deployments, services) • Knowledge of CI/CD pipeline concepts • Basic Linux command-line experience • Understanding of Git version control • Familiarity with YAML configuration files

Lab Environment
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to begin - no need to build your own VM or install software.

Your lab environment includes: • Ubuntu 22.04 LTS with Docker installed • Kubernetes cluster (kind) pre-configured • Git, curl, and other essential tools • Internet access for downloading tools and images

Task 1: Setting Up the Lab Environment
Subtask 1.1: Verify Environment and Install Required Tools
First, let's verify our environment and install the necessary tools for supply chain security.

Check Docker installation:
docker --version
docker info
Verify Kubernetes cluster:
kubectl cluster-info
kubectl get nodes
Install Trivy (Container Vulnerability Scanner):
# Add Trivy repository
sudo apt-get update
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list

# Install Trivy
sudo apt-get update
sudo apt-get install trivy -y

# Verify installation
trivy --version
Install Cosign (Container Signing Tool):
# Download and install Cosign
COSIGN_VERSION="v2.2.2"
wget "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# Verify installation
cosign version
Install additional tools:
# Install jq for JSON processing
sudo apt-get install jq -y

# Install crane for container image manipulation
GO_VERSION="1.21.5"
wget "https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz"
sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
export PATH=$PATH:/usr/local/go/bin
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc

# Install crane
go install github.com/google/go-containerregistry/cmd/crane@latest
sudo cp ~/go/bin/crane /usr/local/bin/
Subtask 1.2: Create Working Directory Structure
Create a structured workspace for our supply chain security lab:

# Create main lab directory
mkdir -p ~/supply-chain-lab
cd ~/supply-chain-lab

# Create subdirectories
mkdir -p {app,pipeline,policies,keys,manifests}

# Create a simple application directory structure
mkdir -p app/src
Task 2: Creating a Sample Application and Container Image
Subtask 2.1: Build a Sample Web Application
Let's create a simple web application that we'll use throughout this lab.

Create application source code:
cd ~/supply-chain-lab/app/src

# Create a simple Node.js application
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Supply Chain Security Lab Application',
    version: '1.0.0',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App listening at http://0.0.0.0:${port}`);
});
EOF

# Create package.json
cat > package.json << 'EOF'
{
  "name": "supply-chain-app",
  "version": "1.0.0",
  "description": "Sample app for supply chain security lab",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
Create Dockerfile:
cat > Dockerfile << 'EOF'
FROM node:18-alpine

# Create app directory
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY app.js .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

# Start application
CMD ["npm", "start"]
EOF
Subtask 2.2: Build and Test the Container Image
Build the container image:
cd ~/supply-chain-lab/app/src
docker build -t supply-chain-app:v1.0.0 .
Test the application locally:
# Run the container
docker run -d --name test-app -p 3000:3000 supply-chain-app:v1.0.0

# Wait a moment for startup
sleep 5

# Test the application
curl http://localhost:3000
curl http://localhost:3000/health

# Stop and remove test container
docker stop test-app
docker rm test-app
Task 3: Implementing Container Image Vulnerability Scanning
Subtask 3.1: Basic Trivy Scanning
Learn how to use Trivy to scan container images for vulnerabilities.

Scan the container image for vulnerabilities:
cd ~/supply-chain-lab

# Basic vulnerability scan
trivy image supply-chain-app:v1.0.0

# Scan with specific severity levels
trivy image --severity HIGH,CRITICAL supply-chain-app:v1.0.0

# Generate JSON report
trivy image --format json --output scan-report.json supply-chain-app:v1.0.0

# View the JSON report
jq '.Results[0].Vulnerabilities | length' scan-report.json
Create a vulnerability scanning script:
cd ~/supply-chain-lab/pipeline

cat > scan-image.sh << 'EOF'
#!/bin/bash

IMAGE_NAME=$1
SEVERITY_THRESHOLD=${2:-HIGH,CRITICAL}
OUTPUT_FORMAT=${3:-table}

if [ -z "$IMAGE_NAME" ]; then
    echo "Usage: $0 <image_name> [severity_threshold] [output_format]"
    echo "Example: $0 myapp:latest HIGH,CRITICAL json"
    exit 1
fi

echo "Scanning image: $IMAGE_NAME"
echo "Severity threshold: $SEVERITY_THRESHOLD"
echo "Output format: $OUTPUT_FORMAT"

# Create reports directory
mkdir -p ../reports

# Perform scan
if [ "$OUTPUT_FORMAT" = "json" ]; then
    trivy image --severity "$SEVERITY_THRESHOLD" --format json --output "../reports/$(basename $IMAGE_NAME)-scan.json" "$IMAGE_NAME"
    
    # Check if vulnerabilities were found
    VULN_COUNT=$(jq '[.Results[]?.Vulnerabilities // []] | add | length' "../reports/$(basename $IMAGE_NAME)-scan.json")
    echo "Found $VULN_COUNT vulnerabilities"
    
    if [ "$VULN_COUNT" -gt 0 ]; then
        echo "❌ Vulnerabilities found! Check the report for details."
        exit 1
    else
        echo "✅ No vulnerabilities found!"
        exit 0
    fi
else
    trivy image --severity "$SEVERITY_THRESHOLD" "$IMAGE_NAME"
fi
EOF

chmod +x scan-image.sh
Test the scanning script:
cd ~/supply-chain-lab/pipeline
./scan-image.sh supply-chain-app:v1.0.0 HIGH,CRITICAL json
Subtask 3.2: Advanced Trivy Configuration
Create more sophisticated scanning configurations for different use cases.

Create Trivy configuration file:
cd ~/supply-chain-lab

cat > .trivyignore << 'EOF'
# Ignore specific CVEs that are false positives or accepted risks
# CVE-2023-12345

# Ignore vulnerabilities in specific paths
/usr/share/doc/**
/var/lib/dpkg/**
EOF

cat > trivy.yaml << 'EOF'
# Trivy configuration file
format: json
output: reports/detailed-scan.json
severity:
  - HIGH
  - CRITICAL
vulnerability:
  type:
    - os
    - library
ignore-unfixed: true
exit-code: 1
timeout: 10m
EOF
Create comprehensive scanning script:
cd ~/supply-chain-lab/pipeline

cat > comprehensive-scan.sh << 'EOF'
#!/bin/bash

set -e

IMAGE_NAME=$1
if [ -z "$IMAGE_NAME" ]; then
    echo "Usage: $0 <image_name>"
    exit 1
fi

echo "🔍 Starting comprehensive security scan for: $IMAGE_NAME"

# Create reports directory
mkdir -p ../reports

# 1. Vulnerability scan
echo "📋 Running vulnerability scan..."
trivy image --config ../trivy.yaml "$IMAGE_NAME"

# 2. Secret scan
echo "🔐 Scanning for secrets..."
trivy image --scanners secret --format json --output "../reports/secrets-$(basename $IMAGE_NAME).json" "$IMAGE_NAME"

# 3. Configuration scan
echo "⚙️  Scanning configuration..."
trivy image --scanners config --format json --output "../reports/config-$(basename $IMAGE_NAME).json" "$IMAGE_NAME"

# 4. Generate summary report
echo "📊 Generating summary report..."
cat > "../reports/scan-summary-$(basename $IMAGE_NAME).txt" << EOL
Security Scan Summary for: $IMAGE_NAME
Scan Date: $(date)
===========================================

Vulnerability Scan: $([ -f "../reports/detailed-scan.json" ] && echo "✅ Completed" || echo "❌ Failed")
Secret Scan: $([ -f "../reports/secrets-$(basename $IMAGE_NAME).json" ] && echo "✅ Completed" || echo "❌ Failed")
Configuration Scan: $([ -f "../reports/config-$(basename $IMAGE_NAME).json" ] && echo "✅ Completed" || echo "❌ Failed")

EOL

echo "✅ Comprehensive scan completed! Check reports directory for details."
EOF

chmod +x comprehensive-scan.sh
Run comprehensive scan:
cd ~/supply-chain-lab/pipeline
./comprehensive-scan.sh supply-chain-app:v1.0.0

# View the summary
cat ../reports/scan-summary-supply-chain-app:v1.0.0.txt
Task 4: Container Image Signing with Cosign
Subtask 4.1: Generate Signing Keys
Set up cryptographic keys for signing container images.

Generate Cosign key pair:
cd ~/supply-chain-lab/keys

# Generate key pair (you'll be prompted for a password)
cosign generate-key-pair

# The above command creates:
# - cosign.key (private key)
# - cosign.pub (public key)

echo "🔑 Key pair generated successfully!"
ls -la cosign.*
Create key management script:
cd ~/supply-chain-lab/pipeline

cat > manage-keys.sh << 'EOF'
#!/bin/bash

KEYS_DIR="../keys"

case "$1" in
    "generate")
        echo "🔑 Generating new key pair..."
        cd "$KEYS_DIR"
        cosign generate-key-pair
        echo "✅ Key pair generated in $KEYS_DIR"
        ;;
    "verify-keys")
        echo "🔍 Verifying key pair..."
        if [ -f "$KEYS_DIR/cosign.key" ] && [ -f "$KEYS_DIR/cosign.pub" ]; then
            echo "✅ Both private and public keys found"
            echo "Private key: $KEYS_DIR/cosign.key"
            echo "Public key: $KEYS_DIR/cosign.pub"
        else
            echo "❌ Key pair not found or incomplete"
            exit 1
        fi
        ;;
    "backup-keys")
        echo "💾 Creating key backup..."
        BACKUP_DIR="../keys/backup-$(date +%Y%m%d-%H%M%S)"
        mkdir -p "$BACKUP_DIR"
        cp "$KEYS_DIR/cosign.key" "$KEYS_DIR/cosign.pub" "$BACKUP_DIR/"
        echo "✅ Keys backed up to $BACKUP_DIR"
        ;;
    *)
        echo "Usage: $0 {generate|verify-keys|backup-keys}"
        exit 1
        ;;
esac
EOF

chmod +x manage-keys.sh
Subtask 4.2: Sign Container Images
Learn how to sign container images using Cosign.

Create image signing script:
cd ~/supply-chain-lab/pipeline

cat > sign-image.sh << 'EOF'
#!/bin/bash

set -e

IMAGE_NAME=$1
PRIVATE_KEY_PATH="../keys/cosign.key"

if [ -z "$IMAGE_NAME" ]; then
    echo "Usage: $0 <image_name>"
    echo "Example: $0 supply-chain-app:v1.0.0"
    exit 1
fi

if [ ! -f "$PRIVATE_KEY_PATH" ]; then
    echo "❌ Private key not found at $PRIVATE_KEY_PATH"
    echo "Run './manage-keys.sh generate' first"
    exit 1
fi

echo "✍️  Signing image: $IMAGE_NAME"

# Sign the image
cosign sign --key "$PRIVATE_KEY_PATH" "$IMAGE_NAME"

echo "✅ Image signed successfully!"

# Verify the signature immediately
echo "🔍 Verifying signature..."
cosign verify --key "../keys/cosign.pub" "$IMAGE_NAME"

echo "✅ Signature verification successful!"
EOF

chmod +x sign-image.sh
Sign the container image:
cd ~/supply-chain-lab/pipeline

# Sign the image (you'll be prompted for the private key password)
./sign-image.sh supply-chain-app:v1.0.0
Create signature verification script:
cd ~/supply-chain-lab/pipeline

cat > verify-signature.sh << 'EOF'
#!/bin/bash

IMAGE_NAME=$1
PUBLIC_KEY_PATH="../keys/cosign.pub"

if [ -z "$IMAGE_NAME" ]; then
    echo "Usage: $0 <image_name>"
    exit 1
fi

if [ ! -f "$PUBLIC_KEY_PATH" ]; then
    echo "❌ Public key not found at $PUBLIC_KEY_PATH"
    exit 1
fi

echo "🔍 Verifying signature for: $IMAGE_NAME"

if cosign verify --key "$PUBLIC_KEY_PATH" "$IMAGE_NAME" > /dev/null 2>&1; then
    echo "✅ Signature verification successful!"
    echo "📋 Signature details:"
    cosign verify --key "$PUBLIC_KEY_PATH" "$IMAGE_NAME"
    exit 0
else
    echo "❌ Signature verification failed!"
    exit 1
fi
EOF

chmod +x verify-signature.sh
Subtask 4.3: Advanced Signing with Annotations
Add metadata and annotations to signed images.

Create advanced signing script:
cd ~/supply-chain-lab/pipeline

cat > sign-with-metadata.sh << 'EOF'
#!/bin/bash

set -e

IMAGE_NAME=$1
BUILD_ID=${2:-$(date +%Y%m%d-%H%M%S)}
GIT_COMMIT=${3:-"unknown"}
PRIVATE_KEY_PATH="../keys/cosign.key"

if [ -z "$IMAGE_NAME" ]; then
    echo "Usage: $0 <image_name> [build_id] [git_commit]"
    exit 1
fi

echo "✍️  Signing image with metadata: $IMAGE_NAME"

# Create attestation data
cat > ../reports/build-metadata.json << EOL
{
  "buildId": "$BUILD_ID",
  "gitCommit": "$GIT_COMMIT",
  "buildTime": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "builder": "supply-chain-lab",
  "scanStatus": "passed"
}
EOL

# Sign with annotations
cosign sign --key "$PRIVATE_KEY_PATH" \
    -a "build-id=$BUILD_ID" \
    -a "git-commit=$GIT_COMMIT" \
    -a "scan-status=passed" \
    "$IMAGE_NAME"

# Attach attestation
cosign attest --key "$PRIVATE_KEY_PATH" \
    --predicate ../reports/build-metadata.json \
    "$IMAGE_NAME"

echo "✅ Image signed with metadata and attestation!"
EOF

chmod +x sign-with-metadata.sh
Test advanced signing:
cd ~/supply-chain-lab/pipeline
./sign-with-metadata.sh supply-chain-app:v1.0.0 "build-001" "abc123def"
Task 5: Creating a Secure CI/CD Pipeline
Subtask 5.1: Build Pipeline Script
Create a comprehensive CI/CD pipeline that includes scanning and signing.

Create main pipeline script:
cd ~/supply-chain-lab/pipeline

cat > secure-pipeline.sh << 'EOF'
#!/bin/bash

set -e

# Configuration
IMAGE_NAME=${1:-"supply-chain-app"}
IMAGE_TAG=${2:-"v1.0.0"}
FULL_IMAGE_NAME="$IMAGE_NAME:$IMAGE_TAG"
BUILD_ID=$(date +%Y%m%d-%H%M%S)

echo "🚀 Starting Secure CI/CD Pipeline"
echo "=================================="
echo "Image: $FULL_IMAGE_NAME"
echo "Build ID: $BUILD_ID"
echo "=================================="

# Step 1: Build the image
echo "📦 Step 1: Building container image..."
cd ../app/src
docker build -t "$FULL_IMAGE_NAME" .
echo "✅ Image built successfully"

# Step 2: Scan for vulnerabilities
echo "🔍 Step 2: Scanning for vulnerabilities..."
cd ../../pipeline
if ./comprehensive-scan.sh "$FULL_IMAGE_NAME"; then
    echo "✅ Security scan passed"
else
    echo "❌ Security scan failed - stopping pipeline"
    exit 1
fi

# Step 3: Sign the image
echo "✍️  Step 3: Signing container image..."
if ./sign-with-metadata.sh "$FULL_IMAGE_NAME" "$BUILD_ID" "pipeline-build"; then
    echo "✅ Image signed successfully"
else
    echo "❌ Image signing failed - stopping pipeline"
    exit 1
fi

# Step 4: Verify signature
echo "🔍 Step 4: Verifying image signature..."
if ./verify-signature.sh "$FULL_IMAGE_NAME"; then
    echo "✅ Signature verification passed"
else
    echo "❌ Signature verification failed - stopping pipeline"
    exit 1
fi

# Step 5: Generate pipeline report
echo "📊 Step 5: Generating pipeline report..."
cat > "../reports/pipeline-report-$BUILD_ID.txt" << EOL
Secure CI/CD Pipeline Report
============================
Build ID: $BUILD_ID
Image: $FULL_IMAGE_NAME
Pipeline Date: $(date)
Status: SUCCESS

Pipeline Steps:
✅ Image Build
✅ Vulnerability Scan
✅ Image Signing
✅ Signature Verification

Next Steps:
- Deploy to staging environment
- Run integration tests
- Deploy to production (if approved)
EOL

echo "✅ Pipeline completed successfully!"
echo "📋 Report saved to: reports/pipeline-report-$BUILD_ID.txt"
EOF

chmod +x secure-pipeline.sh
Run the secure pipeline:
cd ~/supply-chain-lab/pipeline
./secure-pipeline.sh supply-chain-app v1.0.1
Subtask 5.2: Create Pipeline Configuration Files
Set up configuration files for different environments.

Create environment-specific configurations:
cd ~/supply-chain-lab

mkdir -p config/{dev,staging,prod}

# Development configuration
cat > config/dev/pipeline.yaml << 'EOF'
environment: development
security:
  vulnerability_scan:
    enabled: true
    severity_threshold: ["HIGH", "CRITICAL"]
    fail_on_vulnerabilities: false
  image_signing:
    enabled: true
    required: false
  policy_enforcement:
    enabled: false
deployment:
  auto_deploy: true
  approval_required: false
EOF

# Staging configuration
cat > config/staging/pipeline.yaml << 'EOF'
environment: staging
security:
  vulnerability_scan:
    enabled: true
    severity_threshold: ["MEDIUM", "HIGH", "CRITICAL"]
    fail_on_vulnerabilities: true
  image_signing:
    enabled: true
    required: true
  policy_enforcement:
    enabled: true
deployment:
  auto_deploy: false
  approval_required: true
EOF

# Production configuration
cat > config/prod/pipeline.yaml << 'EOF'
environment: production
security:
  vulnerability_scan:
    enabled: true
    severity_threshold: ["LOW", "MEDIUM", "HIGH", "CRITICAL"]
    fail_on_vulnerabilities: true
  image_signing:
    enabled: true
    required: true
  policy_enforcement:
    enabled: true
    strict_mode: true
deployment:
  auto_deploy: false
  approval_required: true
  multi_approval: true
EOF
Task 6: Kubernetes Admission Control and Policy Enforcement
Subtask 6.1: Configure Image Policy Webhook
Set up Kubernetes to only allow signed images.

Create admission controller configuration:
cd ~/supply-chain-lab/policies

cat > cosign-policy.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cosign-policy
  namespace: cosign-system
data:
  policy.yaml: |
    apiVersion: v1alpha1
    kind: Policy
    metadata:
      name: image-signing-policy
    spec:
      requirements:
        - keyRef: cosign-public-key
          ctlogRef: rekor-public-key
      match:
        - version: v1
          resource: pods
        - version: v1
          resource: replicationcontrollers
        - version: v1
          group: apps
          resource: replicasets
        - version: v1
          group: apps
          resource: deployments
        - version: v1
          group: apps
          resource: daemonsets
        - version: v1
          group: apps
          resource: statefulsets
        - version: v1beta1
          group: batch
          resource: cronjobs
        - version: v1
          group: batch
          resource: jobs
EOF
Create namespace and RBAC:
cd ~/supply-chain-lab/policies

cat > namespace-rbac.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: cosign-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cosign-webhook
  namespace: cosign-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cosign-webhook
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["admissionregistration.k8s.io"]
  resources: ["validatingadmissionwebhooks", "mutatingadmissionwebhooks"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cosign-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cosign-webhook
subjects:
- kind: ServiceAccount
  name: cosign-webhook
  namespace: cosign-system
EOF

kubectl apply -f namespace-rbac.yaml
Create public key secret:
cd ~/supply-chain-lab/policies

# Create secret with public key
kubectl create secret generic cosign-public-key \
  --from-file=cosign.pub=../keys/cosign.pub \
  --namespace=cosign-system

# Verify secret creation
kubectl get secret cosign-public-key -n cosign-system -o yaml
Subtask 6.2: Deploy Policy Enforcement
Create a simple policy enforcement mechanism using ValidatingAdmissionWebhook.

Create policy enforcement script:
cd ~/supply-chain-lab/policies

cat > policy-webhook.py << 'EOF'
#!/usr/bin/env python3

import json
import base64
import subprocess
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler
import ssl

class PolicyWebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        body = self.rfile.read(content_length)
        
        try:
            admission_review = json.loads(body.decode('utf-8'))
            response = self.validate_admission(admission_review)
            
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps(response).encode('utf-8'))
            
        except Exception as e:
            print(f"Error processing request: {e}")
            self.send_response(500)
            self.end_headers()
    
    def validate_admission(self, admission_review):
        request = admission_review['request']
        allowed = True
        message = ""
        
        # Extract pod spec
        if 'object' in request and request['object']['kind'] == 'Pod':
            pod_spec = request['object']['spec']
            
            # Check all containers
            for container in pod_spec.get('containers', []):
                image = container['image']
                
                # Verify image signature
                if not self.verify_image_signature(image):
                    allowed = False
                    message = f"Image {image} is not signed or signature verification failed"
                    break
        
        return {
            'apiVersion': 'admission.k8s.io/v1',
            'kind': 'AdmissionReview',
            'response': {
                'uid': request['uid'],
                'allowed': allowed,
                'status': {'message': message} if message else {}
            }
        }
    
    def verify_image_signature(self, image):
        try:
            # Run cosign verify command
            result = subprocess.run([
                'cosign', 'verify', '--key', '/etc/cosign/cosign.pub', image
            ], capture_output=True, text=True, timeout=30)
            
            return result.returncode == 0
        except Exception as e:
            print(f"Error verifying image {image}: {e}")
            return False

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8443), PolicyWebhookHandler)
    print("Policy webhook server starting on port 8443...")
    server.serve_forever()
EOF

chmod +x policy-webhook.py
Create a simpler validation script for testing:
cd ~/supply-chain-lab/policies

cat > validate-deployment.sh << 'EOF'
#!/bin/bash

DEPLOYMENT_FILE=$1

if [ -z "$DEPLOYMENT_FILE" ]; then
    echo "Usage: $0 <deployment_file>"
    exit 1
fi

echo "🔍 Validating deployment: $DEPLOYMENT_FILE"

# Extract image names from deployment
IMAGES=$(kubectl get -f "$DEPLOYMENT_FILE" -o jsonpath='{.spec.template.spec.containers[*].image}' --dry-run=client)

if [ -z "$IMAGES" ]; then
    echo "❌ No images found in deployment"
    exit 1
fi

echo "📋 Found images: $IMAGES"

# Verify each image
for IMAGE in $IMAGES; do
    echo "🔍 Verifying signature for: $IMAGE"
    
    if cosign verify --key ../keys/cosign.pub "$IMAGE" > /dev/null 2>&1; then
        echo "✅ $IMAGE: Signature valid"
    else
        echo "❌ $IMAGE: Signature verification failed"
        echo "🚫 Deployment blocked due to unsigned image"
        exit 1
    fi
done

echo "✅ All images verified successfully"
echo "🚀 Deployment validation passed"
EOF

chmod +x validate-deployment.sh
Task 7: Deploying Signed Images to Kubernetes
Subtask 7.1: Create Kubernetes Manifests
Create deployment manifests for our signed application.

Create deployment manifest:
cd ~/supply-chain-lab/manifests

cat > app-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: supply-chain-app
  namespace: default
  labels:
    app: supply-chain-app
    security.policy/signed: "required"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: supply-chain-app
  template:
    metadata:
      labels:
        app: supply-chain-app
      annotations:
        security.policy/image-scan: "passed"
        security.policy/image-signed: "true"
    spec:
      containers:
      - name: app
        image: supply-chain-app:v1.0.1
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          runAsUser: 1001
          readOnlyRootFilesystem: false
          capabilities:
            drop:
            - ALL
---
apiVersion: v
