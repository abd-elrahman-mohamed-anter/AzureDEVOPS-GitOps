# Azure DevOps GitOps Voting App

A simple Voting App deployed on **Azure Kubernetes Service (AKS)** with an automated CI/CD and GitOps workflow.

The application is based on the [Example Voting App](https://github.com/dockersamples/example-voting-app). I used the application to build and practice a complete DevOps workflow using Azure DevOps, Docker, ACR, Kubernetes, AKS, and Argo CD.

---

## Project Overview

The project includes:

* Azure DevOps Git repository
* Azure DevOps CI pipeline
* Docker image build
* Azure Container Registry (ACR)
* Kubernetes manifests
* Azure Kubernetes Service (AKS)
* Argo CD for GitOps
* Bash script for updating Kubernetes manifests

The main goal was to automate the process of building the application, pushing the Docker image, updating the Kubernetes manifest, and deploying the new version to AKS.

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
      +-------+-------+
      |       |       |
     Vote   Worker  Result
              |
            Redis
              |
          PostgreSQL
```

### Architecture Screenshot

**[SCREENSHOT 1 — Project Architecture]**

---

## Technologies Used

* Azure DevOps
* Azure Container Registry (ACR)
* Azure Kubernetes Service (AKS)
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

# 1. Azure DevOps Repository

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

The pipeline is configured to run when changes are made inside the `vote/` directory.

```yaml
trigger:
  paths:
    include:
      - vote/*
```

### Screenshot

**[SCREENSHOT 2 — Azure DevOps Repository]**

---

# 2. Build Docker Image

The first stage of the pipeline builds the Docker image for the Vote application.

The image is tagged using the Azure DevOps Build ID:

```text
$(Build.BuildId)
```

For example:

```text
azurecicdcontreg.azurecr.io/voteapp:17
```

This makes it easy to know which pipeline run produced a specific image.

### Screenshot

**[SCREENSHOT 3 — Azure DevOps Build Stage]**

---

# 3. Push Image to ACR

After the image is built, the pipeline pushes it to Azure Container Registry.

```text
Registry:
azurecicdcontreg.azurecr.io

Repository:
voteapp
```

The image tag is kept the same between the pipeline and ACR.

Example:

```text
voteapp:17
```

### Screenshot

**[SCREENSHOT 4 — Azure Container Registry Showing voteapp:17]**

---

# 4. Update Kubernetes Manifest

After pushing the image, the pipeline runs a Bash script:

```text
scripts/updateK8sManifests.sh
```

The script takes three arguments:

```text
$1 = application name
$2 = image repository
$3 = image tag
```

Example:

```text
vote voteapp 17
```

The script updates the image inside the Kubernetes deployment.

Before:

```yaml
image: azurecicdcontreg.azurecr.io/voteapp:16
```

After:

```yaml
image: azurecicdcontreg.azurecr.io/voteapp:17
```

The updated manifest is then committed and pushed to Git.

### Screenshot

**[SCREENSHOT 5 — Pipeline Update Stage]**

**[SCREENSHOT 6 — Updated Kubernetes Manifest in Git]**

---

# 5. Deploy to AKS

The Kubernetes manifests are stored in:

```text
k8s-specifications/
```

The application is deployed to Azure Kubernetes Service.

The application contains several components:

```text
Vote
Result
Worker
Redis
PostgreSQL
```

I used Kubernetes Deployments and Services to run the application and allow the components to communicate with each other.

### Screenshot

**[SCREENSHOT 7 — AKS / kubectl get pods]**

Example:

```bash
kubectl get pods
```

### Screenshot

**[SCREENSHOT 8 — Kubernetes Services]**

Example:

```bash
kubectl get svc
```

---

# 6. Argo CD

Argo CD is used for the GitOps part of the project.

The Kubernetes manifests stored in Git are treated as the desired state.

When the pipeline updates the image tag in Git, Argo CD detects the change and synchronizes it with the AKS cluster.

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

### Screenshot

**[SCREENSHOT 9 — Argo CD Application]**

Show:

```text
Synced
Healthy
```

---

# 7. CI/CD Flow

The complete workflow is:

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

### Screenshot

**[SCREENSHOT 10 — Successful Azure DevOps Pipeline]**

---

# 8. Troubleshooting

While working on the project, I faced some issues during the implementation.

### Bash Script Line Endings

The Bash script had Windows `CRLF` line endings when running on the Linux agent.

I converted the file to Linux `LF` format:

```bash
sed -i 's/\r$//' scripts/updateK8sManifests.sh
```

---

### Git Detached HEAD

The pipeline checkout was running in a detached HEAD state.

Because of that, pushing directly to the branch caused an error.

I used:

```bash
git push origin HEAD:main
```

to push the updated commit to the `main` branch.

---

### Git Authentication

The pipeline needed authentication to push the updated Kubernetes manifest back to the private repository.

I configured Git authentication for the pipeline.

**Important:** credentials and Personal Access Tokens should not be stored directly in the source code or exposed in pipeline logs.

---

### Argo CD Authentication

Argo CD initially could not access the private Azure DevOps repository.

I configured the repository credentials in Argo CD so it could access the manifests and synchronize the application with AKS.

### Screenshot

**[SCREENSHOT 11 — Argo CD Repository Configuration]**

---

# 9. Application

The Voting App contains two main web interfaces:

* **Vote** — submit a vote
* **Result** — view the voting results

### Screenshot

**[SCREENSHOT 12 — Voting Application]**

### Screenshot

**[SCREENSHOT 13 — Result Application]**

---

# What I Learned

Through this project, I practiced:

* Building CI pipelines with Azure DevOps
* Docker image building and tagging
* Working with Azure Container Registry
* Deploying applications to AKS
* Kubernetes Deployments and Services
* GitOps with Argo CD
* Bash scripting
* Git automation
* Troubleshooting CI/CD issues
* Working with private Git repositories

---

# Final Result

The final workflow connects Azure DevOps, ACR, Kubernetes, AKS, and Argo CD:

```text
Azure DevOps
     |
     v
Docker Build
     |
     v
Azure Container Registry
     |
     v
Kubernetes Manifest
     |
     v
Git
     |
     v
Argo CD
     |
     v
AKS
     |
     v
Voting Application
```

### Final Application Screenshot

**[SCREENSHOT 14 — Running Voting Application on AKS]**


## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time
