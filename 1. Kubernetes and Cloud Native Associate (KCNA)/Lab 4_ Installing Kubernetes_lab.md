Lab 4: Installing Kubernetes
Objectives
By the end of this lab, you will be able to:

• Set up a complete Kubernetes cluster using kubeadm on Linux systems • Configure the cluster networking with a Container Network Interface (CNI) plugin • Verify cluster installation by checking node statuses and system pods • Use kubectl commands to explore and interact with your Kubernetes cluster • Understand the basic architecture and components of a Kubernetes cluster • Troubleshoot common installation issues and verify cluster functionality

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Linux command line operations • Familiarity with containerization concepts (Docker basics) • Knowledge of networking fundamentals (IP addresses, ports) • Understanding of YAML file structure • Basic knowledge of system administration tasks

Technical Requirements: • Linux-based system with at least 2 GB RAM and 2 CPU cores • Root or sudo access • Internet connectivity for downloading packages • Basic text editor skills (nano, vim, or similar)

Lab Environment Setup
Good News! Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click Start Lab and you'll have access to a pre-configured environment. No need to build your own virtual machine or worry about hardware requirements.

Your lab environment includes: • Ubuntu 20.04 LTS or newer • Pre-installed Docker runtime • Network connectivity configured • Sufficient resources for Kubernetes installation

Task 1: Preparing the System for Kubernetes Installation
Subtask 1.1: Update System Packages
First, let's ensure our system is up to date with the latest packages.

# Update package index
sudo apt update

# Upgrade existing packages
sudo apt upgrade -y

# Install essential packages
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
Subtask 1.2: Disable Swap
Kubernetes requires swap to be disabled for optimal performance.

# Disable swap temporarily
sudo swapoff -a

# Disable swap permanently by commenting out swap entries
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is disabled
free -h
Expected Output: The swap line should show 0B for total, used, and free.

Subtask 1.3: Configure Kernel Modules
Load necessary kernel modules for Kubernetes networking.

# Load required modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Make modules persistent across reboots
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
Subtask 1.4: Configure System Parameters
Set up required sysctl parameters for Kubernetes.

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters without reboot
sudo sysctl --system
Task 2: Installing Container Runtime (containerd)
Subtask 2.1: Install containerd
Kubernetes needs a container runtime. We'll use containerd as it's the recommended choice.

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt update

# Install containerd
sudo apt install -y containerd.io
Subtask 2.2: Configure containerd
# Create containerd configuration directory
sudo mkdir -p /etc/containerd

# Generate default configuration
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Configure containerd to use systemd cgroup driver
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo systemctl status containerd
Expected Output: You should see active (running) status in green.

Task 3: Installing Kubernetes Components
Subtask 3.1: Add Kubernetes Repository
# Add Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package index
sudo apt update
Subtask 3.2: Install Kubernetes Tools
# Install kubelet, kubeadm, and kubectl
sudo apt install -y kubelet kubeadm kubectl

# Prevent automatic updates of Kubernetes packages
sudo apt-mark hold kubelet kubeadm kubectl

# Verify installation
kubeadm version
kubectl version --client
Expected Output: You should see version information for both tools.

Task 4: Initializing the Kubernetes Cluster
Subtask 4.1: Initialize the Master Node
# Initialize the cluster with kubeadm
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=$(hostname -I | awk '{print $1}')
Important: Save the kubeadm join command that appears at the end of the output. You'll need it to add worker nodes later.

Expected Output: You should see a message saying "Your Kubernetes control-plane has initialized successfully!"

Subtask 4.2: Configure kubectl for Regular User
# Create .kube directory
mkdir -p $HOME/.kube

# Copy admin configuration
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Change ownership of the config file
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify kubectl configuration
kubectl cluster-info
Expected Output: You should see cluster information including the Kubernetes master URL.

Task 5: Installing Pod Network Add-on
Subtask 5.1: Install Flannel CNI Plugin
Kubernetes needs a network plugin to enable pod-to-pod communication.

# Apply Flannel network plugin
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Wait for flannel pods to be ready
kubectl wait --for=condition=ready pod -l app=flannel -n kube-flannel --timeout=300s
Subtask 5.2: Remove Taint from Master Node (Single Node Cluster)
For a single-node cluster, we need to allow pods to be scheduled on the master node.

# Remove the master node taint
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# Verify the taint removal
kubectl describe nodes | grep -i taint
Expected Output: You should see NoSchedule taint removed or no taints listed.

Task 6: Verifying Cluster Installation
Subtask 6.1: Check Node Status
# Check node status
kubectl get nodes

# Get detailed node information
kubectl get nodes -o wide
Expected Output: Your node should show Ready status.

Subtask 6.2: Verify System Pods
# Check all system pods
kubectl get pods --all-namespaces

# Check pods in kube-system namespace specifically
kubectl get pods -n kube-system

# Wait for all pods to be running
kubectl wait --for=condition=ready pod --all -n kube-system --timeout=300s
Expected Output: All pods should show Running or Completed status.

Subtask 6.3: Check Cluster Components
# Check cluster component status
kubectl get componentstatuses

# View cluster information
kubectl cluster-info

# Check API server health
kubectl get --raw='/readyz?verbose'
Task 7: Exploring the Cluster with kubectl
Subtask 7.1: Basic Cluster Exploration
# List all namespaces
kubectl get namespaces

# Get cluster version information
kubectl version

# View cluster configuration
kubectl config view

# Check current context
kubectl config current-context
Subtask 7.2: Deploy a Test Application
Let's deploy a simple application to verify everything works.

# Create a test deployment
kubectl create deployment nginx-test --image=nginx:latest

# Expose the deployment as a service
kubectl expose deployment nginx-test --port=80 --type=NodePort

# Check deployment status
kubectl get deployments

# Check pods
kubectl get pods

# Check services
kubectl get services
Subtask 7.3: Verify Application Functionality
# Get detailed information about the nginx pod
kubectl describe pod $(kubectl get pods -l app=nginx-test -o jsonpath='{.items[0].metadata.name}')

# Check service details
kubectl get service nginx-test

# Test the application (get the NodePort)
NODE_PORT=$(kubectl get service nginx-test -o jsonpath='{.spec.ports[0].nodePort}')
curl http://localhost:$NODE_PORT
Expected Output: You should see the default nginx welcome page HTML.

Subtask 7.4: Clean Up Test Resources
# Delete the test deployment and service
kubectl delete deployment nginx-test
kubectl delete service nginx-test

# Verify cleanup
kubectl get deployments
kubectl get services
Task 8: Additional Cluster Verification
Subtask 8.1: Check Resource Usage
# Check node resource usage
kubectl top nodes

# If metrics-server is not installed, install it
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for metrics-server to be ready
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=300s
Subtask 8.2: Explore Kubernetes API
# List available API resources
kubectl api-resources

# Get API versions
kubectl api-versions

# Check cluster events
kubectl get events --sort-by=.metadata.creationTimestamp
Subtask 8.3: Verify Networking
# Check network policies (if any)
kubectl get networkpolicies --all-namespaces

# Verify DNS resolution
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default
Troubleshooting Common Issues
Issue 1: Pods Stuck in Pending State
# Check pod events
kubectl describe pod <pod-name>

# Check node resources
kubectl describe nodes

# Verify network plugin installation
kubectl get pods -n kube-flannel
Issue 2: kubelet Not Starting
# Check kubelet status
sudo systemctl status kubelet

# View kubelet logs
sudo journalctl -xeu kubelet

# Restart kubelet if needed
sudo systemctl restart kubelet
Issue 3: Network Issues
# Check containerd status
sudo systemctl status containerd

# Verify network configuration
ip route show

# Check firewall rules
sudo iptables -L
Verification Checklist
Before concluding this lab, verify the following:

 All system pods are running
 Node status shows Ready
 kubectl commands work without errors
 Test application deployed and accessible
 Network plugin (Flannel) is operational
 Cluster components are healthy
Conclusion
Congratulations! You have successfully completed Lab 4: Installing Kubernetes. In this comprehensive lab, you have accomplished the following:

Key Achievements:

• Cluster Setup: You've built a complete Kubernetes cluster from scratch using kubeadm, learning the fundamental installation process that forms the backbone of container orchestration.

• System Configuration: You've properly configured your Linux system with all necessary prerequisites, including container runtime setup, kernel modules, and system parameters required for Kubernetes operation.

• Network Implementation: You've installed and configured Flannel as your Container Network Interface (CNI) plugin, enabling seamless pod-to-pod communication across your cluster.

• Verification Skills: You've learned essential kubectl commands to monitor, verify, and troubleshoot your Kubernetes installation, skills that are crucial for day-to-day cluster management.

• Practical Application: You've deployed and tested a real application on your cluster, demonstrating that your installation is fully functional and ready for production workloads.

Why This Matters:

Understanding how to install and configure Kubernetes manually is essential for several reasons:

• Foundation Knowledge: This hands-on experience provides deep understanding of Kubernetes architecture and components • Troubleshooting Skills: Manual installation knowledge helps you diagnose and fix issues in any Kubernetes environment • Certification Preparation: These skills directly support your KCNA (Kubernetes and Cloud Native Associate) certification goals • Career Advancement: Kubernetes expertise is highly valued in the current job market, with manual installation skills demonstrating advanced technical competency

Next Steps:

With your Kubernetes cluster now operational, you're ready to explore advanced topics such as: • Deploying multi-tier applications • Implementing persistent storage • Configuring ingress controllers • Setting up monitoring and logging • Implementing security policies

Your newly installed Kubernetes cluster serves as the foundation for all future container orchestration learning and experimentation. Well done on completing this critical milestone in your Kubernetes journey!
