# Azure DevOps GitOps Voting App

A simple Voting App deployed on **Azure Kubernetes Service (AKS)** with an automated CI/CD and GitOps workflow.

The application is based on the [Example Voting App](https://github.com/dockersamples/example-voting-app).
I used it to build and practice a complete DevOps workflow using Azure DevOps, Docker, Azure Container Registry, Kubernetes, AKS, and Argo CD.

## Architecture Overview For Voting app

![Architecture diagram](architecture.excalidraw.png)


---

## Project Overview

The project includes:

* Azure Kubernetes Service (AKS) cluster
* Azure Container Registry (ACR)
* Azure DevOps Git repository
* Azure DevOps CI pipeline
* Docker image build and versioning
* Kubernetes manifests
* Argo CD for GitOps
* Bash automation
* Redis and PostgreSQL

The main goal was to automate the process from **code change to application deployment**.

Build order followed in this project:

```text
1. Create AKS cluster
2. Create ACR (and connect it to AKS)
3. Set up Azure DevOps CI pipeline (build + push image)
4. Run the pipeline
5. Add the Update stage (update K8s manifest + GitOps sync via Argo CD)
```

---

## Architecture

```text
Developer
    |
    v
Azure DevOps
    |
    v
Azure DevOps Pipeline
    |
    +----> Build Docker Image
    |
    +----> Push Image
              |
              v
      Azure Container Registry
              |
              v
      Update Kubernetes Manifest
              |
              v
         Git Repository
              |
              v
           Argo CD
              |
              v
             AKS
              |
      +-------+-------+-------+-------+
      |       |       |       |       |
     Vote   Worker  Result  Redis  PostgreSQL
```
![Architecture diagram](AZ-Screens/architecture-detailed.png)
---

## Technologies Used

* Azure Kubernetes Service (AKS)
* Azure Container Registry (ACR)
* Azure DevOps
* Kubernetes
* Argo CD
* Docker
* Git
* Bash
* Redis
* PostgreSQL
* Python
* Node.js
* .NET

---

# 1. Create the AKS Cluster

The first step was provisioning the **Azure Kubernetes Service (AKS)** cluster that would run the application.

```text
az aks create \
  --resource-group <resource-group> \
  --name <aks-cluster-name> \
  --node-count 2 \
  --generate-ssh-keys
```

Connected to the cluster with:

```bash
az aks get-credentials --resource-group <resource-group> --name <aks-cluster-name>
```
![Project Architecture](AZ-Screens/cluster.png)

---

# 2. Create Azure Container Registry (ACR)

Next, **Azure Container Registry** was created to store the Docker images built by the pipeline, and attached to the AKS cluster so it can pull images from it.

```bash
az acr create --resource-group <resource-group> --name azurecicdcontreg --sku Basic

az aks update --name <aks-cluster-name> --resource-group <resource-group> --attach-acr azurecicdcontreg
```

![Azure Container Registry](AZ-Screens/az-cr.png)

---

# 3. Azure DevOps Repository & CI Pipeline

The source code is stored in an Azure DevOps Git repository.

Main project structure:

```text
vote-app/
├── vote/
├── result/
├── worker/
├── healthchecks/
├── k8s-specifications/
├── scripts/
└── seed-data/
```

The pipeline is configured to run when changes are made inside the `vote/` directory, and runs on a self-hosted agent pool.

```yaml
trigger:
  paths:
    include:
      - vote/*

resources:
- repo: self

variables:
  dockerRegistryServiceConnection: '<service-connection-id>'
  imageRepository: 'voteapp'
  containerRegistry: 'azurecicdcontreg.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/vote/Dockerfile'
  tag: '$(Build.BuildId)'

pool:
  name: 'myazureagent'

stages:
- stage: Build
  displayName: Build
  jobs:
  - job: Build
    displayName: Build
    steps:
    - task: Docker@2
      displayName: Build image
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'build'
        Dockerfile: 'vote/Dockerfile'
        tags: '$(tag)'

- stage: Push
  displayName: Push
  jobs:
  - job: Push
    displayName: Push
    steps:
    - task: Docker@2
      displayName: push an image to container registry
      inputs:
        containerRegistry: '$(dockerRegistryServiceConnection)'
        repository: '$(imageRepository)'
        command: 'push'
        tags: '$(tag)'
```

![Azure DevOps Repository](AZ-Screens/vote-cipipline.png)

### Build Docker Image

The CI pipeline builds the Docker image for the Vote application and tags it with the Azure DevOps Build ID:

```text
azurecicdcontreg.azurecr.io/voteapp:$(Build.BuildId)
```

Example:

```text
azurecicdcontreg.azurecr.io/voteapp:17
```

### Push Image to ACR

After the build, the pipeline pushes the image to the ACR created in step 2.

```text
Registry:   azurecicdcontreg.azurecr.io
Repository: voteapp
Tag:        17
```

---

# 4. Run the Pipeline Build & Push

With the AKS cluster, ACR, and CI stages in place, the pipeline was triggered for the first time to confirm the build and push stages worked end to end before adding the Update stage.

![Successful Pipeline](AZ-Screens/vote-stages.png)

---

# 5. Update Stage — Update Kubernetes Manifest

After the pipeline was confirmed working, the **Update** stage was added, depending on the `Push` stage:

```yaml
- stage: Update
  displayName: Update
  dependsOn: Push
  condition: succeeded()
  jobs:
  - job: Update
    displayName: Update Kubernetes manifest
    steps:

    - script: |
        sed -i 's/\r$//' scripts/updateK8sManifests.sh
      displayName: Convert CRLF to LF

    - task: Bash@3
      displayName: Update Kubernetes manifest
      inputs:
        targetType: 'filePath'
        filePath: 'scripts/updateK8sManifests.sh'
        arguments: 'vote $(imageRepository) $(tag)'
```

The `scripts/updateK8sManifests.sh` script receives three arguments:

```text
$1 = application name    (vote)
$2 = image repository    (voteapp)
$3 = image tag           (Build.BuildId)
```

```bash
#!/bin/bash
echo "========== SCRIPT STARTED =========="
echo "ARG1=$1"
echo "ARG2=$2"
echo "ARG3=$3"
set -x

# Set the repository URL
REPO_URL="https://<PAT>@dev.azure.com/<org>/vote-app/_git/vote-app"

# Clone the git repository
git clone "$REPO_URL" /tmp/temp_repo
cd /tmp/temp_repo

echo "Before:"
grep "image:" k8s-specifications/$1-deployment.yaml

sed -i "s|image:.*|image: azurecicdcontreg.azurecr.io/$2:$3|g" k8s-specifications/$1-deployment.yaml

echo "After:"
grep "image:" k8s-specifications/$1-deployment.yaml

git add .
git commit -m "Update Kubernetes manifest"
git push origin HEAD:main

rm -rf /tmp/temp_repo
```

> **Note:** the PAT above is redacted. In the actual pipeline it should be pulled from a pipeline secret variable or the built-in `System.AccessToken`, never hardcoded in the script.

The script updates the image in the Kubernetes Deployment.

Before:

```yaml
image: azurecicdcontreg.azurecr.io/voteapp:16
```

After:

```yaml
image: azurecicdcontreg.azurecr.io/voteapp:17
```

The updated manifest is then committed and pushed back to Git with `git push origin HEAD:main`, since the pipeline checkout leaves the repo in a detached HEAD state.
# The Old 
![Pipeline Update](AZ-Screens/theoldmainfast.png)
# The New
![Updated Kubernetes Manifest](AZ-Screens/thenewmainfast.png)

### Image was changed
---

# 6. Deploy to AKS

The Kubernetes manifests are stored in:

```text
k8s-specifications/
```

The application contains:

* Vote
* Result
* Worker
* Redis
* PostgreSQL

Kubernetes Deployments and Services are used to run the application and provide communication between the different components.

![AKS Pods](AZ-Screens/all-svc.png)

---

# 7. GitOps with Argo CD

Argo CD is used for the GitOps part of the project.

The Kubernetes manifests stored in Git are treated as the desired state. When the pipeline updates the image tag in Git, Argo CD detects the change and synchronizes it with the AKS cluster.

```text
New Image
    |
    v
Manifest Updated
    |
    v
Git
    |
    v
Argo CD
    |
    v
AKS
```

The Argo CD application should show:

```text
Synced
Healthy
```
## Argo Application

![Argo CD Application](AZ-Screens/argo-app-interface.png)

## Argo Status
![Argo CD Application](AZ-Screens/argo-interface-resources.png)

---

# 8. Full CI/CD Flow

```text
Code Change
     |
     v
Azure DevOps
     |
     v
Build Docker Image
     |
     v
Push Image to ACR
     |
     v
Update Kubernetes Manifest
     |
     v
Git
     |
     v
Argo CD
     |
     v
AKS
```

---

# 9. Troubleshooting

## Bash Script Line Endings

The Bash script had Windows `CRLF` line endings when running on the Linux agent, causing errors like `set: -\r: invalid option` and `$'\r': command not found`.

Fixed by converting the file to Linux `LF` format:

```bash
sed -i 's/\r$//' scripts/updateK8sManifests.sh
```

To prevent this from recurring, a `.gitattributes` entry was added:

```text
*.sh text eol=lf
```

## Git Detached HEAD

The Azure DevOps pipeline checkout was running in a detached HEAD state, so pushing directly to the branch caused an error. Fixed with:

```bash
git push origin HEAD:main
```

## Git Authentication

The pipeline needed authentication to push the updated Kubernetes manifest back to the private Azure DevOps repository.

> **Note:** Personal Access Tokens and other credentials should never be committed to Git or exposed in pipeline logs.

## Argo CD Repository Authentication

Argo CD initially could not access the private Azure DevOps repository. Repository credentials were configured in Argo CD so it could read the Kubernetes manifests and synchronize the application with AKS.

---

# 10. Application

The Voting App contains two main web interfaces:

* **Vote** — submit a vote
* **Result** — view the voting results

![Voting Application](AZ-Screens/port-fr.png)

![Result Application](AZ-Screens/app.png)

---

# What I Learned

Through this project, I practiced:

* Provisioning an AKS cluster and attaching an ACR to it
* Building CI pipelines with Azure DevOps
* Docker image building and tagging
* Deploying applications to AKS
* Kubernetes Deployments and Services
* GitOps with Argo CD
* Bash scripting and automation
* Git workflows and private repository authentication
* Troubleshooting CI/CD issues (line endings, detached HEAD, auth)

---

# Final Result

```text
AKS Cluster
     |
     v
Azure Container Registry
     |
     v
Azure DevOps CI Pipeline
     |
     v
Update Stage (K8s Manifest)
     |
     v
Git
     |
     v
Argo CD
     |
     v
Voting Application Running on AKS
```
---

## Application Components

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time
