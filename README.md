# Voting App CI/CD Pipeline using Azure DevOps and Argo CD

This project demonstrates the implementation of a complete CI/CD pipeline for a Docker-based multi-microservice voting application deployed on Kubernetes. The CI pipeline is built using **Azure DevOps** and the CD is managed by **Argo CD**.

## Project Overview

The voting application consists of three microservices:
- **Vote**: A front-end service where users can cast votes.
- **Result**: A back-end service that displays the results of the votes.
- **Worker**: A background service that processes and stores votes.

The overall CI/CD pipeline workflow involves:
- Building and pushing Docker images for each microservice.
- Deploying microservices to a Kubernetes cluster using Argo CD.
- Monitoring updates and automating deployments via Argo CD.

## Architecture Diagram
![alt text](https://github.com/dockersamples/example-voting-app/blob/main/architecture.excalidraw.png)

## Pipeline Overview

### 1. **Azure DevOps Pipeline (CI)**

Each microservice has its own Azure DevOps pipeline to build and push the Docker images to a container registry (like Azure Container Registry or DockerHub). The process involves:

1. **Build Stage**:
   - Each microservice (`vote`, `result`, `worker`) has a separate pipeline definition in Azure DevOps.
   - Docker images are built from the source code available in the GitHub repository.
2. **Create self-hosted Linux agent**:
   - Create a self-hosted virtual machine to run pipelines.
     
3. **Push Stage**:
   - The pipeline pushes the built Docker images to a container registry.

### 2. **Azure DevOps Pipeline (Update)**

After the images are built and pushed, an additional **update stage** modifies the Kubernetes deployment manifests to point to the new image tags. This update occurs in a specific folder in the GitHub repository that Argo CD monitors.

- The pipeline will:
  - Fetch the latest image tag.
  - Update the Kubernetes manifest in the **k8s-specifications** folder to reflect the new image version.
  - Commit and push the updated manifest to the **k8s-specifications** folder.

### 3. **Argo CD for Continuous Deployment**

Argo CD continuously monitors the Kubernetes manifest folder for changes. When the Azure DevOps pipeline updates the image path in the Kubernetes manifest, Argo CD detects the change and automatically triggers the deployment.

- **Kubernetes Cluster**: The voting app is deployed on a Kubernetes cluster.
- **Argo CD**: Monitors the **k8s-specifications** folder and automatically applies changes to the cluster when updates are detected.

## Steps

### Step 1: Set Up the Repository

1. Clone the Docker voting application GitHub repository.
    ```bash
    git clone https://github.com/dockersamples/example-voting-app.git
    ```

2. The repository should have the following structure:
    ```
    ├── vote/
    ├── result/
    ├── worker/
    ├── k8s-specifications/
    │   ├── vote-deployment.yaml
    │   ├── result-deployment.yaml
    │   └── worker-deployment.yaml
    ```

### Step 2: Set Up Azure DevOps Pipeline (CI)

1. **Create Pipelines**: Create a separate pipeline for each microservice (`vote`, `result`, `worker`) in Azure DevOps.
   
2. **Pipeline YAML File**:
   The pipeline YAML file for each microservice might look like this:

```yaml
trigger:
  paths:
    include:
      - vote/*    # the repo we are using

resources:
  - repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '37868c72-32ef-488d-a490-1415f4b73792'
  imageRepository: 'votingapp'
  containerRegistry: 'ujjwalcicdapp.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/vote/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  name: 'azureagent'    # the name of my newly created self-hosted agent

stages:                               
  - stage: Build                        
    displayName: Build the Voting App
    jobs:
    - job: Build
      displayName: Build
      steps:
      - task: Docker@2
        inputs:
          containerRegistry: 'votingappcicd'
          repository: 'votingapp/vote'          
          command: 'build'
          Dockerfile: 'vote/Dockerfile'

  - stage: Push                           
    displayName: Push the Voting App        
    jobs:
    - job: Push
      displayName: Pushing the voting App
      steps:
      - task: Docker@2
        inputs:
          containerRegistry: 'votingappcicd'
          repository: 'votingapp/vote'
          command: 'push'
```
3. **Push Docker Images**: Ensure each pipeline pushes the Docker images to the container registry with the latest tags.
   
### Step 3: Update Image in Kubernetes Manifests
 1. **Update Stage**: The pipeline includes an additional update stage that modifies the image tags in the Kubernetes manifests.
 2. The script updates the image: path in the k8s-specifications/*.yaml files with the new image tag.
```yaml
    - stage: Update_bash_script
  displayName: Update Bash Script
  jobs:
  - job: Updating_repo_with_bash
    displayName: Updating repo using bash script
    steps:
    - script: |
        # Convert line endings to Unix format
        dos2unix scripts/updateK8sManifests.sh

        # Run the shell script with the appropriate arguments
        bash scripts/updateK8sManifests.sh "worker" "$(imageRepository)" "$(tag)"
      displayName: Run UpdateK8sManifests Script
```
3. Update a bash script inside the script directory.Bash script looks like:
```bash
#!/bin/bash

set -x

# Set the repository URL
REPO_URL="https://<ACCESS-TOKEN>@dev.azure.com/<AZURE-DEVOPS-ORG-NAME>/voting-app/_git/voting-app"

# Clone the git repository into the /tmp directory
git clone "$REPO_URL" /tmp/temp_repo

# Navigate into the cloned repository directory
cd /tmp/temp_repo

# Make changes to the Kubernetes manifest file(s)
# For example, let's say you want to change the image tag in a deployment.yaml file
sed -i "s|image:.*|image: <ACR-REGISTRY-NAME>/$2:$3|g" k8s-specifications/$1-deployment.yaml

# Add the modified files
git add .

# Commit the changes
git commit -m "Update Kubernetes manifest"

# Push the changes back to the repository
git push

# Cleanup: remove the temporary directory
rm -rf /tmp/temp_repo
```
### Step 4: Set Up Argo CD for Continuous Deployment
1. **Install Argo CD**: Install Argo CD in the Kubernetes cluster.
2. **Create Argo CD**: Application: Create an Argo CD application that points to the GitHub repository and the k8s-specifications/ folder.
3. **Sync Argo CD**: Argo CD will monitor the folder for changes and automatically deploy the updated images when new versions are pushed.

### Conclusion
This setup provides a fully automated CI/CD pipeline for a multi-microservice Docker application using Azure DevOps for continuous integration and Argo CD for continuous deployment.

### Project Demonstration

[![Watch the video](https://img.youtube.com/vi/dPZ3eMxnsrY/0.jpg)](https://youtu.be/H304DtMPrwM?si=xWb2_cVJSEBghK_C)


### References
- [Azure DevOps Documentation](https://docs.microsoft.com/en-us/azure/devops/)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Voting App Docker](https://github.com/dockersamples/example-voting-app.git)
