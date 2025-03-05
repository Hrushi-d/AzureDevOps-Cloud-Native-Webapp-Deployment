# Deploying a Cloud Native Web Application Using Azure DevOps

## Introduction

This guide provides a comprehensive walkthrough for deploying a web application using Azure Free Tier. The project leverages cloud infrastructure, containerization, and CI/CD automation to create a production-ready deployment pipeline. Throughout this guide, you'll gain hands-on experience with:

- Setting up Azure infrastructure using Terraform and Azure CLI
- Containerizing applications with Docker
- Orchestrating containers with Azure Kubernetes Service (AKS)
- Implementing CI/CD pipelines with Azure DevOps
- Monitoring Kubernetes workloads in Azure

By following this guide, you'll create a fully automated deployment pipeline for a web application, gaining practical experience with industry-standard DevOps practices.

---

## 1. Provisioning Infrastructure

### Using Terraform

Infrastructure as Code (IaC) allows you to define and provision your cloud resources in a consistent, repeatable manner. We'll use Terraform to create the following resources:

```hcl
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "webapp-rg"
  location = "East US"
}

resource "azurerm_virtual_network" "vnet" {
  name                = "webapp-vnet"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "webapp-aks"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "webappaks"
  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_DS2_v2"
  }
  identity {
    type = "SystemAssigned"
  }
}
```

### Using Azure CLI

Alternatively, you can provision infrastructure using Azure CLI. This involves:

1. Creating a resource group
2. Setting up a virtual network
3. Configuring network security groups with proper inbound/outbound rules:
   - PORT 80 for HTTP traffic
   - PORT 22 for SSH access
4. Creating public IPs and network interfaces for VMs
5. Provisioning two virtual machines:
   - **Dev VM** for application development and testing
   - **Ops VM** for managing deployments and dependencies
6. Creating an AKS cluster for production deployment

### Implementation Steps

1. **Using Terraform**:
   - Install Terraform and configure Azure credentials
   - Define infrastructure in Terraform configuration files
   - Initialize with `terraform init`
   - Apply the configuration using `terraform apply`
   - Verify resources in Azure Portal

2. **Using Azure CLI**:
   - Log in to Azure Portal and open Cloud CLI
   - Create resource group: `az group create --name webapp-rg --location eastus`
   - Create virtual network and subnets
   - Create network security groups with appropriate rules
   - Provision VMs and necessary networking components
   - Create AKS cluster: `az aks create --resource-group webapp-rg --name webapp-aks --node-count 2 --enable-addons monitoring --generate-ssh-keys`

---

## 2. Configuration Management

Once the infrastructure is provisioned, we need to configure our VMs with the necessary tools and services.

### Using Azure VM Run Command

We'll use Azure CLI's `az vm run-command` to automate VM configurations without direct login:

```bash
az vm run-command invoke \
  --resource-group webapp-rg \
  --name dev-vm \
  --command-id RunShellScript \
  --scripts "apt-get update && apt-get install -y docker.io && systemctl enable docker && systemctl start docker"
```

### Configuration Steps

1. Update package lists: `apt-get update`
2. Install Docker: `apt-get install -y docker.io`
3. Enable and start Docker service: `systemctl enable docker && systemctl start docker`
4. Verify configurations on both VMs:
   ```bash
   docker --version
   systemctl status docker
   ```

Configure both the Dev VM and Ops VM following the same process to ensure consistency.

---

## 3. Deploying the Web Application

### Manual Deployment Process

Before automating, it's valuable to understand the manual deployment process:

1. **Development (Dev VM)**:
   - Clone the GitHub repository containing the web application
   - Create a Dockerfile to containerize the application:
     ```dockerfile
     FROM nginx:latest
     COPY index.html /usr/share/nginx/html/index.html
     COPY assets /usr/share/nginx/html/assets
     COPY forms /usr/share/nginx/html/forms
     COPY portfolio-details.html /usr/share/nginx/html/portfolio-details.html
     EXPOSE 80
     CMD ["nginx", "-g", "daemon off;"]
     ```
   - Build the Docker image: `docker build -t webapp:latest .`
   - Test locally: `docker run -d -p 80:80 webapp:latest`
   - Tag the image for Docker Hub: `docker tag webapp:latest username/webapp:latest`
   - Push to Docker Hub: `docker push username/webapp:latest`

2. **Deployment to AKS**:
   - Connect to the AKS cluster: `az aks get-credentials --resource-group webapp-rg --name webapp-aks`
   - Create a Kubernetes deployment:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: auto-nginx-portfolio
       labels:
         app: auto-nginx-portfolio
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: auto-nginx-portfolio
       template:
         metadata:
           labels:
             app: auto-nginx-portfolio
         spec:
           containers:
           - name: auto-nginx-portfolio
             image: hrush18/auto-nginx-portfolio:latest
             ports:
             - containerPort: 80
     ```
   - Apply the deployment: `kubectl apply -f deployment.yaml`
   - Expose the deployment as a service:
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: auto-nginx-portfolio-service
       labels:
         app: auto-nginx-portfolio
     spec:
       type: LoadBalancer
       selector:
         app: auto-nginx-portfolio
       ports:
         - protocol: TCP
           port: 80
           targetPort: 80
     ```
   - Apply the service: `kubectl apply -f service.yaml`
   - Verify the deployment: `kubectl get pods,svc`

---

## 4. Automating Deployment with Azure DevOps

Azure DevOps provides powerful tools for implementing CI/CD pipelines. We'll set up continuous integration and continuous deployment to automate the entire process from code commit to production deployment.

### 4.1 Setting Up Azure DevOps Project

1. **Create an Azure DevOps Organization (if you don't have one)**:
   - Go to [dev.azure.com](https://dev.azure.com)
   - Sign in with your Microsoft account
   - Create a new organization or use an existing one

2. **Create a New Project**:
   - Click on "New project"
   - Enter a project name (e.g., "WebApp-Deployment")
   - Select visibility (private or public)
   - Click "Create"

3. **Set Up Repositories**:
   - Navigate to Repos in your project
   - Create two repositories:
     - **Project Repository**: `portfolio` (for application code)
     - **Admin Repository**: `admin_portfolio` (for deployment configurations)
   - Initialize both repositories with a README file

4. **Clone Repositories**:
   - Get the clone URL for each repository from Azure DevOps
   - Clone to the respective VMs:
     ```bash
     # On Dev VM
     git clone https://dev.azure.com/organization/project/_git/portfolio
     
     # On Ops VM
     git clone https://dev.azure.com/organization/project/_git/admin_portfolio
     ```

### 4.2 Creating Service Connections

Service connections allow Azure DevOps to interact with external services like Docker Hub and Azure.

1. **Create Docker Hub Service Connection**:
   - Go to Project Settings > Service connections
   - Click "New service connection"
   - Select "Docker Registry"
   - Enter your Docker Hub credentials
   - Name the connection (e.g., "DockerHubConnection")
   - Click "Save"

2. **Create Azure Resource Manager Service Connection**:
   - Go to Project Settings > Service connections
   - Click "New service connection"
   - Select "Azure Resource Manager"
   - Follow the authentication steps
   - Select your subscription and resource group
   - Name the connection (e.g., "AzureConnection")
   - Click "Save"

### 4.3 Setting Up the CI Pipeline

The CI pipeline will automate the build and push of Docker images when code changes are detected.

1. **Create a New Pipeline**:
   - Go to Pipelines > Pipelines
   - Click "Create Pipeline"
   - Select "Azure Repos Git" as the source
   - Select the `portfolio` repository

2. **Configure the Pipeline**:
   - Select "Starter pipeline" or "Empty job"
   - Replace the YAML content with the following:

```yaml
trigger:
- main  # Trigger on changes to main branch

pool:
  vmImage: 'ubuntu-latest'  # Use Ubuntu-based agent

variables:
  dockerHubUser: 'yourDockerHubUsername'
  imageName: 'auto-nginx-portfolio'
  imageTag: '$(Build.BuildId)'  # Use build ID as tag

stages:
- stage: Build
  displayName: 'Build and Push Docker Image'
  jobs:
  - job: BuildAndPush
    steps:
    - checkout: self  # Check out the repository

    - task: Docker@2
      displayName: 'Build Docker Image'
      inputs:
        command: build
        repository: $(dockerHubUser)/$(imageName)
        dockerfile: '$(Build.SourcesDirectory)/Dockerfile'
        tags: |
          $(imageTag)
          latest

    - task: Docker@2
      displayName: 'Push Docker Image'
      inputs:
        command: push
        containerRegistry: 'DockerHubConnection'  # Use the service connection
        repository: $(dockerHubUser)/$(imageName)
        tags: |
          $(imageTag)
          latest

    # Trigger the image tag update script
    - task: Bash@3
      displayName: 'Update Deployment Configuration'
      inputs:
        targetType: 'inline'
        script: |
          # Clone admin repository
          git clone https://$(System.AccessToken)@dev.azure.com/organization/project/_git/admin_portfolio /tmp/admin_portfolio
          cd /tmp/admin_portfolio
          
          # Update image tag in deployment.yaml
          sed -i "s|image: .*|image: $(dockerHubUser)/$(imageName):$(imageTag)|" deployment.yaml
          
          # Configure Git
          git config --global user.name "Azure DevOps Pipeline"
          git config --global user.email "pipeline@example.com"
          
          # Commit and push changes
          git add deployment.yaml
          git commit -m "Update image tag to $(imageTag)"
          git push
```

3. **Save and Run the Pipeline**:
   - Click "Save and run"
   - Commit the YAML file to your repository
   - Watch the pipeline execution

### 4.4 Setting Up the CD Pipeline

The CD pipeline will deploy the application to AKS when the deployment configuration changes.

1. **Create a New Pipeline**:
   - Go to Pipelines > Pipelines
   - Click "Create Pipeline"
   - Select "Azure Repos Git" as the source
   - Select the `admin_portfolio` repository

2. **Configure the Pipeline**:
   - Select "Starter pipeline" or "Empty job"
   - Replace the YAML content with the following:

```yaml
trigger:
- main  # Trigger on changes to main branch

pool:
  vmImage: 'ubuntu-latest'  # Use Ubuntu-based agent

stages:
- stage: Deploy
  displayName: 'Deploy to AKS'
  jobs:
  - job: DeployToAKS
    steps:
    - checkout: self  # Check out the repository

    # Install kubectl
    - task: KubectlInstaller@0
      displayName: 'Install kubectl'
      inputs:
        kubectlVersion: 'latest'

    # Set Kubernetes context
    - task: AzureCLI@2
      displayName: 'Set Kubernetes Context'
      inputs:
        azureSubscription: 'AzureConnection'  # Use the service connection
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          az aks get-credentials --resource-group webapp-rg --name webapp-aks --overwrite-existing

    # Apply Kubernetes manifests
    - task: Kubernetes@1
      displayName: 'Apply Deployment'
      inputs:
        connectionType: 'None'
        command: 'apply'
        useConfigurationFile: true
        configuration: '$(Build.SourcesDirectory)/deployment.yaml'

    - task: Kubernetes@1
      displayName: 'Apply Service'
      inputs:
        connectionType: 'None'
        command: 'apply'
        useConfigurationFile: true
        configuration: '$(Build.SourcesDirectory)/service.yaml'

    # Verify deployment
    - task: Bash@3
      displayName: 'Verify Deployment'
      inputs:
        targetType: 'inline'
        script: |
          # Wait for deployment to roll out
          kubectl rollout status deployment/auto-nginx-portfolio
          
          # Get service info
          echo "Service details:"
          kubectl get service auto-nginx-portfolio-service -o wide
          
          # Get deployment info
          echo "Deployment details:"
          kubectl get deployment auto-nginx-portfolio -o wide
```

3. **Save and Run the Pipeline**:
   - Click "Save and run"
   - Commit the YAML file to your repository
   - Watch the pipeline execution

### 4.5 Setting Up Webhook Triggers

To fully automate the process, set up webhook triggers between repositories:

1. **Configure CI Pipeline Trigger**:
   - Ensure the CI pipeline triggers on changes to the main branch of the `portfolio` repository

2. **Configure CD Pipeline Trigger**:
   - Ensure the CD pipeline triggers on changes to the main branch of the `admin_portfolio` repository

### 4.6 Custom Image Tagging Script

For more advanced scenarios, a dedicated script for image tagging:

```bash
#!/bin/bash
# File: docker_image_tag.sh

# Ensure a BUILD_ID is provided
if [ -z "$1" ]; then
  echo "Usage: $0 <BUILD_ID>"
  exit 1
fi
BUILD_ID=$1

# Define repository variables
ADMIN_PORTFOLIO_REPO="https://username:token@dev.azure.com/username/project/_git/admin_portfolio"
DEPLOYMENT_FILE="deployment.yaml"

# Clone the admin_portfolio repository
git clone $ADMIN_PORTFOLIO_REPO /tmp/admin_portfolio
cd /tmp/admin_portfolio

# Pull the latest changes
git pull origin main

# Update the image tag in the deployment.yaml file
sed -i "s|image: .*|image: hrush18/auto-nginx-portfolio:${BUILD_ID}|" ${DEPLOYMENT_FILE}

# Configure Git user
git config --global user.name "username"
git config --global user.email "email@example.com"

# Commit and push changes
git add ${DEPLOYMENT_FILE}
git commit -m "Update image tag to ${BUILD_ID}"
git pull origin main --rebase
git push origin main

# Clean up
cd -
rm -rf /tmp/admin_portfolio
```

### 4.7 Setting Up Pipeline Variables and Secrets

Secure sensitive information using variables and secrets:

1. **Create Variables**:
   - Go to Pipelines > Library
   - Create a variable group (e.g., "DeploymentVariables")
   - Add variables:
     - `dockerHubUser`: Your Docker Hub username
     - `imageName`: The name of your Docker image
     - `resourceGroup`: Your Azure resource group name
     - `aksClusterName`: Your AKS cluster name

2. **Add Secrets**:
   - Add sensitive variables as secrets:
     - `dockerHubPassword`: Your Docker Hub password (mark as secret)
     - `gitPAT`: Personal Access Token for Git operations (mark as secret)

3. **Link Variables to Pipelines**:
   - In each pipeline YAML, add:
   ```yaml
   variables:
     - group: DeploymentVariables
   ```

### 4.8 Setting Up Environment Approvals and Gates

For production deployments, set up approval workflows:

1. **Create an Environment**:
   - Go to Pipelines > Environments
   - Create a new environment (e.g., "Production")
   - Configure approval checks:
     - Add approvers who must approve deployments
     - Set timeout period for approvals

2. **Update CD Pipeline to Use Environment**:
   - Modify the CD pipeline YAML:
   ```yaml
   stages:
   - stage: Deploy
     displayName: 'Deploy to AKS'
     jobs:
     - deployment: DeployToAKS
       environment: Production  # Use the environment with approvals
       strategy:
         runOnce:
           deploy:
             steps:
             # Deployment steps...
   ```

### 4.9 Setting Up Pipeline Notifications

Configure notifications to keep the team informed:

1. **Configure Notifications**:
   - Go to Project Settings > Notifications
   - Add subscription for pipeline events:
     - Pipeline run completed
     - Pipeline run failed
     - Deployment approval pending

2. **Set Up Microsoft Teams/Slack Integration**:
   - Add a webhook to your Teams/Slack channel
   - Configure Azure DevOps to send notifications to the webhook

### 4.10 Monitoring CI/CD Performance

Track pipeline performance metrics:

1. **View Pipeline Analytics**:
   - Go to Pipelines > Analytics
   - Monitor:
     - Pipeline duration
     - Success rate
     - Failure points
     - Wait time for approvals

2. **Set Up Dashboards**:
   - Create custom dashboards with pipeline widgets
   - Add metrics for deployment frequency and success rates

---

## 5. Monitoring Kubernetes with Azure Portal

Azure provides monitoring capabilities for Kubernetes workloads, even in the Free Tier.

### Basic Monitoring Features

1. **Resource Usage Metrics**:
   - CPU and memory utilization
   - Node status and health
   - Pod status and health

2. **Log Analytics**:
   - Container logs
   - Error monitoring
   - Performance insights

3. **Azure Dashboard**:
   - Comprehensive overview of your Azure DevOps workflows
   - Pipeline execution status
   - Deployment status and history

### Accessing Monitoring

1. Navigate to your AKS cluster in the Azure Portal
2. Select "Insights" or "Metrics" from the menu
3. View real-time and historical data on cluster performance

---

## Conclusion

This project demonstrates a complete DevOps workflow for deploying web applications using Azure services. By following this guide, you've learned how to:

- Provision infrastructure using Terraform or Azure CLI
- Configure VMs with necessary tools and services
- Containerize a web application using Docker
- Deploy applications to Azure Kubernetes Service
- Automate the entire deployment process using Azure DevOps CI/CD pipelines
- Monitor Kubernetes workloads using Azure Portal

These skills provide a solid foundation for implementing DevOps practices in cloud environments, enabling more efficient and reliable software delivery.performance

---

## Contact Us 📧

Have questions, feedback, or need assistance? Reach out to:
- Email: [hrushikeshdagwar@gmail.com](mailto:hrushikeshdagwar@gmail.com)

