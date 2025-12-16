Lab 13: Implementing PKI for Kubernetes
Objectives
By the end of this lab, you will be able to:

• Understand the fundamentals of Public Key Infrastructure (PKI) in Kubernetes environments • Set up a custom Certificate Authority (CA) for Kubernetes cluster components • Generate and manage TLS certificates for Kubernetes API server and other critical components • Implement certificate rotation procedures for maintaining cluster security • Verify secure communication between Kubernetes components using custom certificates • Troubleshoot common PKI-related issues in Kubernetes clusters

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes architecture and components • Familiarity with Linux command line operations • Knowledge of SSL/TLS certificates and cryptographic concepts • Understanding of YAML configuration files • Basic networking concepts including DNS and port communication

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own VM or install additional software - everything is ready to use!

Your lab environment includes: • Ubuntu 20.04 LTS with necessary tools pre-installed • OpenSSL for certificate generation and management • kubectl command-line tool • A basic Kubernetes cluster setup • Text editors (nano, vim) for configuration file editing

Lab Tasks
Task 1: Set up a Custom CA for Kubernetes Cluster Components
Subtask 1.1: Create the Certificate Authority Structure
First, we'll create a directory structure to organize our PKI components and generate the root CA certificate.

Create the PKI directory structure:
# Create main PKI directory
mkdir -p ~/k8s-pki/{ca,certs,keys,csr}

# Navigate to the PKI directory
cd ~/k8s-pki

# Create subdirectories for better organization
mkdir -p ca/{private,certs}
mkdir -p api-server/{private,certs}
mkdir -p etcd/{private,certs}
mkdir -p kubelet/{private,certs}
Generate the Root CA private key:
# Generate a 4096-bit RSA private key for the CA
openssl genrsa -out ca/private/ca-key.pem 4096

# Set appropriate permissions for the private key
chmod 400 ca/private/ca-key.pem
Create the CA configuration file:
# Create CA configuration file
cat > ca/ca-config.json << EOF
{
    "signing": {
        "default": {
            "expiry": "8760h"
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "8760h"
            }
        }
    }
}
EOF
Create the CA Certificate Signing Request (CSR) configuration:
# Create CA CSR configuration
cat > ca/ca-csr.json << EOF
{
    "CN": "Kubernetes CA",
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "US",
            "L": "San Francisco",
            "O": "Kubernetes",
            "OU": "CA",
            "ST": "California"
        }
    ]
}
EOF
Subtask 1.2: Generate the Root CA Certificate
Generate the CA certificate using OpenSSL:
# Generate the CA certificate
openssl req -new -x509 -key ca/private/ca-key.pem -out ca/certs/ca.pem -days 365 -config <(
cat << EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
C = US
ST = California
L = San Francisco
O = Kubernetes
OU = CA
CN = Kubernetes CA

[v3_ca]
basicConstraints = CA:TRUE
keyUsage = keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer:always
EOF
)
Verify the CA certificate:
# Display CA certificate details
openssl x509 -in ca/certs/ca.pem -text -noout

# Check certificate validity
openssl x509 -in ca/certs/ca.pem -noout -dates
Subtask 1.3: Create Certificate Templates for Kubernetes Components
Create API Server certificate configuration:
# Create API Server CSR configuration
cat > api-server/api-server-csr.json << EOF
{
    "CN": "kube-apiserver",
    "hosts": [
        "127.0.0.1",
        "localhost",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster.local",
        "10.96.0.1"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "US",
            "L": "San Francisco",
            "O": "system:masters",
            "OU": "Kubernetes The Hard Way",
            "ST": "California"
        }
    ]
}
EOF
Create etcd certificate configuration:
# Create etcd CSR configuration
cat > etcd/etcd-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
        "127.0.0.1",
        "localhost",
        "etcd.kube-system.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "US",
            "L": "San Francisco",
            "O": "etcd",
            "OU": "Kubernetes The Hard Way",
            "ST": "California"
        }
    ]
}
EOF
Task 2: Generate TLS Certificates for Kubernetes Components
Subtask 2.1: Generate API Server Certificates
Create the API Server private key:
# Generate API Server private key
openssl genrsa -out api-server/private/api-server-key.pem 2048

# Set appropriate permissions
chmod 400 api-server/private/api-server-key.pem
Generate API Server certificate:
# Create certificate signing request
openssl req -new -key api-server/private/api-server-key.pem -out api-server/api-server.csr -config <(
cat << EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = California
L = San Francisco
O = system:masters
OU = Kubernetes The Hard Way
CN = kube-apiserver

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = localhost
IP.1 = 127.0.0.1
IP.2 = 10.96.0.1
EOF
)

# Sign the certificate with our CA
openssl x509 -req -in api-server/api-server.csr -CA ca/certs/ca.pem -CAkey ca/private/ca-key.pem -CAcreateserial -out api-server/certs/api-server.pem -days 365 -extensions v3_req -extfile <(
cat << EOF
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = localhost
IP.1 = 127.0.0.1
IP.2 = 10.96.0.1
EOF
)
Subtask 2.2: Generate etcd Certificates
Create etcd private key and certificate:
# Generate etcd private key
openssl genrsa -out etcd/private/etcd-key.pem 2048

# Set permissions
chmod 400 etcd/private/etcd-key.pem

# Create etcd certificate signing request
openssl req -new -key etcd/private/etcd-key.pem -out etcd/etcd.csr -config <(
cat << EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = California
L = San Francisco
O = etcd
OU = Kubernetes The Hard Way
CN = etcd

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = etcd.kube-system.svc.cluster.local
IP.1 = 127.0.0.1
EOF
)

# Sign the etcd certificate
openssl x509 -req -in etcd/etcd.csr -CA ca/certs/ca.pem -CAkey ca/private/ca-key.pem -CAcreateserial -out etcd/certs/etcd.pem -days 365 -extensions v3_req -extfile <(
cat << EOF
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = etcd.kube-system.svc.cluster.local
IP.1 = 127.0.0.1
EOF
)
Subtask 2.3: Generate Client Certificates
Create admin client certificate:
# Create admin client private key
openssl genrsa -out certs/admin-key.pem 2048

# Create admin client certificate
openssl req -new -key certs/admin-key.pem -out csr/admin.csr -subj "/C=US/ST=California/L=San Francisco/O=system:masters/OU=Kubernetes The Hard Way/CN=admin"

# Sign admin certificate
openssl x509 -req -in csr/admin.csr -CA ca/certs/ca.pem -CAkey ca/private/ca-key.pem -CAcreateserial -out certs/admin.pem -days 365
Create kubelet client certificate:
# Create kubelet client private key
openssl genrsa -out kubelet/private/kubelet-key.pem 2048

# Create kubelet client certificate
openssl req -new -key kubelet/private/kubelet-key.pem -out kubelet/kubelet.csr -subj "/C=US/ST=California/L=San Francisco/O=system:nodes/OU=Kubernetes The Hard Way/CN=system:node:worker-1"

# Sign kubelet certificate
openssl x509 -req -in kubelet/kubelet.csr -CA ca/certs/ca.pem -CAkey ca/private/ca-key.pem -CAcreateserial -out kubelet/certs/kubelet.pem -days 365
Task 3: Implement Certificate Rotation for Kubernetes Components
Subtask 3.1: Create Certificate Rotation Scripts
Create a certificate rotation script:
# Create certificate rotation script
cat > rotate-certificates.sh << 'EOF'
#!/bin/bash

# Certificate rotation script for Kubernetes components
set -e

PKI_DIR="$HOME/k8s-pki"
BACKUP_DIR="$PKI_DIR/backup-$(date +%Y%m%d-%H%M%S)"

echo "Starting certificate rotation process..."

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Function to backup existing certificates
backup_certificates() {
    echo "Backing up existing certificates..."
    cp -r "$PKI_DIR/api-server/certs" "$BACKUP_DIR/api-server-certs"
    cp -r "$PKI_DIR/etcd/certs" "$BACKUP_DIR/etcd-certs"
    cp -r "$PKI_DIR/certs" "$BACKUP_DIR/client-certs"
    echo "Backup completed in $BACKUP_DIR"
}

# Function to check certificate expiration
check_certificate_expiration() {
    local cert_file="$1"
    local cert_name="$2"
    
    if [ -f "$cert_file" ]; then
        local expiry_date=$(openssl x509 -in "$cert_file" -noout -enddate | cut -d= -f2)
        local expiry_epoch=$(date -d "$expiry_date" +%s)
        local current_epoch=$(date +%s)
        local days_until_expiry=$(( (expiry_epoch - current_epoch) / 86400 ))
        
        echo "$cert_name expires in $days_until_expiry days ($expiry_date)"
        
        if [ $days_until_expiry -lt 30 ]; then
            echo "WARNING: $cert_name expires in less than 30 days!"
            return 1
        fi
    else
        echo "Certificate file $cert_file not found!"
        return 1
    fi
    return 0
}

# Function to rotate API server certificate
rotate_api_server_cert() {
    echo "Rotating API server certificate..."
    
    # Generate new private key
    openssl genrsa -out "$PKI_DIR/api-server/private/api-server-key-new.pem" 2048
    
    # Generate new certificate
    openssl req -new -key "$PKI_DIR/api-server/private/api-server-key-new.pem" -out "$PKI_DIR/api-server/api-server-new.csr" -config <(
cat << EOL
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = California
L = San Francisco
O = system:masters
OU = Kubernetes The Hard Way
CN = kube-apiserver

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = localhost
IP.1 = 127.0.0.1
IP.2 = 10.96.0.1
EOL
)
    
    # Sign new certificate
    openssl x509 -req -in "$PKI_DIR/api-server/api-server-new.csr" -CA "$PKI_DIR/ca/certs/ca.pem" -CAkey "$PKI_DIR/ca/private/ca-key.pem" -CAcreateserial -out "$PKI_DIR/api-server/certs/api-server-new.pem" -days 365 -extensions v3_req -extfile <(
cat << EOL
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = localhost
IP.1 = 127.0.0.1
IP.2 = 10.96.0.1
EOL
)
    
    # Replace old certificates with new ones
    mv "$PKI_DIR/api-server/private/api-server-key.pem" "$PKI_DIR/api-server/private/api-server-key-old.pem"
    mv "$PKI_DIR/api-server/certs/api-server.pem" "$PKI_DIR/api-server/certs/api-server-old.pem"
    mv "$PKI_DIR/api-server/private/api-server-key-new.pem" "$PKI_DIR/api-server/private/api-server-key.pem"
    mv "$PKI_DIR/api-server/certs/api-server-new.pem" "$PKI_DIR/api-server/certs/api-server.pem"
    
    # Set proper permissions
    chmod 400 "$PKI_DIR/api-server/private/api-server-key.pem"
    
    echo "API server certificate rotated successfully"
}

# Main execution
backup_certificates

# Check certificate expiration
echo "Checking certificate expiration..."
check_certificate_expiration "$PKI_DIR/api-server/certs/api-server.pem" "API Server"
api_server_needs_rotation=$?

check_certificate_expiration "$PKI_DIR/etcd/certs/etcd.pem" "etcd"
etcd_needs_rotation=$?

# Rotate certificates if needed
if [ $api_server_needs_rotation -ne 0 ]; then
    rotate_api_server_cert
fi

echo "Certificate rotation process completed!"
EOF

# Make the script executable
chmod +x rotate-certificates.sh
Subtask 3.2: Implement Automated Certificate Monitoring
Create a certificate monitoring script:
# Create certificate monitoring script
cat > monitor-certificates.sh << 'EOF'
#!/bin/bash

# Certificate monitoring script
PKI_DIR="$HOME/k8s-pki"
LOG_FILE="$PKI_DIR/cert-monitor.log"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Function to check certificate validity
check_cert_validity() {
    local cert_file="$1"
    local cert_name="$2"
    
    if [ ! -f "$cert_file" ]; then
        log_message "ERROR: Certificate file $cert_file not found"
        return 1
    fi
    
    # Check if certificate is valid
    if ! openssl x509 -in "$cert_file" -noout -checkend 0 >/dev/null 2>&1; then
        log_message "ERROR: Certificate $cert_name has expired"
        return 1
    fi
    
    # Check if certificate expires within 30 days
    if openssl x509 -in "$cert_file" -noout -checkend 2592000 >/dev/null 2>&1; then
        log_message "OK: Certificate $cert_name is valid"
        return 0
    else
        log_message "WARNING: Certificate $cert_name expires within 30 days"
        return 2
    fi
}

# Function to send alert (placeholder for actual alerting mechanism)
send_alert() {
    local message="$1"
    log_message "ALERT: $message"
    # In a real environment, you would integrate with your alerting system
    # Example: curl -X POST -H 'Content-type: application/json' --data '{"text":"'"$message"'"}' YOUR_WEBHOOK_URL
}

# Main monitoring logic
log_message "Starting certificate monitoring check"

# Check all certificates
certificates=(
    "$PKI_DIR/ca/certs/ca.pem:Root CA"
    "$PKI_DIR/api-server/certs/api-server.pem:API Server"
    "$PKI_DIR/etcd/certs/etcd.pem:etcd"
    "$PKI_DIR/certs/admin.pem:Admin Client"
    "$PKI_DIR/kubelet/certs/kubelet.pem:Kubelet"
)

alert_needed=false

for cert_info in "${certificates[@]}"; do
    cert_file="${cert_info%:*}"
    cert_name="${cert_info#*:}"
    
    check_cert_validity "$cert_file" "$cert_name"
    result=$?
    
    if [ $result -eq 1 ]; then
        send_alert "Certificate $cert_name has expired or is invalid"
        alert_needed=true
    elif [ $result -eq 2 ]; then
        send_alert "Certificate $cert_name expires within 30 days"
        alert_needed=true
    fi
done

if [ "$alert_needed" = false ]; then
    log_message "All certificates are valid and not expiring soon"
fi

log_message "Certificate monitoring check completed"
EOF

# Make the script executable
chmod +x monitor-certificates.sh
Subtask 3.3: Set up Automated Certificate Rotation
Create a cron job for certificate monitoring:
# Add cron job for daily certificate monitoring
(crontab -l 2>/dev/null; echo "0 9 * * * $HOME/k8s-pki/monitor-certificates.sh") | crontab -

# Verify cron job was added
crontab -l
Test the certificate rotation process:
# Run the certificate monitoring script
./monitor-certificates.sh

# Check the log file
cat ~/k8s-pki/cert-monitor.log
Task 4: Verify Secure Communication Between Components
Subtask 4.1: Create Verification Scripts
Create a certificate verification script:
# Create certificate verification script
cat > verify-certificates.sh << 'EOF'
#!/bin/bash

# Certificate verification script
PKI_DIR="$HOME/k8s-pki"

echo "=== Kubernetes PKI Certificate Verification ==="
echo

# Function to verify certificate against CA
verify_certificate() {
    local cert_file="$1"
    local cert_name="$2"
    local ca_file="$PKI_DIR/ca/certs/ca.pem"
    
    echo "Verifying $cert_name certificate..."
    
    if [ ! -f "$cert_file" ]; then
        echo "  ERROR: Certificate file not found: $cert_file"
        return 1
    fi
    
    # Verify certificate against CA
    if openssl verify -CAfile "$ca_file" "$cert_file" >/dev/null 2>&1; then
        echo "  ✓ Certificate is valid and signed by our CA"
    else
        echo "  ✗ Certificate verification failed"
        return 1
    fi
    
    # Display certificate details
    echo "  Certificate Details:"
    echo "    Subject: $(openssl x509 -in "$cert_file" -noout -subject | sed 's/subject=//')"
    echo "    Issuer: $(openssl x509 -in "$cert_file" -noout -issuer | sed 's/issuer=//')"
    echo "    Valid From: $(openssl x509 -in "$cert_file" -noout -startdate | sed 's/notBefore=//')"
    echo "    Valid Until: $(openssl x509 -in "$cert_file" -noout -enddate | sed 's/notAfter=//')"
    
    # Check Subject Alternative Names for server certificates
    if openssl x509 -in "$cert_file" -noout -ext subjectAltName >/dev/null 2>&1; then
        echo "    Subject Alternative Names:"
        openssl x509 -in "$cert_file" -noout -ext subjectAltName | grep -A 10 "X509v3 Subject Alternative Name:" | tail -n +2 | sed 's/^/      /'
    fi
    
    echo
    return 0
}

# Function to test certificate chain
test_certificate_chain() {
    local server_cert="$1"
    local server_key="$2"
    local ca_cert="$3"
    local test_name="$4"
    
    echo "Testing certificate chain for $test_name..."
    
    # Create a temporary server configuration
    local temp_config=$(mktemp)
    cat > "$temp_config" << EOL
[req]
distinguished_name = req_distinguished_name

[req_distinguished_name]
EOL
    
    # Test if private key matches certificate
    local cert_modulus=$(openssl x509 -noout -modulus -in "$server_cert" 2>/dev/null | openssl md5)
    local key_modulus=$(openssl rsa -noout -modulus -in "$server_key" 2>/dev/null | openssl md5)
    
    if [ "$cert_modulus" = "$key_modulus" ]; then
        echo "  ✓ Private key matches certificate"
    else
        echo "  ✗ Private key does not match certificate"
        rm -f "$temp_config"
        return 1
    fi
    
    # Verify certificate chain
    if openssl verify -CAfile "$ca_cert" "$server_cert" >/dev/null 2>&1; then
        echo "  ✓ Certificate chain is valid"
    else
        echo "  ✗ Certificate chain verification failed"
        rm -f "$temp_config"
        return 1
    fi
    
    rm -f "$temp_config"
    echo
    return 0
}

# Verify CA certificate
echo "1. Verifying Root CA Certificate"
echo "================================"
if [ -f "$PKI_DIR/ca/certs/ca.pem" ]; then
    echo "CA Certificate Details:"
    echo "  Subject: $(openssl x509 -in "$PKI_DIR/ca/certs/ca.pem" -noout -subject | sed 's/subject=//')"
    echo "  Valid From: $(openssl x509 -in "$PKI_DIR/ca/certs/ca.pem" -noout -startdate | sed 's/notBefore=//')"
    echo "  Valid Until: $(openssl x509 -in "$PKI_DIR/ca/certs/ca.pem" -noout -enddate | sed 's/notAfter=//')"
    echo "  ✓ Root CA certificate found and readable"
else
    echo "  ✗ Root CA certificate not found"
    exit 1
fi
echo

# Verify individual certificates
echo "2. Verifying Individual Certificates"
echo "===================================="
verify_certificate "$PKI_DIR/api-server/certs/api-server.pem" "API Server"
verify_certificate "$PKI_DIR/etcd/certs/etcd.pem" "etcd"
verify_certificate "$PKI_DIR/certs/admin.pem" "Admin Client"
verify_certificate "$PKI_DIR/kubelet/certs/kubelet.pem" "Kubelet"

# Test certificate chains
echo "3. Testing Certificate Chains"
echo "============================="
test_certificate_chain "$PKI_DIR/api-server/certs/api-server.pem" "$PKI_DIR/api-server/private/api-server-key.pem" "$PKI_DIR/ca/certs/ca.pem" "API Server"
test_certificate_chain "$PKI_DIR/etcd/certs/etcd.pem" "$PKI_DIR/etcd/private/etcd-key.pem" "$PKI_DIR/ca/certs/ca.pem" "etcd"

echo "Certificate verification completed!"
EOF

# Make the script executable
chmod +x verify-certificates.sh
Subtask 4.2: Test Certificate-Based Authentication
Create a test script for certificate-based authentication:
# Create authentication test script
cat > test-auth.sh << 'EOF'
#!/bin/bash

# Certificate-based authentication test script
PKI_DIR="$HOME/k8s-pki"

echo "=== Testing Certificate-Based Authentication ==="
echo

# Function to test client certificate authentication
test_client_auth() {
    local client_cert="$1"
    local client_key="$2"
    local ca_cert="$3"
    local client_name="$4"
    
    echo "Testing $client_name certificate authentication..."
    
    # Create a temporary kubeconfig for testing
    local temp_kubeconfig=$(mktemp)
    
    # Note: This is a simulation since we don't have a running cluster
    # In a real environment, you would test against an actual API server
    
    echo "  Certificate Details:"
    echo "    Subject: $(openssl x509 -in "$client_cert" -noout -subject | sed 's/subject=//')"
    
    # Verify the client certificate
    if openssl verify -CAfile "$ca_cert" "$client_cert" >/dev/null 2>&1; then
        echo "  ✓ Client certificate is valid"
    else
        echo "  ✗ Client certificate verification failed"
        rm -f "$temp_kubeconfig"
        return 1
    fi
    
    # Check if private key matches certificate
    local cert_modulus=$(openssl x509 -noout -modulus -in "$client_cert" 2>/dev/null | openssl md5)
    local key_modulus=$(openssl rsa -noout -modulus -in "$client_key" 2>/dev/null | openssl md5)
    
    if [ "$cert_modulus" = "$key_modulus" ]; then
        echo "  ✓ Private key matches certificate"
    else
        echo "  ✗ Private key does not match certificate"
        rm -f "$temp_kubeconfig"
        return 1
    fi
    
    # Create kubeconfig content (for demonstration)
    cat > "$temp_kubeconfig" << EOL
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: $(base64 -w 0 "$ca_cert")
    server: https://127.0.0.1:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: $client_name
  name: $client_name@kubernetes
current-context: $client_name@kubernetes
users:
- name: $client_name
  user:
    client-certificate-data: $(base64 -w 0 "$client_cert")
    client-key-data: $(base64 -w 0 "$client_key")
EOL
    
    echo "  ✓ Kubeconfig created successfully"
    echo "  ✓ Certificate-based authentication configuration is valid"
    
    rm -f "$temp_kubeconfig"
    echo
    return 0
}

# Test admin client authentication
test_client_auth "$PKI_DIR/certs/admin.pem" "$PKI_DIR/keys/admin-key.pem" "$PKI_DIR/ca/certs/ca.pem" "admin"

# Test kubelet client authentication
test_client_auth "$PKI_DIR/kubelet/certs/kubelet.pem" "$PKI_DIR/kubelet/private/kubelet-key.pem" "$PKI_DIR/ca/certs/ca.pem" "kubelet"

echo "Authentication testing completed!"
EOF

# Make the script executable
chmod +x test-auth.sh
Subtask 4.3: Verify TLS Communication
Create a TLS communication test script:
# Create TLS communication test script
cat > test-tls-communication.sh << 'EOF'
#!/bin/bash

# TLS communication test script
PKI_DIR="$HOME/k8s-pki"

echo "=== Testing TLS Communication ==="
echo

# Function to test TLS handshake
test_tls_handshake() {
    local server_cert="$1"
    local server_key="$2"
    local ca_cert="$3"
    local service_name="$4"
    local port="$5"
    
    echo "Testing TLS handshake for $service_name..."
    
    # Start a simple TLS server in background for testing
    local temp_server_config=$(
