Lab 14: Managing Secrets with Vault
Objectives
By the end of this lab, you will be able to:

• Deploy HashiCorp Vault in a Kubernetes environment • Configure Vault for Kubernetes integration using service accounts • Store and retrieve static secrets securely from Vault • Implement dynamic secrets to manage short-lived database credentials • Understand Vault's authentication and authorization mechanisms • Configure Vault policies for secure access control • Integrate applications with Vault for automated secret management

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with command-line interface operations • Knowledge of YAML configuration files • Understanding of basic security concepts like authentication and authorization • Experience with kubectl commands • Basic knowledge of database concepts

Lab Environment
Al Nafi provides you with a pre-configured Linux-based cloud machine with all necessary tools installed. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Your lab environment includes: • Kubernetes cluster (minikube) • kubectl CLI tool • Helm package manager • curl and jq utilities • PostgreSQL database

Task 1: Deploy HashiCorp Vault in Kubernetes
Subtask 1.1: Prepare the Kubernetes Environment
First, let's ensure our Kubernetes cluster is running and create a dedicated namespace for Vault.

# Start minikube if not already running
minikube start

# Verify cluster status
kubectl cluster-info

# Create a namespace for Vault
kubectl create namespace vault

# Set the namespace as default for this session
kubectl config set-context --current --namespace=vault
Subtask 1.2: Install Vault using Helm
We'll use Helm to deploy Vault with a development configuration that's perfect for learning.

# Add the HashiCorp Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com

# Update Helm repositories
helm repo update

# Install Vault in development mode
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=myroot" \
  --set "injector.enabled=false"
Subtask 1.3: Verify Vault Deployment
Let's check that Vault is running properly.

# Check pod status
kubectl get pods

# Wait for the vault pod to be ready (this may take a few minutes)
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vault --timeout=300s

# Check Vault service
kubectl get svc
Subtask 1.4: Access Vault
Now let's access Vault and verify it's working.

# Port forward to access Vault locally
kubectl port-forward svc/vault 8200:8200 &

# Set environment variables for Vault CLI
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='myroot'

# Check Vault status
kubectl exec -it vault-0 -- vault status
Task 2: Configure Vault for Kubernetes Integration
Subtask 2.1: Enable Kubernetes Authentication
We need to configure Vault to authenticate with Kubernetes service accounts.

# Execute commands inside the Vault pod
kubectl exec -it vault-0 -- /bin/sh

# Inside the Vault pod, enable Kubernetes auth method
vault auth enable kubernetes

# Configure Kubernetes authentication
vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Exit the Vault pod
exit
Subtask 2.2: Create a Service Account for Applications
Let's create a service account that our applications will use to authenticate with Vault.

# Create a service account
kubectl create serviceaccount vault-auth

# Create a secret for the service account (required for older Kubernetes versions)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault-auth
type: kubernetes.io/service-account-token
EOF

# Bind the service account to a cluster role (for demo purposes)
kubectl create clusterrolebinding vault-auth-binding \
  --clusterrole=system:auth-delegator \
  --serviceaccount=vault:vault-auth
Subtask 2.3: Configure Vault Policy and Role
Now we'll create a policy and role for our applications.

# Create a policy file
cat <<EOF > app-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "database/creds/my-role" {
  capabilities = ["read"]
}
EOF

# Apply the policy to Vault
kubectl cp app-policy.hcl vault-0:/tmp/app-policy.hcl
kubectl exec -it vault-0 -- vault policy write myapp-policy /tmp/app-policy.hcl

# Create a Kubernetes role
kubectl exec -it vault-0 -- vault write auth/kubernetes/role/myapp-role \
    bound_service_account_names=vault-auth \
    bound_service_account_namespaces=vault \
    policies=myapp-policy \
    ttl=24h
Task 3: Store and Retrieve Static Secrets
Subtask 3.1: Enable Key-Value Secrets Engine
Let's enable the key-value secrets engine to store static secrets.

# Enable KV secrets engine version 2
kubectl exec -it vault-0 -- vault secrets enable -path=secret kv-v2
Subtask 3.2: Store Static Secrets
Now we'll store some application secrets.

# Store database connection details
kubectl exec -it vault-0 -- vault kv put secret/myapp/database \
    username=appuser \
    password=supersecret123 \
    host=database.example.com \
    port=5432

# Store API keys
kubectl exec -it vault-0 -- vault kv put secret/myapp/api \
    stripe_key=sk_test_123456789 \
    sendgrid_key=SG.abcdefghijk \
    jwt_secret=my-jwt-secret-key

# Store configuration values
kubectl exec -it vault-0 -- vault kv put secret/myapp/config \
    debug=true \
    log_level=info \
    max_connections=100
Subtask 3.3: Retrieve Static Secrets
Let's verify we can retrieve the secrets we stored.

# Retrieve database secrets
kubectl exec -it vault-0 -- vault kv get secret/myapp/database

# Retrieve specific field
kubectl exec -it vault-0 -- vault kv get -field=password secret/myapp/database

# Retrieve secrets in JSON format
kubectl exec -it vault-0 -- vault kv get -format=json secret/myapp/api
Subtask 3.4: Create a Test Application to Access Secrets
Let's create a simple application that retrieves secrets from Vault.

# Create a test application deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: vault
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      serviceAccountName: vault-auth
      containers:
      - name: test-app
        image: alpine:latest
        command: ["/bin/sh"]
        args: ["-c", "while true; do sleep 30; done"]
        env:
        - name: VAULT_ADDR
          value: "http://vault:8200"
EOF

# Wait for the pod to be ready
kubectl wait --for=condition=ready pod -l app=test-app --timeout=300s
Now let's test accessing Vault from the application:

# Get the pod name
TEST_POD=$(kubectl get pods -l app=test-app -o jsonpath='{.items[0].metadata.name}')

# Install curl in the test pod
kubectl exec -it $TEST_POD -- apk add --no-cache curl jq

# Get a Vault token using Kubernetes authentication
kubectl exec -it $TEST_POD -- sh -c '
JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
VAULT_TOKEN=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -d "{\"jwt\":\"$JWT\",\"role\":\"myapp-role\"}" \
  http://vault:8200/v1/auth/kubernetes/login | jq -r .auth.client_token)

# Use the token to retrieve secrets
curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
  http://vault:8200/v1/secret/data/myapp/database | jq .data.data
'
Task 4: Implement Dynamic Secrets
Subtask 4.1: Deploy PostgreSQL Database
First, let's deploy a PostgreSQL database that we'll use for dynamic secrets.

# Create a PostgreSQL deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: vault
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: rootpassword
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: vault
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
EOF

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres --timeout=300s
Subtask 4.2: Configure Database Secrets Engine
Now let's configure Vault's database secrets engine.

# Enable the database secrets engine
kubectl exec -it vault-0 -- vault secrets enable database

# Configure the PostgreSQL connection
kubectl exec -it vault-0 -- vault write database/config/my-postgresql-database \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/myapp?sslmode=disable" \
    allowed_roles="my-role" \
    username="postgres" \
    password="rootpassword"
Subtask 4.3: Create Database Role for Dynamic Secrets
Let's create a role that defines what permissions dynamic users will have.

# Create a database role
kubectl exec -it vault-0 -- vault write database/roles/my-role \
    db_name=my-postgresql-database \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
Subtask 4.4: Generate Dynamic Database Credentials
Now let's generate some dynamic credentials.

# Generate dynamic credentials
kubectl exec -it vault-0 -- vault read database/creds/my-role

# Generate credentials multiple times to see different usernames
kubectl exec -it vault-0 -- vault read database/creds/my-role
kubectl exec -it vault-0 -- vault read database/creds/my-role
Subtask 4.5: Test Dynamic Credentials
Let's verify that the dynamic credentials actually work with our database.

# Get PostgreSQL pod name
POSTGRES_POD=$(kubectl get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Generate credentials and extract username/password
CREDS=$(kubectl exec -it vault-0 -- vault read -format=json database/creds/my-role)
USERNAME=$(echo $CREDS | jq -r .data.username)
PASSWORD=$(echo $CREDS | jq -r .data.password)

echo "Generated credentials:"
echo "Username: $USERNAME"
echo "Password: $PASSWORD"

# Test the credentials by connecting to PostgreSQL
kubectl exec -it $POSTGRES_POD -- psql -U $USERNAME -d myapp -c "SELECT current_user, now();"
Subtask 4.6: Demonstrate Credential Lifecycle
Let's see how dynamic credentials have a limited lifetime.

# Check active database connections
kubectl exec -it vault-0 -- vault list sys/leases/lookup/database/creds/my-role

# Generate credentials with custom TTL
kubectl exec -it vault-0 -- vault read database/creds/my-role ttl=30s

# You can also revoke credentials manually
# First, generate credentials and note the lease_id
LEASE_OUTPUT=$(kubectl exec -it vault-0 -- vault read -format=json database/creds/my-role)
LEASE_ID=$(echo $LEASE_OUTPUT | jq -r .lease_id)

echo "Lease ID: $LEASE_ID"

# Revoke the lease (this will delete the database user)
kubectl exec -it vault-0 -- vault lease revoke $LEASE_ID
Task 5: Advanced Vault Operations
Subtask 5.1: Create Application Integration Script
Let's create a more realistic application that demonstrates how to integrate with Vault.

# Create a script that applications can use to get secrets
cat <<EOF > vault-integration.sh
#!/bin/bash

# Vault Integration Script for Applications
VAULT_ADDR="http://vault:8200"
VAULT_ROLE="myapp-role"

# Function to get Vault token using Kubernetes auth
get_vault_token() {
    local jwt=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
    local response=\$(curl -s -X POST \\
        -H "Content-Type: application/json" \\
        -d "{\"jwt\":\"\$jwt\",\"role\":\"\$VAULT_ROLE\"}" \\
        \$VAULT_ADDR/v1/auth/kubernetes/login)
    
    echo \$response | jq -r .auth.client_token
}

# Function to get static secrets
get_static_secret() {
    local path=\$1
    local field=\$2
    local token=\$(get_vault_token)
    
    local response=\$(curl -s -H "X-Vault-Token: \$token" \\
        \$VAULT_ADDR/v1/secret/data/\$path)
    
    if [ -n "\$field" ]; then
        echo \$response | jq -r .data.data.\$field
    else
        echo \$response | jq .data.data
    fi
}

# Function to get dynamic database credentials
get_db_credentials() {
    local token=\$(get_vault_token)
    
    curl -s -H "X-Vault-Token: \$token" \\
        \$VAULT_ADDR/v1/database/creds/my-role | jq .data
}

# Example usage
echo "=== Static Secrets ==="
echo "Database password: \$(get_static_secret myapp/database password)"
echo "API key: \$(get_static_secret myapp/api stripe_key)"

echo ""
echo "=== Dynamic Database Credentials ==="
get_db_credentials
EOF

# Copy the script to our test application
kubectl cp vault-integration.sh $TEST_POD:/tmp/vault-integration.sh
kubectl exec -it $TEST_POD -- chmod +x /tmp/vault-integration.sh

# Run the integration script
kubectl exec -it $TEST_POD -- /tmp/vault-integration.sh
Subtask 5.2: Monitor Vault Audit Logs
Let's enable audit logging to see what's happening in Vault.

# Enable file audit logging
kubectl exec -it vault-0 -- vault audit enable file file_path=/vault/logs/audit.log

# Generate some activity
kubectl exec -it $TEST_POD -- /tmp/vault-integration.sh

# View audit logs
kubectl exec -it vault-0 -- tail -f /vault/logs/audit.log
Subtask 5.3: Backup and Restore Secrets
Let's learn how to backup our Vault data.

# Create a backup of our secrets
kubectl exec -it vault-0 -- sh -c '
# Export static secrets
vault kv get -format=json secret/myapp/database > /tmp/backup-database.json
vault kv get -format=json secret/myapp/api > /tmp/backup-api.json
vault kv get -format=json secret/myapp/config > /tmp/backup-config.json

# Export policies
vault policy read myapp-policy > /tmp/backup-policy.hcl

echo "Backup completed"
'

# Copy backups to local machine
kubectl cp vault-0:/tmp/backup-database.json ./backup-database.json
kubectl cp vault-0:/tmp/backup-api.json ./backup-api.json
kubectl cp vault-0:/tmp/backup-config.json ./backup-config.json
kubectl cp vault-0:/tmp/backup-policy.hcl ./backup-policy.hcl

echo "Backup files copied to local directory"
ls -la backup-*
Troubleshooting Common Issues
Issue 1: Vault Pod Not Starting
If the Vault pod fails to start:

# Check pod logs
kubectl logs vault-0

# Check pod description for events
kubectl describe pod vault-0

# Verify Helm installation
helm list -n vault
Issue 2: Authentication Failures
If Kubernetes authentication fails:

# Verify service account exists
kubectl get serviceaccount vault-auth

# Check if the secret exists
kubectl get secret vault-auth-secret

# Verify the role configuration
kubectl exec -it vault-0 -- vault read auth/kubernetes/role/myapp-role
Issue 3: Database Connection Issues
If dynamic secrets fail:

# Check PostgreSQL pod status
kubectl get pods -l app=postgres

# Test database connectivity
kubectl exec -it vault-0 -- vault read database/config/my-postgresql-database

# Check database logs
kubectl logs -l app=postgres
Issue 4: Permission Denied Errors
If you get permission errors:

# Check current policies
kubectl exec -it vault-0 -- vault token lookup

# Verify policy contents
kubectl exec -it vault-0 -- vault policy read myapp-policy

# Check token capabilities
kubectl exec -it vault-0 -- vault token capabilities secret/myapp/database
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Delete the test application
kubectl delete deployment test-app

# Delete PostgreSQL
kubectl delete deployment postgres
kubectl delete service postgres

# Uninstall Vault
helm uninstall vault -n vault

# Delete the namespace
kubectl delete namespace vault

# Stop port forwarding (if still running)
pkill -f "kubectl port-forward"
Conclusion
Congratulations! You have successfully completed Lab 14: Managing Secrets with Vault. In this lab, you accomplished several important tasks:

What You Learned: • Vault Deployment: You deployed HashiCorp Vault in a Kubernetes environment using Helm, learning how to set up a secure secrets management system.

• Kubernetes Integration: You configured Vault to authenticate with Kubernetes using service accounts, enabling seamless integration between your applications and Vault.

• Static Secrets Management: You stored and retrieved various types of static secrets including database credentials, API keys, and configuration values, understanding how to organize secrets in a hierarchical structure.

• Dynamic Secrets: You implemented dynamic database credentials that are automatically generated with limited lifetimes, significantly improving security by eliminating long-lived credentials.

• Security Policies: You created and applied Vault policies to control access to secrets, implementing the principle of least privilege.

• Application Integration: You built practical examples showing how real applications can authenticate with Vault and retrieve secrets programmatically.

Why This Matters: Managing secrets securely is crucial in modern cloud-native applications. Traditional approaches of storing secrets in configuration files or environment variables create security risks. Vault provides:

• Centralized Secret Management: All secrets are stored in one secure location with proper access controls • Dynamic Credentials: Short-lived credentials reduce the impact of credential compromise • Audit Logging: Complete visibility into who accessed what secrets and when • Encryption: All secrets are encrypted both in transit and at rest • Integration: Native integration with Kubernetes and other cloud-native tools

Real-World Applications: The skills you've learned are directly applicable to: • Securing microservices architectures • Managing database credentials in production environments • Implementing zero-trust security models • Meeting compliance requirements for secret management • Automating credential rotation and lifecycle management

This knowledge is essential for the Kubernetes and Cloud Native Security Associate (KCSA) certification and will help you implement robust security practices in production Kubernetes environments.
