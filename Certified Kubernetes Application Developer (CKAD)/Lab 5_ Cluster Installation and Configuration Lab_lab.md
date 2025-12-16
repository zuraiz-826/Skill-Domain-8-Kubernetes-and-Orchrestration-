Lab 5: Cluster Installation and Configuration Lab
Objectives
By the end of this lab, students will be able to:

• Set up a multi-node Kubernetes cluster using kubeadm • Configure cluster control plane and worker nodes • Understand the cluster initialization process and node joining procedures • Perform etcd backup operations using etcdctl • Restore cluster state from etcd backups • Troubleshoot common cluster setup issues • Verify cluster health and functionality

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Linux command line operations • Familiarity with containerization concepts (Docker) • Knowledge of basic networking concepts • Understanding of YAML file structure • Previous experience with Kubernetes basic concepts (pods, services, deployments)

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own virtual machines or install additional software.

Your lab environment includes: • 3 Ubuntu 20.04 LTS machines (1 control plane, 2 worker nodes) • Pre-installed Docker runtime • Pre-installed kubeadm, kubelet, and kubectl tools • Network connectivity between all nodes

Task 1: Kubernetes Cluster Setup with kubeadm
Subtask 1.1: Verify Environment and Prerequisites
First, let's verify that all required components are installed and ready on all nodes.

Step 1: Connect to the control plane node (master node)

# Check if kubeadm is installed
kubeadm version

# Check if kubelet is installed
kubelet --version

# Check if kubectl is installed
kubectl version --client

# Check if Docker is running
sudo systemctl status docker
Step 2: Verify the same on worker nodes

# SSH to worker node 1
ssh worker1

# Run the same verification commands
kubeadm version
kubelet --version
kubectl version --client
sudo systemctl status docker

# Exit back to control plane
exit

# SSH to worker node 2
ssh worker2

# Run verification commands again
kubeadm version
kubelet --version
kubectl version --client
sudo systemctl status docker

# Exit back to control plane
exit
Subtask 1.2: Initialize the Kubernetes Control Plane
Step 1: Initialize the cluster on the control plane node

# Initialize the cluster with kubeadm
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$(hostname -i)

# Note: Save the join command that appears at the end of the output
# It will look like: kubeadm join <IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
Step 2: Configure kubectl for the current user

# Create .kube directory
mkdir -p $HOME/.kube

# Copy admin configuration
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Change ownership
sudo chown $(id -u):$(id -g) $HOME/.kube/config
Step 3: Verify the control plane is running

# Check cluster info
kubectl cluster-info

# Check node status
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
Subtask 1.3: Install Pod Network Add-on
Step 1: Install Flannel network add-on

# Apply Flannel network configuration
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Wait for flannel pods to be ready
kubectl get pods -n kube-flannel
Step 2: Verify network setup

# Check if control plane node is ready
kubectl get nodes

# Check all system pods are running
kubectl get pods -n kube-system
Task 2: Configure Worker Nodes
Subtask 2.1: Join Worker Node 1 to the Cluster
Step 1: Connect to worker node 1

# SSH to worker node 1
ssh worker1
Step 2: Join the cluster using the command from kubeadm init output

# Use the join command from the control plane initialization
# Replace <IP>, <token>, and <hash> with actual values from your init output
sudo kubeadm join <CONTROL_PLANE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

# Exit back to control plane
exit
Subtask 2.2: Join Worker Node 2 to the Cluster
Step 1: Connect to worker node 2

# SSH to worker node 2
ssh worker2
Step 2: Join the cluster

# Use the same join command
sudo kubeadm join <CONTROL_PLANE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

# Exit back to control plane
exit
Subtask 2.3: Verify Cluster Configuration
Step 1: Check all nodes are joined and ready

# Check node status
kubectl get nodes

# Get detailed node information
kubectl get nodes -o wide

# Check node labels
kubectl get nodes --show-labels
Step 2: Test cluster functionality

# Create a test deployment
kubectl create deployment nginx-test --image=nginx

# Scale the deployment
kubectl scale deployment nginx-test --replicas=3

# Check pod distribution across nodes
kubectl get pods -o wide

# Clean up test deployment
kubectl delete deployment nginx-test
Task 3: etcd Backup and Restore Operations
Subtask 3.1: Install and Configure etcdctl
Step 1: Install etcdctl if not already available

# Check if etcdctl is available
which etcdctl

# If not installed, download and install
ETCD_VER=v3.5.9
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd-${ETCD_VER}-linux-amd64.tar.gz

tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
rm -rf etcd-${ETCD_VER}-linux-amd64*
Step 2: Set up etcdctl environment variables

# Export etcdctl API version
export ETCDCTL_API=3

# Set etcd endpoints
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379

# Set certificate paths
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
Subtask 3.2: Create Sample Data for Backup Testing
Step 1: Create some resources to backup

# Create a namespace
kubectl create namespace backup-test

# Create a configmap
kubectl create configmap test-config --from-literal=key1=value1 --from-literal=key2=value2 -n backup-test

# Create a secret
kubectl create secret generic test-secret --from-literal=username=admin --from-literal=password=secret123 -n backup-test

# Create a deployment
kubectl create deployment test-app --image=nginx -n backup-test

# Verify resources exist
kubectl get all,configmap,secret -n backup-test
Subtask 3.3: Perform etcd Backup
Step 1: Create backup directory

# Create backup directory
sudo mkdir -p /opt/etcd-backup
sudo chown $(whoami):$(whoami) /opt/etcd-backup
Step 2: Perform the backup

# Create etcd snapshot
sudo etcdctl snapshot save /opt/etcd-backup/etcd-backup-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=$ETCDCTL_ENDPOINTS \
  --cacert=$ETCDCTL_CACERT \
  --cert=$ETCDCTL_CERT \
  --key=$ETCDCTL_KEY

# Verify backup file
ls -la /opt/etcd-backup/

# Check backup status
sudo etcdctl snapshot status /opt/etcd-backup/etcd-backup-*.db \
  --endpoints=$ETCDCTL_ENDPOINTS \
  --cacert=$ETCDCTL_CACERT \
  --cert=$ETCDCTL_CERT \
  --key=$ETCDCTL_KEY
Subtask 3.4: Simulate Data Loss and Restore
Step 1: Delete the test resources to simulate data loss

# Delete the test namespace and all its resources
kubectl delete namespace backup-test

# Verify resources are gone
kubectl get namespace backup-test
Step 2: Stop etcd and kubelet services

# Stop kubelet
sudo systemctl stop kubelet

# Move current etcd data
sudo mv /var/lib/etcd /var/lib/etcd.backup
Step 3: Restore from backup

# Restore etcd from backup
sudo etcdctl snapshot restore /opt/etcd-backup/etcd-backup-*.db \
  --data-dir=/var/lib/etcd \
  --endpoints=$ETCDCTL_ENDPOINTS \
  --cacert=$ETCDCTL_CACERT \
  --cert=$ETCDCTL_CERT \
  --key=$ETCDCTL_KEY

# Fix ownership
sudo chown -R etcd:etcd /var/lib/etcd
Step 4: Restart services and verify restore

# Start kubelet
sudo systemctl start kubelet

# Wait for cluster to be ready
sleep 30

# Check cluster status
kubectl get nodes

# Verify restored resources
kubectl get namespace backup-test
kubectl get all,configmap,secret -n backup-test
Task 4: Cluster Health Verification and Troubleshooting
Subtask 4.1: Comprehensive Cluster Health Check
Step 1: Check cluster components

# Check cluster info
kubectl cluster-info

# Check component status
kubectl get componentstatuses

# Check all nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system
Step 2: Check etcd health

# Check etcd member list
sudo etcdctl member list \
  --endpoints=$ETCDCTL_ENDPOINTS \
  --cacert=$ETCDCTL_CACERT \
  --cert=$ETCDCTL_CERT \
  --key=$ETCDCTL_KEY

# Check etcd endpoint health
sudo etcdctl endpoint health \
  --endpoints=$ETCDCTL_ENDPOINTS \
  --cacert=$ETCDCTL_CACERT \
  --cert=$ETCDCTL_CERT \
  --key=$ETCDCTL_KEY
Subtask 4.2: Test Cluster Functionality
Step 1: Deploy a test application

# Create a test deployment
kubectl create deployment cluster-test --image=nginx --replicas=3

# Expose the deployment
kubectl expose deployment cluster-test --port=80 --type=NodePort

# Check deployment status
kubectl get deployment cluster-test
kubectl get pods -l app=cluster-test -o wide
kubectl get service cluster-test
Step 2: Test service connectivity

# Get service details
kubectl get service cluster-test

# Test internal connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- cluster-test

# Clean up test resources
kubectl delete deployment cluster-test
kubectl delete service cluster-test
Troubleshooting Common Issues
Issue 1: Node Not Ready
Symptoms: Node shows "NotReady" status

Solution:

# Check node conditions
kubectl describe node <node-name>

# Check kubelet logs
sudo journalctl -u kubelet -f

# Restart kubelet if needed
sudo systemctl restart kubelet
Issue 2: Pod Network Issues
Symptoms: Pods cannot communicate with each other

Solution:

# Check flannel pods
kubectl get pods -n kube-flannel

# Restart flannel if needed
kubectl delete pods -n kube-flannel -l app=flannel

# Check network configuration
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
Issue 3: etcd Backup/Restore Failures
Symptoms: etcdctl commands fail with certificate errors

Solution:

# Verify certificate paths
ls -la /etc/kubernetes/pki/etcd/

# Check etcd pod status
kubectl get pods -n kube-system | grep etcd

# Verify environment variables
echo $ETCDCTL_ENDPOINTS
echo $ETCDCTL_CACERT
Conclusion
In this comprehensive lab, you have successfully:

• Set up a complete Kubernetes cluster using kubeadm, learning the fundamental process of cluster initialization and node joining • Configured both control plane and worker nodes, understanding the roles and responsibilities of each component • Implemented pod networking with Flannel, enabling communication between pods across different nodes • Mastered etcd backup and restore operations, which are critical skills for maintaining cluster data integrity and disaster recovery • Performed cluster health verification, learning how to monitor and troubleshoot cluster issues

These skills are essential for the Certified Kubernetes Administrator (CKA) certification and real-world Kubernetes operations. Understanding cluster setup, maintenance, and backup procedures ensures you can manage production Kubernetes environments effectively and handle critical situations like data recovery.

The hands-on experience with kubeadm, etcdctl, and cluster troubleshooting provides you with practical knowledge that directly applies to enterprise Kubernetes deployments. Remember that regular backups and cluster health monitoring are crucial practices for maintaining reliable Kubernetes infrastructure.
