تفضل، هذا هو ملف `README.md` كاملاً في **بلوك واحد فقط** (داخل علامات `` ```markdown ``). يمكنك نسخه بالكامل ولصقه مباشرة في ملف `README.md` على GitHub.

```markdown
# Azure DevOps GitOps Voting App

A simple Voting App deployed on **Azure Kubernetes Service (AKS)** with an automated CI/CD and GitOps workflow.

The application is based on the [Example Voting App](https://github.com/dockersamples/example-voting-app). I used the application to build and practice a complete DevOps workflow using Azure DevOps, Docker, Azure Container Registry, Kubernetes, AKS, and Argo CD.

---

## Project Overview

The project includes:

* Azure DevOps Git repository
* Azure DevOps CI pipeline
* Docker image build and versioning
* Azure Container Registry (ACR)
* Kubernetes manifests
* Azure Kubernetes Service (AKS)
* Argo CD for GitOps
* Bash automation
* Redis and PostgreSQL

The main goal was to automate the process from **code change to application deployment**.

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

![Project Architecture](AZ-Screens/01-architecture.png)
*(احفظ صورة Argo CD Application Details Tree أو أي صورة تعبر عن الهيكل باسم `01-architecture.png`)*

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

![Azure DevOps Repository](AZ-Screens/02-azure-devops-repository.png)
*(احفظ صورة Azure DevOps Pipelines الرئيسية باسم `02-azure-devops-repository.png`)*

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

This makes it easy to identify which pipeline run produced a specific image.

![Azure DevOps Build](AZ-Screens/03-pipeline-build.png)
*(احفظ صورة الـ Pipeline التي تظهر الـ Jobs والـ Stages باسم `03-pipeline-build.png`)*

---

# 3. Push Image to Azure Container Registry

After the image is built, the pipeline pushes it to **Azure Container Registry**.

```text
Registry:
azurecicdcontreg.azurecr.io

Repository:
voteapp
```

The same image tag is used in ACR.

Example:

```text
voteapp:17
```

![Azure Container Registry](AZ-Screens/04-acr.png)
*(احفظ صورة Azure Container Registry التي تظهر الـ Tags باسم `04-acr.png`)*

---

# 4. Update Kubernetes Manifest

After pushing the image, the pipeline runs the Bash script:

```text
scripts/updateK8sManifests.sh
```

The script receives three arguments:

```text
$1 = application name
$2 = image repository
$3 = image tag
```

Example:

```text
vote voteapp 17
```

The script updates the image in the Kubernetes Deployment.

Before:

```yaml
image: azurecicdcontreg.azurecr.io/voteapp:16
```

After:

```yaml
image: azurecicdcontreg.azurecr.io/voteapp:17
```

The updated manifest is then committed and pushed back to Git.

![Pipeline Update](AZ-Screens/05-pipeline-update.png)
*(احفظ صورة الـ Terminal التي تظهر `Update Kubernetes manifest` باسم `05-pipeline-update.png`)*

![Updated Kubernetes Manifest](AZ-Screens/06-updated-manifest.png)
*(احفظ صورة كود الـ YAML التي تظهر `image: azurecicdcontreg.azurecr.io/voteapp:17` باسم `06-updated-manifest.png`)*

---

# 5. Deploy to AKS

The Kubernetes manifests are stored in:

```text
k8s-specifications/
```

The application is deployed to **Azure Kubernetes Service (AKS)**.

The application contains:

* Vote
* Result
* Worker
* Redis
* PostgreSQL

Kubernetes Deployments and Services are used to run the application and provide communication between the different components.

![AKS Pods](AZ-Screens/07-aks-pods.png)
*(احفظ صورة الـ Terminal التي تظهر `kubectl get pods` باسم `07-aks-pods.png`)*

![Kubernetes Services](AZ-Screens/08-kubernetes-services.png)
*(احفظ صورة الـ Terminal التي تظهر `kubectl get svc` باسم `08-kubernetes-services.png`)*

---

# 6. GitOps with Argo CD

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

The Argo CD application should show:

```text
Synced
Healthy
```

![Argo CD Application](AZ-Screens/09-argocd.png)
*(احفظ صورة Argo CD Applications الرئيسية التي تظهر `Synced` و `Healthy` باسم `09-argocd.png`)*

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

![Successful Pipeline](AZ-Screens/10-pipeline-success.png)
*(احفظ صورة Azure DevOps Pipeline الناجحة باسم `10-pipeline-success.png`)*

---

# 8. Troubleshooting

During the implementation, I faced and resolved several issues.

## Bash Script Line Endings

The Bash script had Windows `CRLF` line endings when running on the Linux agent.

I converted the file to Linux `LF` format:

```bash
sed -i 's/\r$//' scripts/updateK8sManifests.sh
```

---

## Git Detached HEAD

The Azure DevOps pipeline checkout was running in a detached HEAD state.

Because of this, pushing directly to the branch caused an error.

I used:

```bash
git push origin HEAD:main
```

to push the updated commit to the `main` branch.

---

## Git Authentication

The pipeline needed authentication to push the updated Kubernetes manifest back to the private Azure DevOps repository.

Git authentication was configured for the pipeline.

> **Note:** Personal Access Tokens and other credentials should never be committed to Git or exposed in pipeline logs.

---

## Argo CD Repository Authentication

Argo CD initially could not access the private Azure DevOps repository.

I configured the repository credentials in Argo CD so it could read the Kubernetes manifests and synchronize the application with AKS.

![Argo CD Repository Configuration](AZ-Screens/11-argocd-repository.png)
*(احفظ صورة إعدادات Argo CD أو صورة الـ Login باسم `11-argocd-repository.png`)*

---

# 9. Application

The Voting App contains two main web interfaces:

* **Vote** — submit a vote
* **Result** — view the voting results

![Voting Application](AZ-Screens/12-voting-app.png)
*(احفظ صورة واجهة التصويت باسم `12-voting-app.png`)*

![Result Application](AZ-Screens/13-result-app.png)
*(احفظ صورة واجهة النتائج باسم `13-result-app.png`)*

---

# What I Learned

Through this project, I practiced:

* Building CI pipelines with Azure DevOps
* Docker image building and tagging
* Working with Azure Container Registry
* Deploying applications to AKS
* Kubernetes Deployments and Services
* GitOps with Argo CD
* Bash scripting and automation
* Git workflows
* Private Git repository authentication
* Troubleshooting CI/CD issues

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

![Running Application on AKS](AZ-Screens/14-final-application.png)
*(احفظ صورة Argo CD التي تظهر الـ Pods وهي تعمل `Running` باسم `14-final-application.png`)*
```


## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time
