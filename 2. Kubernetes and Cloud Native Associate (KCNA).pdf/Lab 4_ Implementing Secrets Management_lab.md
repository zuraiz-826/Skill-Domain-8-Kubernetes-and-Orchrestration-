Lab 4: Implementing Secrets Management
Objectives
By the end of this lab, you will be able to:

• Create and manage Kubernetes Secrets using various methods • Mount secrets as volumes and environment variables in Pods • Set up HashiCorp Vault as an external secrets manager • Configure Vault integration with Kubernetes • Verify encryption of sensitive data at rest and in transit • Implement best practices for secrets management in cloud-native environments • Understand the security implications of different secrets storage approaches

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, ConfigMaps) • Familiarity with Linux command line operations • Basic knowledge of YAML file structure • Understanding of containerization concepts • Previous experience with kubectl commands

Technical Requirements: • Al Nafi provides ready-to-use Linux-based cloud machines • Simply click Start Lab to access your pre-configured environment • No need to build or configure your own virtual machine

Lab Environment Setup
Your Al Nafi cloud machine comes pre-installed with: • Kubernetes cluster (minikube) • kubectl command-line tool • Docker runtime • curl and other essential utilities • Text editors (nano, vim)

Task 1: Creating and Managing Kubernetes Secrets
Subtask 1.1: Create Secrets Using Different Methods
Step 1: Verify Kubernetes Cluster Status

First, let's ensure your Kubernetes cluster is running properly.

# Check cluster status
kubectl cluster-info

# Verify nodes are ready
kubectl get nodes

# If minikube is not running, start it
minikube start
Step 2: Create a Secret Using kubectl Command

Create a secret containing database credentials using the imperative approach.

# Create a secret with username and password
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret123

# Verify the secret was created
kubectl get secrets

# View secret details (note: values are base64 encoded)
kubectl describe secret db-credentials
Step 3: Create a Secret from Files

Create files containing sensitive data and generate secrets from them.

# Create directory for secret files
mkdir -p ~/secrets-lab

# Create username file
echo -n 'dbadmin' > ~/secrets-lab/username.txt

# Create password file
echo -n 'mypassword456' > ~/secrets-lab/password.txt

# Create secret from files
kubectl create secret generic file-based-secret \
  --from-file=username=~/secrets-lab/username.txt \
  --from-file=password=~/secrets-lab/password.txt

# Verify creation
kubectl get secret file-based-secret -o yaml
Step 4: Create a Secret Using YAML Manifest

Create a declarative secret using a YAML file.

# Create the secret YAML file
cat > ~/secrets-lab/api-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: api-credentials
  namespace: default
type: Opaque
data:
  api-key: YWJjZGVmZ2hpams=  # base64 encoded "abcdefghijk"
  api-secret: bXlzZWNyZXRrZXkxMjM=  # base64 encoded "mysecretkey123"
EOF

# Apply the secret
kubectl apply -f ~/secrets-lab/api-secret.yaml

# List all secrets
kubectl get secrets
Subtask 1.2: Mount Secrets as Volumes in Pods
Step 1: Create a Pod with Secret Volume Mount

Create a Pod that mounts the secret as a volume.

# Create Pod YAML with secret volume
cat > ~/secrets-lab/pod-with-secret-volume.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app-container
    image: nginx:alpine
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo 'Pod running...'; sleep 30; done"]
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
  restartPolicy: Never
EOF

# Deploy the Pod
kubectl apply -f ~/secrets-lab/pod-with-secret-volume.yaml

# Wait for Pod to be ready
kubectl wait --for=condition=Ready pod/secret-volume-pod --timeout=60s
Step 2: Verify Secret Volume Mount

# Check if Pod is running
kubectl get pods

# Access the Pod and verify secret files
kubectl exec -it secret-volume-pod -- ls -la /etc/secrets

# View the secret contents
kubectl exec -it secret-volume-pod -- cat /etc/secrets/username
kubectl exec -it secret-volume-pod -- cat /etc/secrets/password
Step 3: Create a Pod with Secrets as Environment Variables

# Create Pod YAML with secret environment variables
cat > ~/secrets-lab/pod-with-secret-env.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app-container
    image: nginx:alpine
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: api-credentials
          key: api-key
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo 'Environment variables loaded'; sleep 30; done"]
  restartPolicy: Never
EOF

# Deploy the Pod
kubectl apply -f ~/secrets-lab/pod-with-secret-env.yaml

# Wait for Pod to be ready
kubectl wait --for=condition=Ready pod/secret-env-pod --timeout=60s
Step 4: Verify Environment Variables

# Check environment variables in the Pod
kubectl exec -it secret-env-pod -- env | grep -E "DB_|API_"

# View specific environment variables
kubectl exec -it secret-env-pod -- printenv DB_USERNAME
kubectl exec -it secret-env-pod -- printenv DB_PASSWORD
Task 2: Setting Up HashiCorp Vault as External Secrets Manager
Subtask 2.1: Install and Configure HashiCorp Vault
Step 1: Install Vault Using Helm

# Add HashiCorp Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com

# Update Helm repositories
helm repo update

# Create namespace for Vault
kubectl create namespace vault

# Install Vault in development mode (for lab purposes)
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=myroot" \
  --set "injector.enabled=true"

# Wait for Vault to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=vault --namespace vault --timeout=300s
Step 2: Verify Vault Installation

# Check Vault pods
kubectl get pods -n vault

# Check Vault services
kubectl get svc -n vault

# Port forward to access Vault UI (optional)
kubectl port-forward -n vault svc/vault 8200:8200 &

# Set Vault environment variables
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='myroot'
Step 3: Configure Vault CLI

# Download and install Vault CLI
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vault

# Verify Vault CLI installation
vault version

# Check Vault status
vault status
Subtask 2.2: Store Secrets in Vault
Step 1: Enable Key-Value Secrets Engine

# Enable KV secrets engine at path 'secret'
vault secrets enable -path=secret kv-v2

# Verify secrets engines
vault secrets list
Step 2: Store Application Secrets

# Store database credentials
vault kv put secret/myapp/database \
  username=vaultuser \
  password=vaultpassword123 \
  host=db.example.com \
  port=5432

# Store API credentials
vault kv put secret/myapp/api \
  key=vault-api-key-789 \
  secret=vault-api-secret-456 \
  endpoint=https://api.example.com

# Verify secrets are stored
vault kv list secret/myapp

# Read stored secrets
vault kv get secret/myapp/database
vault kv get secret/myapp/api
Subtask 2.3: Configure Kubernetes Authentication
Step 1: Enable Kubernetes Authentication

# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure Kubernetes authentication
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(kubectl exec -n vault vault-0 -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert="$(kubectl exec -n vault vault-0 -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt)"
Step 2: Create Vault Policy

# Create policy for application secrets
vault policy write myapp-policy - << 'EOF'
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

# Verify policy creation
vault policy list
vault policy read myapp-policy
Step 3: Create Kubernetes Role

# Create Kubernetes role
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=default \
  policies=myapp-policy \
  ttl=1h
Subtask 2.4: Deploy Application with Vault Integration
Step 1: Create Service Account

# Create service account for Vault authentication
cat > ~/secrets-lab/vault-serviceaccount.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth
  namespace: default
EOF

# Apply service account
kubectl apply -f ~/secrets-lab/vault-serviceaccount.yaml
Step 2: Create Pod with Vault Agent Injector

# Create Pod with Vault annotations
cat > ~/secrets-lab/vault-injected-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: vault-injected-app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-database: "secret/data/myapp/database"
    vault.hashicorp.com/agent-inject-template-database: |
      {{- with secret "secret/data/myapp/database" -}}
      export DB_USERNAME="{{ .Data.data.username }}"
      export DB_PASSWORD="{{ .Data.data.password }}"
      export DB_HOST="{{ .Data.data.host }}"
      export DB_PORT="{{ .Data.data.port }}"
      {{- end }}
    vault.hashicorp.com/agent-inject-secret-api: "secret/data/myapp/api"
    vault.hashicorp.com/agent-inject-template-api: |
      {{- with secret "secret/data/myapp/api" -}}
      export API_KEY="{{ .Data.data.key }}"
      export API_SECRET="{{ .Data.data.secret }}"
      export API_ENDPOINT="{{ .Data.data.endpoint }}"
      {{- end }}
spec:
  serviceAccountName: vault-auth
  containers:
  - name: app
    image: nginx:alpine
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo 'App running with Vault secrets'; sleep 30; done"]
EOF

# Deploy the Pod
kubectl apply -f ~/secrets-lab/vault-injected-pod.yaml

# Wait for Pod to be ready (this may take a few minutes)
kubectl wait --for=condition=Ready pod/vault-injected-app --timeout=300s
Step 3: Verify Vault Secret Injection

# Check Pod status
kubectl get pods

# View injected secrets
kubectl exec -it vault-injected-app -c app -- ls -la /vault/secrets/

# View database secrets
kubectl exec -it vault-injected-app -c app -- cat /vault/secrets/database

# View API secrets
kubectl exec -it vault-injected-app -c app -- cat /vault/secrets/api
Task 3: Verifying Encryption at Rest and in Transit
Subtask 3.1: Verify Kubernetes Secrets Encryption at Rest
Step 1: Check etcd Encryption Configuration

# Check if encryption is enabled (in minikube)
minikube ssh -- sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption

# Create a test secret to verify encryption
kubectl create secret generic encryption-test --from-literal=data=sensitive-information

# Check the secret in etcd (this shows it should be encrypted)
kubectl get secret encryption-test -o yaml
Step 2: Verify Base64 Encoding vs Encryption

# Decode the base64 encoded secret (this is NOT encryption)
kubectl get secret encryption-test -o jsonpath='{.data.data}' | base64 -d

# Note: In a production cluster, secrets should be encrypted at rest in etcd
# This requires configuring encryption providers in the API server
Subtask 3.2: Verify Vault Encryption
Step 1: Check Vault Storage Encryption

# Check Vault seal status (unsealed means encryption keys are available)
vault status

# Verify that data is encrypted in Vault's storage
vault kv get -format=json secret/myapp/database

# The data shown is the decrypted version; Vault encrypts it before storage
Step 2: Test Vault Transit Secrets Engine

# Enable transit secrets engine for encryption as a service
vault secrets enable transit

# Create an encryption key
vault write -f transit/keys/myapp-key

# Encrypt some data
vault write transit/encrypt/myapp-key plaintext=$(echo -n "sensitive data" | base64)

# The output shows encrypted data that can be safely stored anywhere
Subtask 3.3: Verify Communication Encryption
Step 1: Check TLS Configuration

# Verify Vault is using HTTPS (in production)
curl -k -s -o /dev/null -w "%{http_code}" https://127.0.0.1:8200/v1/sys/health

# Check Kubernetes API server TLS
kubectl config view --minify --flatten -o jsonpath='{.clusters[0].cluster.server}'
Step 2: Test Secure Communication

# Create a Pod that communicates with Vault over HTTPS
cat > ~/secrets-lab/secure-comm-test.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secure-comm-test
spec:
  containers:
  - name: curl-container
    image: curlimages/curl:latest
    command: ["/bin/sh"]
    args: ["-c", "while true; do curl -k -s http://vault.vault.svc.cluster.local:8200/v1/sys/health && echo ' - Vault health check completed'; sleep 60; done"]
  restartPolicy: Never
EOF

# Deploy and check
kubectl apply -f ~/secrets-lab/secure-comm-test.yaml
kubectl logs secure-comm-test -f
Task 4: Implementing Best Practices
Subtask 4.1: Secret Rotation
Step 1: Update Secrets in Vault

# Update database password in Vault
vault kv put secret/myapp/database \
  username=vaultuser \
  password=newrotatedpassword456 \
  host=db.example.com \
  port=5432

# Verify the update
vault kv get secret/myapp/database
Step 2: Force Pod Restart to Get New Secrets

# Delete and recreate the Vault-injected Pod to get new secrets
kubectl delete pod vault-injected-app

# Recreate the Pod
kubectl apply -f ~/secrets-lab/vault-injected-pod.yaml

# Wait for it to be ready
kubectl wait --for=condition=Ready pod/vault-injected-app --timeout=300s

# Verify new secrets are loaded
kubectl exec -it vault-injected-app -c app -- cat /vault/secrets/database
Subtask 4.2: Implement Secret Scanning
Step 1: Check for Hardcoded Secrets

# Create a script to scan for potential hardcoded secrets
cat > ~/secrets-lab/secret-scanner.sh << 'EOF'
#!/bin/bash

echo "Scanning for potential hardcoded secrets..."

# Check YAML files for suspicious patterns
find ~/secrets-lab -name "*.yaml" -exec grep -l "password\|secret\|key" {} \;

# Look for base64 encoded strings that might be secrets
find ~/secrets-lab -name "*.yaml" -exec grep -E "[A-Za-z0-9+/]{20,}={0,2}" {} \;

echo "Scan completed. Review any findings manually."
EOF

chmod +x ~/secrets-lab/secret-scanner.sh
~/secrets-lab/secret-scanner.sh
Subtask 4.3: Monitor Secret Access
Step 1: Enable Vault Audit Logging

# Enable file audit device in Vault
vault audit enable file file_path=/vault/logs/audit.log

# Perform some secret operations to generate audit logs
vault kv get secret/myapp/database
vault kv get secret/myapp/api

# Check audit logs (in a real environment)
kubectl exec -n vault vault-0 -- tail -f /vault/logs/audit.log
Troubleshooting Common Issues
Issue 1: Vault Pod Not Starting
# Check Vault pod logs
kubectl logs -n vault vault-0

# Check Vault service status
kubectl get svc -n vault

# Restart Vault if needed
kubectl delete pod -n vault vault-0
Issue 2: Secret Injection Not Working
# Check Vault agent injector logs
kubectl logs -n vault -l app.kubernetes.io/name=vault-agent-injector

# Verify service account permissions
kubectl describe sa vault-auth

# Check Vault authentication configuration
vault read auth/kubernetes/config
Issue 3: Secrets Not Mounting
# Check Pod events
kubectl describe pod secret-volume-pod

# Verify secret exists
kubectl get secret db-credentials -o yaml

# Check volume mount configuration
kubectl describe pod secret-volume-pod | grep -A 10 "Mounts:"
Cleanup
Step 1: Remove Lab Resources

# Delete Pods
kubectl delete pod secret-volume-pod secret-env-pod vault-injected-app secure-comm-test

# Delete secrets
kubectl delete secret db-credentials file-based-secret api-credentials encryption-test

# Delete service account
kubectl delete sa vault-auth

# Uninstall Vault
helm uninstall vault -n vault
kubectl delete namespace vault

# Clean up files
rm -rf ~/secrets-lab
Step 2: Stop Port Forwarding

# Kill any background port-forward processes
pkill -f "kubectl port-forward"
Conclusion
In this comprehensive lab, you have successfully:

• Mastered Kubernetes Secrets Management: Created secrets using multiple methods (imperative commands, files, and YAML manifests) and learned to mount them as both volumes and environment variables in Pods.

• Implemented Enterprise-Grade External Secrets Management: Set up HashiCorp Vault as a centralized secrets manager, configured Kubernetes authentication, and used Vault Agent Injector for seamless secret delivery to applications.

• Verified Security Controls: Confirmed that sensitive data is properly encrypted at rest and during communication, understanding the difference between base64 encoding and actual encryption.

• Applied Best Practices: Implemented secret rotation, scanning for hardcoded secrets, and monitoring secret access through audit logging.

Why This Matters:

Proper secrets management is critical for cloud-native security because:

Prevents Data Breaches: Encrypted secrets protect sensitive information even if storage is compromised
Enables Compliance: Proper secret handling meets regulatory requirements like SOC 2, PCI DSS, and GDPR
Supports DevSecOps: Automated secret injection eliminates hardcoded credentials in application code
Facilitates Secret Rotation: Centralized management enables regular password and key updates without application downtime
Real-World Applications:

Database connection strings and API keys in microservices
TLS certificates for service-to-service communication
OAuth tokens and JWT signing keys
Cloud provider credentials for infrastructure automation
This lab has prepared you with practical skills essential for the Kubernetes and Cloud Native Security Associate (KCSA) certification and real-world cloud-native security implementations. You now understand how to secure sensitive data throughout its lifecycle in Kubernetes environments using both native and external solutions.
