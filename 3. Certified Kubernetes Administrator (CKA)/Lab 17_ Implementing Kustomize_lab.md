Lab 17: Implementing Kustomize
Objectives
By the end of this lab, you will be able to:

• Understand the fundamentals of Kustomize and its role in Kubernetes configuration management • Create a base configuration for a Kubernetes deployment using Kustomize • Implement overlays to customize configurations for different environments • Deploy applications to staging and production environments using Kustomize • Validate and troubleshoot Kustomize-based deployments • Apply best practices for managing multi-environment Kubernetes configurations

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Deployments, Services) • Familiarity with YAML syntax and structure • Experience with kubectl command-line tool • Understanding of containerization concepts • Basic knowledge of Linux command line operations

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install additional software.

Your lab environment includes: • Ubuntu 20.04 LTS with kubectl pre-installed • Kustomize integrated with kubectl (version 1.28+) • A local Kubernetes cluster (kind) ready for use • Text editors (nano, vim) for file editing

Task 1: Understanding Kustomize Fundamentals
Subtask 1.1: Verify Kustomize Installation
First, let's verify that Kustomize is available in your environment.

# Check kubectl version (Kustomize is built into kubectl 1.14+)
kubectl version --client

# Verify Kustomize functionality
kubectl kustomize --help
Subtask 1.2: Create Project Directory Structure
Create a well-organized directory structure for your Kustomize project.

# Create the main project directory
mkdir -p ~/kustomize-lab
cd ~/kustomize-lab

# Create directory structure for base and overlays
mkdir -p base
mkdir -p overlays/staging
mkdir -p overlays/production

# Verify the structure
tree . || ls -la
Task 2: Creating Base Configuration
Subtask 2.1: Create Base Deployment Configuration
Create the base deployment configuration that will be shared across environments.

# Navigate to the base directory
cd ~/kustomize-lab/base

# Create the deployment YAML file
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
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
Subtask 2.2: Create Base Service Configuration
Create a service to expose the application.

# Create the service YAML file
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  labels:
    app: web-app
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF
Subtask 2.3: Create Base ConfigMap
Create a ConfigMap for application configuration.

# Create the configmap YAML file
cat > configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
data:
  environment: "base"
  log_level: "info"
  database_host: "localhost"
  database_port: "5432"
EOF
Subtask 2.4: Create Base Kustomization File
Create the main kustomization file that defines the base configuration.

# Create the kustomization.yaml file
cat > kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  name: web-app-base

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  version: v1.0.0
  managed-by: kustomize

commonAnnotations:
  description: "Web application base configuration"
EOF
Subtask 2.5: Validate Base Configuration
Test the base configuration to ensure it's valid.

# Generate and view the base configuration
kubectl kustomize .

# Verify the output contains all expected resources
kubectl kustomize . | grep -E "^kind:"
Task 3: Creating Staging Overlay
Subtask 3.1: Create Staging Kustomization
Navigate to the staging overlay directory and create the staging-specific configuration.

# Navigate to staging overlay directory
cd ~/kustomize-lab/overlays/staging

# Create staging kustomization file
cat > kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  name: web-app-staging

namespace: staging

resources:
- ../../base

namePrefix: staging-

commonLabels:
  environment: staging
  tier: non-production

patchesStrategicMerge:
- deployment-patch.yaml
- configmap-patch.yaml

replicas:
- name: web-app
  count: 1
EOF
Subtask 3.2: Create Staging Deployment Patch
Create a patch file to customize the deployment for staging.

# Create deployment patch for staging
cat > deployment-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: web-app
        image: nginx:1.21-alpine
        env:
        - name: ENVIRONMENT
          value: "staging"
        resources:
          requests:
            memory: "32Mi"
            cpu: "100m"
          limits:
            memory: "64Mi"
            cpu: "200m"
EOF
Subtask 3.3: Create Staging ConfigMap Patch
Create a patch to modify the ConfigMap for staging environment.

# Create configmap patch for staging
cat > configmap-patch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
data:
  environment: "staging"
  log_level: "debug"
  database_host: "staging-db.example.com"
  database_port: "5432"
  debug_mode: "true"
EOF
Subtask 3.4: Create Staging Namespace
Create a namespace definition for staging.

# Create namespace file
cat > namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    managed-by: kustomize
EOF

# Add namespace to kustomization resources
cat >> kustomization.yaml << 'EOF'

- namespace.yaml
EOF
Subtask 3.5: Validate Staging Configuration
Test the staging overlay configuration.

# Generate and view the staging configuration
kubectl kustomize .

# Check that staging-specific changes are applied
kubectl kustomize . | grep -A 5 -B 5 "staging"
Task 4: Creating Production Overlay
Subtask 4.1: Create Production Kustomization
Navigate to the production overlay directory and create production-specific configuration.

# Navigate to production overlay directory
cd ~/kustomize-lab/overlays/production

# Create production kustomization file
cat > kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  name: web-app-production

namespace: production

resources:
- ../../base
- namespace.yaml

namePrefix: prod-

commonLabels:
  environment: production
  tier: production

patchesStrategicMerge:
- deployment-patch.yaml
- configmap-patch.yaml
- service-patch.yaml

replicas:
- name: web-app
  count: 3

images:
- name: nginx
  newTag: "1.21"
EOF
Subtask 4.2: Create Production Deployment Patch
Create a patch file with production-specific deployment settings.

# Create deployment patch for production
cat > deployment-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: web-app
        image: nginx:1.21
        env:
        - name: ENVIRONMENT
          value: "production"
        resources:
          requests:
            memory: "128Mi"
            cpu: "500m"
          limits:
            memory: "256Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
Subtask 4.3: Create Production ConfigMap Patch
Create production-specific configuration values.

# Create configmap patch for production
cat > configmap-patch.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
data:
  environment: "production"
  log_level: "warn"
  database_host: "prod-db.example.com"
  database_port: "5432"
  debug_mode: "false"
  cache_enabled: "true"
  max_connections: "100"
EOF
Subtask 4.4: Create Production Service Patch
Modify the service for production requirements.

# Create service patch for production
cat > service-patch.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  - protocol: TCP
    port: 443
    targetPort: 80
    name: https
EOF
Subtask 4.5: Create Production Namespace
Create the production namespace.

# Create namespace file
cat > namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    managed-by: kustomize
  annotations:
    description: "Production environment for web application"
EOF
Subtask 4.6: Validate Production Configuration
Test the production overlay configuration.

# Generate and view the production configuration
kubectl kustomize .

# Verify production-specific settings
kubectl kustomize . | grep -E "(replicas|environment|prod-)"
Task 5: Deploying to Different Environments
Subtask 5.1: Deploy to Staging Environment
Deploy the application to the staging environment.

# Navigate to staging overlay
cd ~/kustomize-lab/overlays/staging

# Apply the staging configuration
kubectl apply -k .

# Verify the deployment
kubectl get all -n staging

# Check the ConfigMap values
kubectl get configmap -n staging staging-web-app-config -o yaml
Subtask 5.2: Deploy to Production Environment
Deploy the application to the production environment.

# Navigate to production overlay
cd ~/kustomize-lab/overlays/production

# Apply the production configuration
kubectl apply -k .

# Verify the deployment
kubectl get all -n production

# Check that 3 replicas are running
kubectl get deployment -n production prod-web-app
Subtask 5.3: Compare Deployments
Compare the configurations between environments.

# Compare replica counts
echo "Staging replicas:"
kubectl get deployment -n staging staging-web-app -o jsonpath='{.spec.replicas}'
echo ""

echo "Production replicas:"
kubectl get deployment -n production prod-web-app -o jsonpath='{.spec.replicas}'
echo ""

# Compare resource limits
echo "Staging resource limits:"
kubectl get deployment -n staging staging-web-app -o jsonpath='{.spec.template.spec.containers[0].resources.limits}'
echo ""

echo "Production resource limits:"
kubectl get deployment -n production prod-web-app -o jsonpath='{.spec.template.spec.containers[0].resources.limits}'
echo ""
Task 6: Validation and Testing
Subtask 6.1: Test Application Connectivity
Test that the applications are running correctly in both environments.

# Test staging application
kubectl port-forward -n staging service/staging-web-app-service 8080:80 &
STAGING_PID=$!

# Test the staging endpoint (in a new terminal or after a few seconds)
curl http://localhost:8080 || echo "Staging app is running"

# Stop port forwarding
kill $STAGING_PID

# Test production application
kubectl port-forward -n production service/prod-web-app-service 8081:80 &
PROD_PID=$!

# Test the production endpoint
curl http://localhost:8081 || echo "Production app is running"

# Stop port forwarding
kill $PROD_PID
Subtask 6.2: Validate Environment-Specific Configurations
Verify that environment-specific configurations are correctly applied.

# Check staging environment variables
kubectl exec -n staging deployment/staging-web-app -- env | grep ENVIRONMENT

# Check production environment variables
kubectl exec -n production deployment/prod-web-app -- env | grep ENVIRONMENT

# Verify ConfigMap differences
echo "Staging ConfigMap:"
kubectl get configmap -n staging staging-web-app-config -o jsonpath='{.data.environment}'
echo ""

echo "Production ConfigMap:"
kubectl get configmap -n production prod-web-app-config -o jsonpath='{.data.environment}'
echo ""
Subtask 6.3: Test Configuration Updates
Test updating configurations using Kustomize.

# Navigate to base directory
cd ~/kustomize-lab/base

# Update the base image version
sed -i 's/nginx:1.21/nginx:1.22/g' deployment.yaml

# Apply updates to both environments
cd ~/kustomize-lab/overlays/staging
kubectl apply -k .

cd ~/kustomize-lab/overlays/production
kubectl apply -k .

# Verify the image updates
kubectl get deployment -n staging staging-web-app -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""
kubectl get deployment -n production prod-web-app -o jsonpath='{.spec.template.spec.containers[0].image}'
echo ""
Task 7: Advanced Kustomize Features
Subtask 7.1: Using Generators
Create a Secret using Kustomize generators.

# Navigate to base directory
cd ~/kustomize-lab/base

# Add secret generator to kustomization.yaml
cat >> kustomization.yaml << 'EOF'

secretGenerator:
- name: web-app-secret
  literals:
  - username=admin
  - password=defaultpass
EOF

# Test the generator
kubectl kustomize .
Subtask 7.2: Using Transformers
Add resource transformers to modify resources.

# Navigate to production overlay
cd ~/kustomize-lab/overlays/production

# Add transformer to kustomization.yaml
cat >> kustomization.yaml << 'EOF'

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: web-app
  path: cpu-limit-patch.json
EOF

# Create JSON patch file
cat > cpu-limit-patch.json << 'EOF'
[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/resources/limits/cpu",
    "value": "2000m"
  }
]
EOF

# Apply and verify the patch
kubectl apply -k .
kubectl get deployment -n production prod-web-app -o jsonpath='{.spec.template.spec.containers[0].resources.limits.cpu}'
echo ""
Troubleshooting Common Issues
Issue 1: Kustomization File Not Found
Problem: Error message "unable to find kustomization file"

Solution:

# Ensure you're in the correct directory
pwd
ls -la kustomization.yaml

# Check file name spelling (must be exactly "kustomization.yaml")
Issue 2: Resource Not Found in Base
Problem: Error about missing resources in base directory

Solution:

# Verify base directory structure
cd ~/kustomize-lab/base
ls -la

# Check that all resources listed in kustomization.yaml exist
kubectl kustomize . --dry-run
Issue 3: Patch Not Applied
Problem: Strategic merge patches not working as expected

Solution:

# Verify patch file syntax
kubectl kustomize overlays/staging --dry-run

# Check that patch targets match exactly with base resources
# Ensure metadata names and kinds match
Lab Cleanup
Clean up the resources created during this lab.

# Delete staging resources
kubectl delete -k ~/kustomize-lab/overlays/staging

# Delete production resources
kubectl delete -k ~/kustomize-lab/overlays/production

# Delete namespaces
kubectl delete namespace staging production

# Verify cleanup
kubectl get all --all-namespaces | grep -E "(staging|production)"
Conclusion
Congratulations! You have successfully completed Lab 17: Implementing Kustomize. In this lab, you have accomplished the following:

Key Achievements: • Mastered Kustomize Fundamentals: You learned how Kustomize works as a template-free way to customize Kubernetes configurations • Created Base Configurations: You built reusable base configurations that serve as the foundation for multiple environments • Implemented Environment Overlays: You created staging and production overlays that customize the base configuration for specific environments • Applied Configuration Management Best Practices: You organized your configurations in a maintainable directory structure • Deployed Multi-Environment Applications: You successfully deployed the same application with different configurations to staging and production • Validated Environment-Specific Settings: You verified that each environment received its appropriate configuration values • Explored Advanced Features: You experimented with generators, transformers, and JSON patches

Why This Matters: Kustomize is a powerful tool that solves real-world problems in Kubernetes configuration management. Instead of maintaining separate YAML files for each environment (which leads to duplication and inconsistency), Kustomize allows you to define a base configuration once and then apply environment-specific customizations through overlays. This approach reduces configuration drift, makes updates easier to manage, and follows the DRY (Don't Repeat Yourself) principle.

Real-World Applications: • DevOps Teams use Kustomize to manage applications across development, staging, and production environments • Platform Engineers leverage Kustomize to provide standardized deployment patterns while allowing customization • Application Developers benefit from simplified deployment processes without needing to understand complex templating systems • Organizations achieve better compliance and governance through consistent base configurations

Next Steps: • Explore Kustomize with GitOps workflows using tools like ArgoCD or Flux • Learn about Kustomize components for even more modular configurations • Practice with more complex scenarios involving multiple applications and shared resources • Integrate Kustomize into CI/CD pipelines for automated deployments

This lab has provided you with practical, hands-on experience with Kustomize that directly applies to the Certified Kubernetes Application Developer (CKAD) certification and real-world Kubernetes operations. The skills you've developed here will help you manage complex, multi-environment Kubernetes deployments with confidence and efficiency.
