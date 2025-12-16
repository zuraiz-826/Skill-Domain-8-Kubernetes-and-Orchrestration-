Lab 5: Setting Up a Single-Node Kubernetes Cluster with Minikube
Objectives
By the end of this lab, you will be able to:

• Install and configure Minikube on a Linux system • Start and manage a single-node Kubernetes cluster • Use kubectl to interact with and verify cluster health • Understand basic Kubernetes cluster components and their status • Stop and restart a Minikube cluster while maintaining persistence • Troubleshoot common Minikube installation and startup issues

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Linux command line operations • Familiarity with containerization concepts (Docker basics) • Understanding of what Kubernetes is and its basic architecture • Knowledge of YAML file structure (helpful but not required) • Access to a terminal or command prompt

Note: Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click Start Lab to access your pre-configured environment - no need to build your own VM or install an operating system.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with: • Ubuntu 20.04 LTS or newer • Docker runtime installed and configured • Internet connectivity for downloading required packages • Sufficient resources (2 CPU cores, 4GB RAM minimum)

Task 1: Installing Minikube
Subtask 1.1: Update System Packages
First, ensure your system is up to date with the latest packages.

sudo apt update && sudo apt upgrade -y
Subtask 1.2: Install Required Dependencies
Install curl and other essential tools needed for Minikube installation.

sudo apt install -y curl wget apt-transport-https
Subtask 1.3: Download and Install Minikube
Download the latest stable version of Minikube for Linux.

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
Make the downloaded file executable and move it to your system PATH.

sudo install minikube-linux-amd64 /usr/local/bin/minikube
Verify the installation by checking the Minikube version.

minikube version
Expected Output:

minikube version: v1.32.0
commit: 8220a6eb95f0a4d75f7f2d7b14cef975f050512d
Subtask 1.4: Install kubectl
kubectl is the command-line tool for interacting with Kubernetes clusters. Download and install it.

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
Make kubectl executable and move it to your PATH.

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
Verify kubectl installation.

kubectl version --client
Expected Output:

Client Version: v1.29.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Task 2: Starting Your First Minikube Cluster
Subtask 2.1: Start Minikube with Docker Driver
Start your single-node Kubernetes cluster using Docker as the container runtime.

minikube start --driver=docker
Note: The first startup may take 3-5 minutes as it downloads the Kubernetes components and container images.

Expected Output:

😄  minikube v1.32.0 on Ubuntu 20.04
✨  Using the docker driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=4000MB) ...
🐳  Preparing Kubernetes v1.28.3 on Docker 24.0.7 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
Subtask 2.2: Verify Cluster Status
Check that your Minikube cluster is running properly.

minikube status
Expected Output:

minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
Subtask 2.3: Configure kubectl Context
Ensure kubectl is configured to communicate with your Minikube cluster.

kubectl config current-context
Expected Output:

minikube
Task 3: Verifying Cluster Health and Resources
Subtask 3.1: Check Cluster Information
Get basic information about your Kubernetes cluster.

kubectl cluster-info
Expected Output:

Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
Subtask 3.2: List All Nodes
View all nodes in your cluster (should show one node since this is a single-node setup).

kubectl get nodes
Expected Output:

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.28.3
Get detailed information about your node.

kubectl describe node minikube
Subtask 3.3: Check System Pods
View all system pods running in the kube-system namespace.

kubectl get pods -n kube-system
Expected Output:

NAME                               READY   STATUS    RESTARTS   AGE
coredns-5dd5756b68-xxxxx          1/1     Running   0          3m
etcd-minikube                     1/1     Running   0          3m
kube-apiserver-minikube           1/1     Running   0          3m
kube-controller-manager-minikube  1/1     Running   0          3m
kube-proxy-xxxxx                  1/1     Running   0          3m
kube-scheduler-minikube           1/1     Running   0          3m
storage-provisioner               1/1     Running   0          3m
Subtask 3.4: Check Available Resources
View the resource usage of your cluster.

kubectl top node
If the metrics server is not available, you can install it:

minikube addons enable metrics-server
Wait a few minutes, then try again:

kubectl top node
Subtask 3.5: List Available Namespaces
See all namespaces in your cluster.

kubectl get namespaces
Expected Output:

NAME              STATUS   AGE
default           Active   5m
kube-node-lease   Active   5m
kube-public       Active   5m
kube-system       Active   5m
Task 4: Testing Cluster Functionality
Subtask 4.1: Deploy a Test Application
Create a simple nginx deployment to test your cluster.

kubectl create deployment hello-minikube --image=nginx:latest
Check the deployment status.

kubectl get deployments
Expected Output:

NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1/1     1            1           30s
Subtask 4.2: Expose the Deployment
Create a service to expose your deployment.

kubectl expose deployment hello-minikube --type=NodePort --port=80
Check the service.

kubectl get services
Expected Output:

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-minikube   NodePort    10.96.xxx.xxx   <none>        80:xxxxx/TCP   15s
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP        8m
Subtask 4.3: Access the Application
Get the URL to access your application.

minikube service hello-minikube --url
Test the application using curl.

curl $(minikube service hello-minikube --url)
Clean up the test deployment.

kubectl delete deployment hello-minikube
kubectl delete service hello-minikube
Task 5: Stopping and Restarting the Cluster
Subtask 5.1: Stop the Minikube Cluster
Stop your cluster while preserving its state.

minikube stop
Expected Output:

✋  Stopping node "minikube"  ...
🛑  Powering off "minikube" via SSH ...
🛑  1 node stopped.
Verify the cluster is stopped.

minikube status
Expected Output:

minikube
type: Control Plane
host: Stopped
kubelet: Stopped
apiserver: Stopped
kubeconfig: Configured
Subtask 5.2: Restart the Cluster
Start the cluster again to verify persistence.

minikube start
Expected Output:

😄  minikube v1.32.0 on Ubuntu 20.04
✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.28.3 on Docker 24.0.7 ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
Subtask 5.3: Verify Cluster Persistence
Check that your cluster components are still running after restart.

kubectl get nodes
kubectl get pods -n kube-system
Verify that any persistent volumes or configurations remain intact.

kubectl get pv
kubectl get storageclass
Task 6: Exploring Minikube Features
Subtask 6.1: Access Kubernetes Dashboard
Enable the Kubernetes dashboard addon.

minikube addons enable dashboard
Start the dashboard in a separate terminal or background process.

minikube dashboard --url
Note: This will provide a URL to access the Kubernetes dashboard in a web browser.

Subtask 6.2: View Available Addons
List all available Minikube addons.

minikube addons list
Enable a useful addon like ingress.

minikube addons enable ingress
Verify the addon is running.

kubectl get pods -n ingress-nginx
Subtask 6.3: Check Minikube Configuration
View your current Minikube configuration.

minikube config view
Check the IP address of your Minikube cluster.

minikube ip
Troubleshooting Common Issues
Issue 1: Minikube Won't Start
If Minikube fails to start, try these solutions:

# Check if Docker is running
sudo systemctl status docker

# Start Docker if it's not running
sudo systemctl start docker

# Delete and recreate the cluster
minikube delete
minikube start --driver=docker
Issue 2: kubectl Commands Not Working
If kubectl commands fail:

# Check if kubectl is configured correctly
kubectl config current-context

# If not pointing to minikube, set it manually
kubectl config use-context minikube
Issue 3: Insufficient Resources
If you encounter resource issues:

# Start with more resources
minikube delete
minikube start --driver=docker --memory=4096 --cpus=2
Issue 4: Network Issues
If you have network connectivity problems:

# Check Minikube status
minikube status

# Restart with different network settings
minikube delete
minikube start --driver=docker --network-plugin=cni
Lab Cleanup
When you're finished with the lab, you can clean up resources:

# Stop the cluster
minikube stop

# Delete the cluster completely (optional)
minikube delete

# Remove downloaded binaries (optional)
sudo rm /usr/local/bin/minikube
sudo rm /usr/local/bin/kubectl
Conclusion
Congratulations! You have successfully completed Lab 5: Setting Up a Single-Node Kubernetes Cluster with Minikube.

What You Accomplished
In this lab, you:

• Installed Minikube and kubectl - Set up the essential tools for running a local Kubernetes cluster • Created a single-node Kubernetes cluster - Launched a fully functional Kubernetes environment on a single machine • Verified cluster health - Used kubectl commands to check cluster status, nodes, and system components • Tested cluster functionality - Deployed and exposed a sample application to ensure everything works correctly • Managed cluster lifecycle - Learned how to stop and restart your cluster while maintaining persistence • Explored Minikube features - Discovered addons and additional capabilities available in Minikube

Why This Matters
Understanding how to set up and manage a Kubernetes cluster is fundamental for:

• Development and Testing - Minikube provides a safe environment to develop and test Kubernetes applications locally • Learning Kubernetes - Having a local cluster allows you to experiment with Kubernetes concepts without cloud costs • KCNA Certification - This knowledge directly supports the Kubernetes and Cloud Native Associate certification objectives • Career Development - Kubernetes skills are highly sought after in the cloud computing industry • Production Readiness - The concepts learned here scale up to managing production Kubernetes clusters

Next Steps
Now that you have a working Minikube cluster, you can:

• Deploy more complex applications with multiple pods and services • Explore Kubernetes networking and storage concepts • Practice with ConfigMaps, Secrets, and other Kubernetes resources • Learn about Helm charts for application packaging • Experiment with different Kubernetes deployment strategies

This foundational knowledge prepares you for more advanced Kubernetes topics and real-world container orchestration scenarios.
