Lab 2: Securing Kubernetes API Server
Objectives
By the end of this lab, students will be able to:

• Configure Role-Based Access Control (RBAC) to restrict user permissions and implement the principle of least privilege • Enable API server encryption at rest to protect sensitive data stored in etcd • Implement TLS certificates to secure API communications between clients and the Kubernetes API server • Understand the security implications of unsecured Kubernetes API servers • Apply security best practices for production Kubernetes environments • Troubleshoot common API server security configuration issues

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Kubernetes architecture and components • Familiarity with Linux command line operations • Knowledge of YAML file structure and syntax • Understanding of basic networking concepts (TCP/IP, ports, certificates) • Experience with kubectl command-line tool • Basic knowledge of public key infrastructure (PKI) concepts

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Kubernetes already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Kubernetes from scratch.

Your lab environment includes: • Ubuntu 20.04 LTS with Kubernetes 1.28+ installed • kubectl configured and ready to use • OpenSSL for certificate generation • All necessary tools pre-installed

Task 1: Configure RBAC to Restrict User Permissions
Subtask 1.1: Understanding Current RBAC Configuration
First, let's examine the current RBAC setup in your cluster.

Check existing cluster roles:
kubectl get clusterroles | head -20
Examine the default service account permissions:
kubectl describe clusterrolebinding system:basic-user
View current role bindings:
kubectl get rolebindings --all-namespaces
Subtask 1.2: Create a Custom Namespace for Testing
Create a dedicated namespace for our security testing:
kubectl create namespace security-lab
Verify the namespace creation:
kubectl get namespaces | grep security-lab
Subtask 1.3: Create a Limited User with Restricted Permissions
Generate a private key for our test user:
openssl genrsa -out testuser.key 2048
Create a certificate signing request (CSR):
openssl req -new -key testuser.key -out testuser.csr -subj "/CN=testuser/O=developers"
Create a Kubernetes CSR object:
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: testuser-csr
spec:
  request: $(cat testuser.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
Approve the certificate signing request:
kubectl certificate approve testuser-csr
Extract the signed certificate:
kubectl get csr testuser-csr -o jsonpath='{.status.certificate}' | base64 -d > testuser.crt
Subtask 1.4: Create Custom Role with Limited Permissions
Create a role that only allows reading pods in the security-lab namespace:
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-lab
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF
Create a role binding to associate the user with the role:
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: security-lab
subjects:
- kind: User
  name: testuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
Subtask 1.5: Test RBAC Configuration
Configure kubectl to use the new user credentials:
kubectl config set-credentials testuser --client-certificate=testuser.crt --client-key=testuser.key
kubectl config set-context testuser-context --cluster=kubernetes --user=testuser
Create a test pod in the security-lab namespace using admin privileges:
kubectl create deployment nginx-test --image=nginx --replicas=1 -n security-lab
Switch to the testuser context and test permissions:
kubectl config use-context testuser-context
Test allowed operations (should work):
kubectl get pods -n security-lab
Test forbidden operations (should fail):
kubectl get pods -n default
kubectl delete pod -n security-lab --all
kubectl create deployment test --image=nginx -n security-lab
Switch back to admin context:
kubectl config use-context kubernetes-admin@kubernetes
Task 2: Enable API Server Encryption for Sensitive Data
Subtask 2.1: Create Encryption Configuration
Generate an encryption key:
head -c 32 /dev/urandom | base64
Create the encryption configuration file:
cat <<EOF > /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: $(head -c 32 /dev/urandom | base64)
  - identity: {}
EOF
Subtask 2.2: Configure API Server for Encryption
Backup the current API server manifest:
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml.backup
Edit the API server configuration to enable encryption:
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
Add the encryption configuration to the API server arguments: Add this line under the command section:
- --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
Add the volume mount for the encryption config: Add under volumeMounts:
- mountPath: /etc/kubernetes/encryption-config.yaml
  name: encryption-config
  readOnly: true
Add the volume definition: Add under volumes:
- hostPath:
    path: /etc/kubernetes/encryption-config.yaml
    type: File
  name: encryption-config
Subtask 2.3: Verify Encryption is Working
Wait for the API server to restart (this happens automatically):
kubectl get pods -n kube-system | grep kube-apiserver
Create a test secret:
kubectl create secret generic test-secret --from-literal=password=supersecret -n security-lab
Verify the secret is encrypted in etcd:
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/security-lab/test-secret | hexdump -C
The output should show encrypted data rather than plain text.

Subtask 2.4: Encrypt Existing Secrets
Force re-encryption of all existing secrets:
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
Verify encryption of existing secrets:
kubectl get secret -n kube-system | head -5
Task 3: Implement TLS Certificates to Secure API Communications
Subtask 3.1: Examine Current TLS Configuration
Check current API server certificate:
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 5 "Subject:"
Verify certificate validity:
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
Check certificate subject alternative names:
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 5 "Subject Alternative Name"
Subtask 3.2: Generate Custom TLS Certificates
Create a custom CA for demonstration:
mkdir -p ~/custom-certs
cd ~/custom-certs

# Generate CA private key
openssl genrsa -out ca.key 4096

# Generate CA certificate
openssl req -new -x509 -key ca.key -sha256 -subj "/C=US/ST=CA/O=MyOrg/CN=MyCA" -days 3650 -out ca.crt
Create a certificate signing request for the API server:
# Generate private key for API server
openssl genrsa -out apiserver-custom.key 4096

# Create certificate signing request
openssl req -new -key apiserver-custom.key -out apiserver-custom.csr -subj "/C=US/ST=CA/O=MyOrg/CN=kubernetes-api"
Create a certificate extensions file:
cat > apiserver-custom.ext <<EOF
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
DNS.1=kubernetes
DNS.2=kubernetes.default
DNS.3=kubernetes.default.svc
DNS.4=kubernetes.default.svc.cluster.local
DNS.5=localhost
IP.1=127.0.0.1
IP.2=10.96.0.1
EOF
Sign the certificate with our custom CA:
openssl x509 -req -in apiserver-custom.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out apiserver-custom.crt -days 365 -extensions v3_req -extfile apiserver-custom.ext
Subtask 3.3: Configure Client Certificate Authentication
Create a client certificate for secure communication:
# Generate client private key
openssl genrsa -out client.key 4096

# Create client certificate signing request
openssl req -new -key client.key -out client.csr -subj "/C=US/ST=CA/O=MyOrg/CN=api-client"

# Sign client certificate
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
Test the custom certificates:
# Verify certificate chain
openssl verify -CAfile ca.crt apiserver-custom.crt
openssl verify -CAfile ca.crt client.crt
Subtask 3.4: Configure Secure Communication
Create a kubeconfig file using the custom certificates:
cat > custom-kubeconfig.yaml <<EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: $(pwd)/ca.crt
    server: https://127.0.0.1:6443
  name: custom-cluster
contexts:
- context:
    cluster: custom-cluster
    user: custom-user
  name: custom-context
current-context: custom-context
users:
- name: custom-user
  user:
    client-certificate: $(pwd)/client.crt
    client-key: $(pwd)/client.key
EOF
Test secure communication with curl:
# Test API server accessibility with proper certificates
curl --cacert ca.crt --cert client.crt --key client.key https://127.0.0.1:6443/api/v1/namespaces
Subtask 3.5: Implement Certificate Rotation Strategy
Create a script for certificate monitoring:
cat > check-cert-expiry.sh <<'EOF'
#!/bin/bash

echo "Checking Kubernetes certificate expiration dates..."
echo "=================================================="

for cert in /etc/kubernetes/pki/*.crt; do
    if [ -f "$cert" ]; then
        echo "Certificate: $(basename $cert)"
        sudo openssl x509 -in "$cert" -noout -dates | grep "notAfter"
        echo "---"
    fi
done

echo "Checking etcd certificates..."
for cert in /etc/kubernetes/pki/etcd/*.crt; do
    if [ -f "$cert" ]; then
        echo "Certificate: $(basename $cert)"
        sudo openssl x509 -in "$cert" -noout -dates | grep "notAfter"
        echo "---"
    fi
done
EOF

chmod +x check-cert-expiry.sh
Run the certificate expiry check:
./check-cert-expiry.sh
Verification and Testing
Comprehensive Security Test
Create a comprehensive test script:
cat > security-test.sh <<'EOF'
#!/bin/bash

echo "=== Kubernetes API Server Security Test ==="
echo

# Test 1: RBAC Configuration
echo "1. Testing RBAC Configuration..."
kubectl auth can-i get pods --as=testuser -n security-lab
kubectl auth can-i delete pods --as=testuser -n security-lab
kubectl auth can-i get pods --as=testuser -n default
echo

# Test 2: Encryption at Rest
echo "2. Testing Encryption at Rest..."
kubectl create secret generic encryption-test --from-literal=data=sensitive-information -n security-lab
echo "Secret created. Checking if encrypted in etcd..."
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/security-lab/encryption-test | grep -q "sensitive-information"
if [ $? -eq 0 ]; then
    echo "WARNING: Secret appears to be stored in plain text!"
else
    echo "SUCCESS: Secret is encrypted in etcd"
fi
echo

# Test 3: TLS Configuration
echo "3. Testing TLS Configuration..."
echo "API Server Certificate Details:"
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -subject -dates
echo

echo "=== Security Test Complete ==="
EOF

chmod +x security-test.sh
Run the comprehensive security test:
./security-test.sh
Troubleshooting Common Issues
Issue 1: API Server Won't Start After Encryption Configuration
Symptoms: API server pod is not running after adding encryption configuration.

Solution:

# Check API server logs
sudo journalctl -u kubelet | grep apiserver

# Verify encryption config file syntax
sudo cat /etc/kubernetes/encryption-config.yaml

# Restore backup if needed
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml.backup /etc/kubernetes/manifests/kube-apiserver.yaml
Issue 2: RBAC Permissions Not Working
Symptoms: User can access resources they shouldn't be able to.

Solution:

# Check role bindings
kubectl describe rolebinding read-pods -n security-lab

# Verify user context
kubectl config current-context

# Check what the user can do
kubectl auth can-i --list --as=testuser -n security-lab
Issue 3: Certificate Verification Failures
Symptoms: TLS handshake failures or certificate verification errors.

Solution:

# Check certificate validity
openssl x509 -in /path/to/cert.crt -noout -dates

# Verify certificate chain
openssl verify -CAfile ca.crt client.crt

# Check certificate subject alternative names
openssl x509 -in /path/to/cert.crt -text -noout | grep -A 5 "Subject Alternative Name"
Cleanup
Remove test resources:
kubectl delete namespace security-lab
kubectl delete csr testuser-csr
kubectl config delete-context testuser-context
kubectl config delete-user testuser
Clean up certificate files:
rm -rf ~/custom-certs
rm -f testuser.key testuser.crt testuser.csr
rm -f security-test.sh check-cert-expiry.sh
Conclusion
In this lab, you have successfully implemented three critical security measures for the Kubernetes API server:

RBAC Configuration: You learned how to create custom roles with limited permissions, bind users to specific roles, and test access controls. This implements the principle of least privilege, ensuring users only have access to resources they absolutely need.

Encryption at Rest: You configured the API server to encrypt sensitive data stored in etcd, protecting secrets and configuration data even if someone gains access to the underlying storage. This is crucial for compliance and data protection requirements.

TLS Certificate Security: You implemented proper certificate management for secure API communications, including custom certificate generation, client certificate authentication, and certificate monitoring strategies.

These security measures are essential for production Kubernetes environments because they:

Prevent unauthorized access to cluster resources
Protect sensitive data from being exposed in storage
Ensure all communications with the API server are encrypted and authenticated
Provide audit trails for access control decisions
Meet compliance requirements for enterprise environments
Understanding and implementing these security controls is fundamental for anyone working with Kubernetes in production environments and is a key requirement for the Kubernetes and Cloud Native Security Associate (KCSA) certification.

The skills you've developed in this lab will help you secure real-world Kubernetes deployments and protect against common security vulnerabilities that can compromise entire cluster infrastructures.
