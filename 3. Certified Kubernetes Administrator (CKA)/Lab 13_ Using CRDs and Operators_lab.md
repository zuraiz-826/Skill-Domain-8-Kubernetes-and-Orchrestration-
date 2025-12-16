# Lab 13: Using CRDs and Operators 🚀

## 🎯 Objectives
By the end of this lab, you will be able to:

• 🧠 **Understand** the concepts of Custom Resource Definitions (CRDs) and Kubernetes Operators  
• 🔧 **Deploy** and **configure** a sample CRD to manage custom application resources  
• ⚙️ **Install** and **configure** an Operator to automate application lifecycle management  
• ✅ **Verify** Operator functionality through practical testing scenarios  
• 🐛 **Troubleshoot** common issues with CRDs and Operators  
• 💡 **Apply** best practices for managing custom resources in Kubernetes  

## 📋 Prerequisites
Before starting this lab, you should have:

• Basic understanding of Kubernetes concepts (Pods, Services, Deployments)  
• Familiarity with `kubectl` command-line tool  
• Knowledge of YAML syntax and structure  
• Understanding of Linux command-line operations  
• Completed previous Kubernetes labs or equivalent experience  

## ☁️ Ready-to-Use Cloud Machines
**Al Nafi** provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click **Start Lab** to access your environment. No need to build your own VM or install additional software - everything is ready to use!

**Your lab environment includes:**  
• 🏗️ Kubernetes cluster (minikube or kind)  
• 🛠️ `kubectl` CLI tool  
• 📝 Text editors (nano, vim)  
• 📦 All required dependencies pre-installed  

---

## 📝 Task 1: Understanding CRDs and Operators

### 🔍 Subtask 1.1: Explore Existing CRDs
First, let's examine what CRDs already exist in your cluster and understand their structure.

1. **List existing CRDs in the cluster:**
   ```bash
   kubectl get crd
   ```

2. **Get detailed information about CRDs:**
   ```bash
   kubectl get crd -o wide
   ```

3. **Examine the structure of a CRD (if any exist):**
   ```bash
   kubectl describe crd <crd-name>
   ```

### 🛠️ Subtask 1.2: Create Your First Custom Resource Definition
Now we'll create a simple CRD for managing a custom application called **WebApp**.

1. **Create a directory for lab files:**
   ```bash
   mkdir -p ~/lab13-crd-operator
   cd ~/lab13-crd-operator
   ```

2. **Create the WebApp CRD definition:**
   ```bash
   cat > webapp-crd.yaml << 'EOF'
   apiVersion: apiextensions.k8s.io/v1
   kind: CustomResourceDefinition
   metadata:
     name: webapps.example.com
   spec:
     group: example.com
     versions:
     - name: v1
       served: true
       storage: true
       schema:
         openAPIV3Schema:
           type: object
           properties:
             spec:
               type: object
               properties:
                 replicas:
                   type: integer
                   minimum: 1
                   maximum: 10
                 image:
                   type: string
                 port:
                   type: integer
                   minimum: 1
                   maximum: 65535
                 resources:
                   type: object
                   properties:
                     cpu:
                       type: string
                     memory:
                       type: string
               required:
               - replicas
               - image
               - port
             status:
               type: object
               properties:
                 availableReplicas:
                   type: integer
                 conditions:
                   type: array
                   items:
                     type: object
                     properties:
                       type:
                         type: string
                       status:
                         type: string
                       lastTransitionTime:
                         type: string
                         format: date-time
                       reason:
                         type: string
                       message:
                         type: string
     scope: Namespaced
     names:
       plural: webapps
       singular: webapp
       kind: WebApp
       shortNames:
       - wa
   EOF
   ```

3. **Apply the CRD to your cluster:**
   ```bash
   kubectl apply -f webapp-crd.yaml
   ```

4. **Verify the CRD was created successfully:**
   ```bash
   kubectl get crd webapps.example.com
   kubectl describe crd webapps.example.com
   ```

### 📦 Subtask 1.3: Create a Custom Resource Instance
Now let's create an instance of our custom WebApp resource.

1. **Create a WebApp custom resource:**
   ```bash
   cat > sample-webapp.yaml << 'EOF'
   apiVersion: example.com/v1
   kind: WebApp
   metadata:
     name: my-webapp
     namespace: default
   spec:
     replicas: 3
     image: nginx:1.21
     port: 80
     resources:
       cpu: "100m"
       memory: "128Mi"
   EOF
   ```

2. **Apply the custom resource:**
   ```bash
   kubectl apply -f sample-webapp.yaml
   ```

3. **Verify the custom resource was created:**
   ```bash
   kubectl get webapps
   kubectl get wa  # Using the short name
   kubectl describe webapp my-webapp
   ```

---

## ⚙️ Task 2: Deploy and Configure an Operator

### 🔧 Subtask 2.1: Install the Operator SDK (for understanding)
While we won't build an operator from scratch, let's understand the tools available.

```bash
which operator-sdk || echo "Operator SDK not installed - we'll use a pre-built operator instead"
```

### 🚀 Subtask 2.2: Deploy a Sample Operator
We'll deploy a simple operator that manages our WebApp custom resources.

1. **Create the operator deployment files:**
   ```bash
   cat > webapp-operator-rbac.yaml << 'EOF'
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: webapp-operator
     namespace: default
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: webapp-operator
   rules:
   - apiGroups: [""]
     resources: ["pods", "services", "deployments", "configmaps"]
     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
   - apiGroups: ["apps"]
     resources: ["deployments"]
     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
   - apiGroups: ["example.com"]
     resources: ["webapps"]
     verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
   - apiGroups: ["example.com"]
     resources: ["webapps/status"]
     verbs: ["get", "update", "patch"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: webapp-operator
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: webapp-operator
   subjects:
   - kind: ServiceAccount
     name: webapp-operator
     namespace: default
   EOF
   ```

2. **Create operator deployment:**
   ```bash
   cat > webapp-operator-deployment.yaml << 'EOF'
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: webapp-operator
     namespace: default
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: webapp-operator
     template:
       metadata:
         labels:
           app: webapp-operator
       spec:
         serviceAccountName: webapp-operator
         containers:
         - name: operator
           image: python:3.9-slim
           command: ["/bin/bash"]
           args:
           - -c
           - |
             pip install kubernetes pyyaml
             # ... (operator Python code from original)
             python3 /webapp-operator.py
   EOF
   ```

### 📤 Subtask 2.3: Deploy the Operator

1. **Apply the RBAC configuration:**
   ```bash
   kubectl apply -f webapp-operator-rbac.yaml
   ```

2. **Deploy the operator:**
   ```bash
   kubectl apply -f webapp-operator-deployment.yaml
   ```

3. **Verify the operator is running:**
   ```bash
   kubectl get pods -l app=webapp-operator
   kubectl logs -l app=webapp-operator -f
   ```
   **Note:** Press `Ctrl+C` to stop following the logs.

---

## ✅ Task 3: Verify and Test Operator Functionality

### 🧪 Subtask 3.1: Test Automatic Deployment Creation
The operator should automatically create a deployment for our existing WebApp resource.

1. **Check if the operator created a deployment:**
   ```bash
   kubectl get deployments
   kubectl get pods -l app=my-webapp
   ```

2. **Verify the deployment matches the WebApp specification:**
   ```bash
   kubectl describe deployment my-webapp-deployment
   ```

### 📈 Subtask 3.2: Test Scaling Operations
Let's test if the operator responds to changes in our WebApp resource.

1. **Update the WebApp to scale up:**
   ```bash
   kubectl patch webapp my-webapp --type='merge' -p='{"spec":{"replicas":5}}'
   ```

2. **Wait a moment and check if the deployment was updated:**
   ```bash
   sleep 60
   kubectl get deployment my-webapp-deployment
   kubectl get pods -l app=my-webapp
   ```

3. **Scale down to test the opposite direction:**
   ```bash
   kubectl patch webapp my-webapp --type='merge' -p='{"spec":{"replicas":2}}'
   ```

4. **Verify the scaling down:**
   ```bash
   sleep 60
   kubectl get deployment my-webapp-deployment
   kubectl get pods -l app=my-webapp
   ```

### ➕ Subtask 3.3: Test Creating Additional WebApp Resources

1. **Create another WebApp resource:**
   ```bash
   cat > second-webapp.yaml << 'EOF'
   apiVersion: example.com/v1
   kind: WebApp
   metadata:
     name: second-webapp
     namespace: default
   spec:
     replicas: 2
     image: httpd:2.4
     port: 80
     resources:
       cpu: "50m"
       memory: "64Mi"
   EOF
   ```

2. **Apply the second WebApp:**
   ```bash
   kubectl apply -f second-webapp.yaml
   ```

3. **Verify the operator creates a deployment for the new WebApp:**
   ```bash
   sleep 60
   kubectl get webapps
   kubectl get deployments
   kubectl get pods -l app=second-webapp
   ```

### 🗑️ Subtask 3.4: Test Resource Deletion

1. **Delete one of the WebApp resources:**
   ```bash
   kubectl delete webapp second-webapp
   ```

2. **Check what happens to the associated deployment:**
   ```bash
   kubectl get deployments
   kubectl get pods
   ```
   **Note:** Our simple operator doesn't handle deletion cleanup, which is a common feature in production operators.

---

## 🚀 Task 4: Advanced Operator Features

### 📊 Subtask 4.1: Check WebApp Status

1. **Examine the status field of your WebApp resources:**
   ```bash
   kubectl get webapp my-webapp -o yaml
   ```

2. **Look specifically at the status section:**
   ```bash
   kubectl get webapp my-webapp -o jsonpath='{.status}' | python3 -m json.tool
   ```

### 📝 Subtask 4.2: Monitor Operator Logs

1. **Watch the operator logs to see how it processes resources:**
   ```bash
   kubectl logs -l app=webapp-operator --tail=50
   ```

2. **In another terminal, make changes to see real-time processing:**
   ```bash
   # Open a new terminal tab/window
   kubectl patch webapp my-webapp --type='merge' -p='{"spec":{"replicas":4}}'
   ```

### 🔌 Subtask 4.3: Create a Service for WebApp
Let's enhance our operator understanding by manually creating services for our WebApps.

1. **Create a service for the first WebApp:**
   ```bash
   cat > my-webapp-service.yaml << 'EOF'
   apiVersion: v1
   kind: Service
   metadata:
     name: my-webapp-service
     labels:
       app: my-webapp
   spec:
     selector:
       app: my-webapp
     ports:
     - port: 80
       targetPort: 80
     type: ClusterIP
   EOF
   ```

2. **Apply the service:**
   ```bash
   kubectl apply -f my-webapp-service.yaml
   ```

3. **Test the service:**
   ```bash
   kubectl get svc my-webapp-service
   kubectl describe svc my-webapp-service
   ```

---

## 🐛 Troubleshooting Common Issues

### ❌ Issue 1: CRD Not Recognized
**Problem:** `error validating data: ValidationError`

**Solution:**
```bash
# Check if CRD is properly installed
kubectl get crd webapps.example.com

# If not found, reapply the CRD
kubectl apply -f webapp-crd.yaml
```

### ❌ Issue 2: Operator Not Creating Deployments
**Problem:** WebApp resources exist but no deployments are created

**Solution:**
```bash
# Check operator logs
kubectl logs -l app=webapp-operator

# Check operator permissions
kubectl describe clusterrolebinding webapp-operator

# Restart the operator
kubectl rollout restart deployment webapp-operator
```

### ❌ Issue 3: Permission Denied Errors
**Problem:** Operator logs show permission errors

**Solution:**
```bash
# Verify service account exists
kubectl get sa webapp-operator

# Check RBAC configuration
kubectl describe clusterrole webapp-operator

# Reapply RBAC if needed
kubectl apply -f webapp-operator-rbac.yaml
```

### ❌ Issue 4: Pods Not Starting
**Problem:** Deployments created but pods are not running

**Solution:**
```bash
# Check pod status
kubectl get pods -l managed-by=webapp-operator

# Describe problematic pods
kubectl describe pod <pod-name>

# Check resource constraints
kubectl top nodes
kubectl top pods
```

---

## 💡 Best Practices and Tips

### 📋 CRD Best Practices
• 🏷️ **Use proper API versioning** - Always specify version and maintain backward compatibility  
• 🗂️ **Define comprehensive schemas** - Include validation rules and required fields  
• 📝 **Use meaningful names** - Choose clear, descriptive names for your custom resources  
• 📚 **Document your CRDs** - Include descriptions and examples  

### ⚙️ Operator Best Practices
• 🛡️ **Implement proper error handling** - Handle API errors gracefully  
• 📊 **Use status subresources** - Keep users informed about resource state  
• 🧹 **Implement cleanup logic** - Handle resource deletion properly  
• 📈 **Monitor resource usage** - Ensure your operator doesn't consume excessive resources  
• 🏆 **Use leader election** - For high availability in production  

### 🔒 Security Considerations
• 🔐 **Principle of least privilege** - Grant only necessary permissions  
• 👤 **Use service accounts** - Don't run operators with cluster-admin privileges  
• ✅ **Validate input** - Always validate custom resource specifications  
• 🔒 **Secure communication** - Use TLS for external communications  

---

## 🧹 Cleanup
To clean up the resources created in this lab:

1. **Delete WebApp resources:**
   ```bash
   kubectl delete webapp my-webapp
   kubectl delete -f second-webapp.yaml 2>/dev/null || true
   ```

2. **Delete the operator:**
   ```bash
   kubectl delete -f webapp-operator-deployment.yaml
   ```

3. **Delete RBAC resources:**
   ```bash
   kubectl delete -f webapp-operator-rbac.yaml
   ```

4. **Delete the CRD:**
   ```bash
   kubectl delete -f webapp-crd.yaml
   ```

5. **Delete services:**
   ```bash
   kubectl delete -f my-webapp-service.yaml 2>/dev/null || true
   ```

6. **Clean up any remaining deployments:**
   ```bash
   kubectl delete deployment --selector=managed-by=webapp-operator
   ```

7. **Remove lab files:**
   ```bash
   cd ~
   rm -rf ~/lab13-crd-operator
   ```

---

## 🎉 Conclusion
**Congratulations!** You have successfully completed **Lab 13** on using CRDs and Operators. Here's what you accomplished:

### 🏆 Key Achievements:
• 🏗️ **Created Custom Resource Definitions** - You learned how to extend Kubernetes API with custom resources, defining schemas and validation rules for your WebApp resource type.

• ⚙️ **Deployed a Functional Operator** - You built and deployed an operator that automatically manages the lifecycle of your custom WebApp resources, demonstrating the power of automation in Kubernetes.

• 🧪 **Tested Operator Functionality** - You verified that your operator correctly responds to changes in custom resources, automatically creating and updating deployments based on WebApp specifications.

• 🚀 **Explored Advanced Features** - You examined status reporting, scaling operations, and resource management, gaining insight into production-ready operator patterns.

### 🌟 Why This Matters:
CRDs and Operators are fundamental to extending Kubernetes capabilities and implementing Infrastructure as Code principles. They enable:

• 🎯 **Application-Specific APIs** - Create intuitive interfaces for complex applications  
• 🤖 **Automated Operations** - Reduce manual intervention and human error  
• 🔄 **Consistent Deployments** - Ensure applications are deployed and managed consistently  
• 🧠 **Domain Expertise Encoding** - Capture operational knowledge in code  

### 🌍 Real-World Applications:
• 🗄️ **Database Operators** - Automatically manage database clusters, backups, and scaling  
• 📊 **Monitoring Solutions** - Deploy and configure monitoring stacks with custom resources  
• 🔄 **CI/CD Pipelines** - Create custom resources for build and deployment workflows  
• ☁️ **Multi-Cloud Management** - Abstract cloud-specific resources behind unified APIs  

### 🚀 Next Steps:
• 🔍 Explore existing operators in the [OperatorHub](https://operatorhub.io)  
• 🛠️ Learn about the Operator SDK for building production operators  
• 📚 Study controller-runtime patterns for advanced operator development  
• 🏗️ Practice with Helm Operators and Ansible Operators for different use cases  

This lab provided hands-on experience with one of Kubernetes' most powerful extensibility features, preparing you for the CKAD certification and real-world Kubernetes operations. The concepts you've learned here form the foundation for building sophisticated, automated application management solutions! 🎓✨
