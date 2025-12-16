Lab 4: Helm Package Management
Objectives
By the end of this lab, you will be able to:

• Install and configure Helm on a Linux system • Set up and manage Helm repositories • Deploy applications using pre-built Helm charts • Customize Helm chart values to modify application behavior • Understand the relationship between Helm charts, values, and Kubernetes resources • Manage application lifecycle using Helm commands • Troubleshoot common Helm deployment issues

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (pods, services, deployments) • Familiarity with YAML file structure • Basic Linux command-line knowledge • Understanding of containerized applications • Access to a Kubernetes cluster (provided by Al Nafi cloud environment)

Note: Al Nafi provides ready-to-use Linux-based cloud machines with Kubernetes pre-installed. Simply click Start Lab to begin - no need to build your own VM or cluster.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with: • Ubuntu Linux operating system • Kubernetes cluster (single-node or multi-node) • kubectl command-line tool • Internet connectivity for downloading Helm and charts

Task 1: Install Helm and Set Up Repository
Subtask 1.1: Install Helm
First, let's install the latest version of Helm on your system.

Download and install Helm using the official installation script:
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
Verify the Helm installation:
helm version
Expected output should show Helm version 3.x.x:

version.BuildInfo{Version:"v3.13.0", GitCommit:"825e86f6a7a38cef1112bfa606e4127a706749b1", GitTreeState:"clean", GoVersion:"go1.20.8"}
Check Helm help to understand available commands:
helm help
Subtask 1.2: Add Helm Repositories
Helm repositories are collections of charts that you can install. Let's add some popular repositories.

Add the official Bitnami repository (contains many production-ready charts):
helm repo add bitnami https://charts.bitnami.com/bitnami
Add the official stable repository:
helm repo add stable https://charts.helm.sh/stable
Update repository information to get the latest chart versions:
helm repo update
List all configured repositories:
helm repo list
Expected output:

NAME    URL
bitnami https://charts.bitnami.com/bitnami
stable  https://charts.helm.sh/stable
Search for available charts in the repositories:
helm search repo nginx
This will show all nginx-related charts available in your configured repositories.

Task 2: Deploy a Sample Application Using Helm Chart
Subtask 2.1: Explore Available Charts
Search for Apache web server charts:
helm search repo apache
Get detailed information about the Bitnami Apache chart:
helm show chart bitnami/apache
View the default values for the Apache chart:
helm show values bitnami/apache
This command displays all configurable parameters and their default values.

Subtask 2.2: Deploy Apache Web Server
Create a namespace for our application:
kubectl create namespace helm-lab
Install Apache using Helm with a release name:
helm install my-apache bitnami/apache --namespace helm-lab
Key Terms:

Release: A specific deployment of a chart with a unique name
Chart: A package containing Kubernetes resource templates
Namespace: Kubernetes logical grouping for resources
Check the deployment status:
helm status my-apache --namespace helm-lab
List all Helm releases:
helm list --namespace helm-lab
Verify Kubernetes resources were created:
kubectl get all --namespace helm-lab
Subtask 2.3: Access the Application
Check the service created by Helm:
kubectl get svc --namespace helm-lab
Port-forward to access the Apache server locally:
kubectl port-forward --namespace helm-lab svc/my-apache 8080:80
Open a new terminal and test the connection:
curl http://localhost:8080
You should see the Apache default page HTML content.

Stop the port-forward by pressing Ctrl+C in the first terminal.
Task 3: Customize Helm Chart Values
Subtask 3.1: Modify Replica Count
Create a custom values file to override default settings:
cat > custom-values.yaml << EOF
replicaCount: 3

service:
  type: NodePort
  nodePorts:
    http: 30080

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
EOF
Upgrade the existing release with custom values:
helm upgrade my-apache bitnami/apache --namespace helm-lab --values custom-values.yaml
Verify the replica count increased:
kubectl get pods --namespace helm-lab
You should now see 3 Apache pods running.

Check the service type changed to NodePort:
kubectl get svc --namespace helm-lab
Subtask 3.2: Add Environment Variables
Update the custom values file to include environment variables:
cat > custom-values.yaml << EOF
replicaCount: 2

service:
  type: NodePort
  nodePorts:
    http: 30080

extraEnvVars:
  - name: APACHE_SERVER_NAME
    value: "my-custom-server"
  - name: ENVIRONMENT
    value: "development"

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
EOF
Apply the updated configuration:
helm upgrade my-apache bitnami/apache --namespace helm-lab --values custom-values.yaml
Verify environment variables are set in the pods:
kubectl exec --namespace helm-lab deployment/my-apache -- env | grep -E "(APACHE_SERVER_NAME|ENVIRONMENT)"
Subtask 3.3: Override Values Using Command Line
You can also override values directly from the command line without creating a file.

Scale down to 1 replica using command-line override:
helm upgrade my-apache bitnami/apache --namespace helm-lab --set replicaCount=1
Verify the change:
kubectl get pods --namespace helm-lab
Add multiple overrides in a single command:
helm upgrade my-apache bitnami/apache --namespace helm-lab \
  --set replicaCount=2 \
  --set service.type=ClusterIP \
  --set extraEnvVars[0].name=APP_ENV \
  --set extraEnvVars[0].value=production
Task 4: Helm Management Operations
Subtask 4.1: View Release History
Check the revision history of your release:
helm history my-apache --namespace helm-lab
This shows all the upgrades and changes made to the release.

Get detailed information about a specific revision:
helm get values my-apache --namespace helm-lab --revision 1
Subtask 4.2: Rollback Operations
Rollback to a previous revision if needed:
helm rollback my-apache 1 --namespace helm-lab
Verify the rollback:
helm history my-apache --namespace helm-lab
kubectl get pods --namespace helm-lab
Subtask 4.3: Template Debugging
Generate Kubernetes manifests without installing (dry-run):
helm template my-apache bitnami/apache --namespace helm-lab --values custom-values.yaml
This shows exactly what Kubernetes resources would be created.

Debug template rendering:
helm install my-apache-debug bitnami/apache --namespace helm-lab --dry-run --debug
Task 5: Clean Up Resources
Subtask 5.1: Uninstall Helm Release
Remove the Helm release:
helm uninstall my-apache --namespace helm-lab
Verify resources are removed:
kubectl get all --namespace helm-lab
Delete the namespace:
kubectl delete namespace helm-lab
Subtask 5.2: Clean Up Files
Remove custom values file:
rm custom-values.yaml
Troubleshooting Common Issues
Issue 1: Helm Installation Fails
Problem: Permission denied during Helm installation Solution:

sudo curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
Issue 2: Repository Update Fails
Problem: Unable to update repository Solution:

helm repo remove bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
Issue 3: Pod Fails to Start
Problem: Insufficient resources Solution: Check resource limits and node capacity:

kubectl describe pod <pod-name> --namespace helm-lab
kubectl top nodes
Issue 4: Values Not Applied
Problem: Custom values not taking effect Solution: Verify YAML syntax and use debug mode:

helm upgrade my-apache bitnami/apache --namespace helm-lab --values custom-values.yaml --debug
Key Concepts Summary
Helm Architecture:

Chart: Package containing Kubernetes resource templates
Release: Instance of a chart deployed to a cluster
Repository: Collection of charts
Values: Configuration parameters for charts
Important Commands:

helm install: Deploy a new release
helm upgrade: Update an existing release
helm rollback: Revert to a previous version
helm uninstall: Remove a release
helm list: Show all releases
Conclusion
In this lab, you have successfully:

• Installed Helm package manager and configured repositories • Deployed a web application using a pre-built Helm chart • Customized application behavior by modifying replica counts and environment variables • Learned to manage application lifecycle through upgrades and rollbacks • Understood the relationship between Helm charts, values, and Kubernetes resources

Why This Matters: Helm simplifies Kubernetes application management by providing templating, versioning, and package management capabilities. This is essential for production environments where you need to deploy, update, and manage complex applications consistently across different environments. These skills are fundamental for the CKAD certification and real-world Kubernetes operations.

Next Steps: Practice deploying different types of applications using various Helm charts, explore chart development, and learn about Helm hooks for advanced deployment scenarios.
