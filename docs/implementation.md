# Implementation

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7508553a-294f-4285-80b9-6560e57a9e01" />

## 1. Implementation Overview

This document describes how the VProfile GitOps platform was assembled and connected.

The implementation was performed in the following major sequence:

```text
1. Create Git repositories
        ↓
2. Configure Git SSH authentication
        ↓
3. Prepare Kubernetes deployment manifests
        ↓
4. Convert manifests into a Helm chart
        ↓
5. Configure application CI/CD
        ↓
6. Configure ECR and pipeline IAM access
        ↓
7. Configure GitHub Secrets and Variables
        ↓
8. Provision AWS/EKS infrastructure with Terraform
        ↓
9. Configure EKS access
        ↓
10. Install cert-manager
        ↓
11. Install AWS Load Balancer Controller
        ↓
12. Install Argo CD
        ↓
13. Configure HTTPS / ALB access
        ↓
14. Register Helm repository with Argo CD
        ↓
15. Create Argo CD Project and Application
        ↓
16. Validate end-to-end GitOps deployment
```

The implementation connects three lifecycle domains:

```text
Application
    ↓
GitHub Actions
    ↓
Amazon ECR

Deployment
    ↓
Helm
    ↓
Argo CD
    ↓
Amazon EKS

Infrastructure
    ↓
Terraform
    ↓
AWS / EKS
```

---

## 2. Repository Preparation

The project uses three Git repositories:

```text
vprofile-app
vprofile-helm
vprofile-infra
```

### Repository responsibilities

| Repository | Purpose |
|---|---|
| `vprofile-app` | Application source and application CI/CD |
| `vprofile-helm` | Helm charts and Argo CD deployment configuration |
| `vprofile-infra` | Terraform infrastructure configuration |

The separation allows the three concerns to evolve independently.

An application change does not require an infrastructure change.

A Helm deployment change does not require rebuilding the application.

An infrastructure change does not require rebuilding the application image.

---

## 3. GitHub SSH Authentication

SSH authentication was configured to allow the local environment to work with the private GitHub repositories.

The implementation uses:

```text
Local Machine
      │
      ▼
SSH Private Key
      │
      ▼
SSH Config Host Alias
      │
      ▼
GitHub
```

A host-specific SSH configuration can be used when multiple GitHub accounts or keys are involved.

Example:

```text
Host github.com-devops4sure
  HostName github.com
  User git
  IdentityFile ~/.ssh/devops4sure
  IdentitiesOnly yes
```

The private key remains on the local machine.

The corresponding public key is registered with GitHub.

The SSH configuration allows Git to select the intended key when communicating with GitHub.

---

## 4. Local Repository Workspace

The three repositories were cloned into a common local workspace.

Example:

```bash
mkdir -p ~/Desktop/gitops
cd ~/Desktop/gitops

git clone <vprofile-helm-ssh-url>
git clone <vprofile-infra-ssh-url>
git clone <vprofile-app-ssh-url>
```

The resulting workspace is:

```text
gitops/
├── vprofile-app/
├── vprofile-helm/
└── vprofile-infra/
```

This local structure makes the relationships between the three repositories explicit during implementation.

---

## 5. Kubernetes Manifest Preparation

The existing VProfile Kubernetes definitions were used as the starting point for the Helm implementation.

The supplied Kubernetes definitions contained resources such as:

```text
Deployment
Service
PersistentVolumeClaim
Secret
Ingress
```

These manifests were placed into the Helm repository as the source material for the chart conversion.

The original application workload was not rewritten as part of this project.

The implementation work was to transform the deployment configuration into a reusable Helm structure.

---

## 6. Helm Chart Implementation

### Objective

The original Kubernetes manifests contained hardcoded values.

The Helm implementation parameterizes those values so the deployment can be controlled through `values.yaml`.

Conceptually:

```text
Static Kubernetes Manifest
        ↓
Hardcoded image / replicas / storage / ingress
        ↓
Helm Template
        ↓
{{ .Values.xxx }}
        ↓
values.yaml
```

This makes the deployment configuration suitable for CI/CD automation.

---

## 7. Helm Chart Structure

The resulting chart follows the project structure:

```text
vprofile-helm/
├── helm/
│   └── vprofile/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── app-deployment.yaml
│           ├── db-deployment.yaml
│           ├── mc-deployment.yaml
│           ├── rmq-deployment.yaml
│           ├── services.yaml
│           ├── ingress.yaml
│           ├── secret.yaml
│           ├── pvc.yaml
│           └── dockerregistry-secret.yaml
```

The chart uses separate template files for logical resource groups rather than placing all Kubernetes resources into one large template.

---

## 8. `values.yaml` as the CI/CD Interface

The most important implementation decision in the Helm chart is the use of `values.yaml` as the interface between CI and deployment configuration.

Conceptually:

```text
GitHub Actions
      │
      │ update image tag
      ▼
values.yaml
      │
      ▼
Helm Templates
      │
      ▼
Kubernetes Resources
```

The application pipeline does not need to modify individual Kubernetes templates.

Instead, it updates the image configuration in `values.yaml`.

The important application values include:

```yaml
app:
  image: <ECR repository>
  tag: <image tag>
```

The image tag becomes the deployment trigger for the GitOps flow.

---

## 9. Helm Configuration for AWS EKS

The Helm chart was configured for the AWS/EKS environment.

Important values include:

```text
ingress.enabled
ingress.host
db.storageClass
application image
application image tag
```

For the demonstrated EKS environment:

```text
ingress.enabled = true
db.storageClass = gp2
```

The ingress configuration contains AWS Application Load Balancer annotations.

The chart also supports conditional configuration for Docker registry credentials where required.

---

## 10. AI-Assisted Helm Generation

GitHub Copilot was used to assist with generating the Helm chart from the existing Kubernetes manifests.

The process was:

```text
Existing Kubernetes Manifests
        ↓
Initial Copilot Prompt
        ↓
Generated Helm Chart
        ↓
Inspect Output
        ↓
Identify Issues
        ↓
Refine Requirements
        ↓
Regenerate / Adjust
        ↓
Manual Verification
        ↓
Commit Chart
```

The important engineering responsibility remained the definition and verification of the desired chart structure.

AI assistance was used for code generation; it does not change the ownership boundary of the underlying VProfile application.

---

## 11. Helm Validation Before Git Push

Before publishing the Helm chart, the important configuration values were checked.

The implementation specifically required verifying:

- chart structure
- template files
- `values.yaml`
- image repository
- image tags
- ingress configuration
- domain/host
- EKS storage class
- conditional resources

The Helm repository was then committed and pushed.

Example:

```bash
cd vprofile-helm

git add .
git commit -m "add vprofile helm chart"
git push origin main
```

This Git repository subsequently becomes the deployment source watched by Argo CD.

---

## 12. SonarQube Setup

A SonarQube server was prepared as the quality-analysis component of the application CI pipeline.

The implementation used an EC2 instance to host SonarQube.

The pipeline relationship is:

```text
Pull Request
      ↓
GitHub Actions
      ↓
Maven Build / Test
      ↓
Checkstyle
      ↓
SonarQube Analysis
      ↓
Quality Gate
```

A SonarQube authentication token was generated for the pipeline.

The token was not stored in source code.

Instead, it was later configured as a GitHub Actions Secret.

---

## 13. Application Repository Preparation

The application repository contains the application source and build configuration required for CI/CD.

The implementation includes the required build and containerization artifacts, such as:

```text
src/
pom.xml
Dockerfile
sonar-project.properties
.github/
└── workflows/
```

The application source itself is treated as the supplied workload rather than as a portfolio claim of application development.

The DevOps implementation surrounds the workload with build, quality, container, and deployment automation.

---

## 14. GitHub Actions Pipeline Design

The application pipeline uses two logical phases.

### Pull Request — CI

```text
Pull Request
      ↓
Build
      ↓
Unit Tests
      ↓
Checkstyle
      ↓
SonarQube Analysis
      ↓
Quality Gate
      ↓
PASS / FAIL
```

The PR phase is intentionally focused on validation.

It does not create and publish the Docker image.

### Merge to `main` — Build and Delivery

```text
Merge to main
      ↓
Docker Build
      ↓
Amazon ECR
      ↓
Update Helm values.yaml
```

This separates quality validation from creation of the deployable container artifact.

---

## 15. Container Image Build

After a successful merge to `main`, GitHub Actions builds the application container image.

The flow is:

```text
vprofile-app
      ↓
Dockerfile
      ↓
Docker Build
      ↓
Container Image
```

The resulting image is tagged and pushed to Amazon ECR.

The image tag is then used as the deployment version in the Helm repository.

---

## 16. Amazon ECR Configuration

An Amazon ECR repository was created to store application container images.

Conceptually:

```text
GitHub Actions
      │
      ▼
Docker Build
      │
      ▼
Amazon ECR
      │
      ▼
EKS Nodes
      │
      ▼
Kubernetes Pods
```

The ECR repository must exist in the AWS region used by the pipeline.

The implementation therefore keeps the AWS region and ECR repository name as pipeline configuration rather than hardcoding them throughout the workflow.

---

## 17. GitHub Actions AWS Authentication

The demonstrated pipeline authenticates to AWS using credentials stored as GitHub Secrets.

The pipeline configuration uses secrets such as:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

These values are not committed to Git.

The pipeline consumes them at runtime.

Conceptually:

```text
GitHub Secret
      ↓
GitHub Actions
      ↓
AWS Authentication
      ↓
Amazon ECR
```

The practical material uses an IAM identity for the GitHub Actions pipeline.

The credential-based implementation should be treated as a project/demo implementation rather than as the final production security model.

---

## 18. Cross-Repository Git Authentication

The application pipeline needs to update the separate `vprofile-helm` repository.

The authentication relationship is:

```text
vprofile-app GitHub Actions
          │
          ▼
GitHub authentication
          │
          ▼
vprofile-helm
          │
          ▼
values.yaml
```

A GitHub Personal Access Token was used for cross-repository write access.

The pipeline therefore has:

```text
HELM_REPO_USER
GITOPS_PAT
```

available as GitHub Secrets.

The PAT is used to authenticate the pipeline when committing the updated Helm configuration.

---

## 19. GitHub Secrets and Variables

Sensitive credentials are stored as GitHub Secrets.

The demonstrated inventory includes:

```text
SONAR_TOKEN
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
HELM_REPO_USER
GITOPS_PAT
SLACK_WEBHOOK
```

Non-sensitive configuration is stored as GitHub Variables:

```text
AWS_REGION
ECR_REPOSITORY
HELM_REPO_NAME
```

The SonarQube URL is environment-dependent and was configured after the SonarQube server was available.

The naming is an exact contract between GitHub repository configuration and workflow code.

For example:

```yaml
${{ secrets.AWS_ACCESS_KEY_ID }}
```

must correspond exactly to:

```text
AWS_ACCESS_KEY_ID
```

---

## 20. GitHub Actions → Helm Update

After successfully building and pushing the container image, the application pipeline updates the Helm deployment state.

Conceptually:

```text
New Git Commit
      ↓
Docker Image
      ↓
ECR
      ↓
New Image Tag
      ↓
vprofile-helm/helm/vprofile/values.yaml
```

The pipeline modifies the image tag rather than directly modifying Kubernetes resources.

This is the key bridge between CI and GitOps CD.

---

## 21. Terraform Infrastructure Preparation

The infrastructure repository contains Terraform configuration for the AWS/EKS platform.

The implementation follows the bootstrap-then-automate pattern:

```text
Terraform Code
      ↓
Run Locally
      ↓
Validate
      ↓
Plan
      ↓
Apply
      ↓
AWS Infrastructure
      ↓
EKS Cluster
```

The infrastructure was first established and validated before considering pipeline automation.

This reduces the complexity of troubleshooting because the Terraform code can be verified independently of GitHub Actions.

---

## 22. EKS Infrastructure

Terraform is responsible for establishing the AWS/EKS platform required by the application deployment.

The resulting environment includes:

```text
AWS
 │
 ├── Networking / VPC
 │
 └── Amazon EKS
      │
      ├── Control Plane
      │
      └── Managed Node Group
```

The exact infrastructure implementation remains represented by the Terraform source rather than being reproduced in this document.

The important implementation boundary is:

```text
Terraform
    ↓
Infrastructure

Argo CD
    ↓
Application deployment
```

---

## 23. EKS Access Configuration

After the EKS cluster was created, local Kubernetes access was configured.

The kubeconfig was updated with:

```bash
aws eks update-kubeconfig \
  --name vprofile-eks-cluster \
  --region us-east-1
```

Connectivity was then checked with:

```bash
kubectl get pods
```

This establishes the local administrative connection required for the remaining Kubernetes setup.

---

## 24. AWS Load Balancer Controller IAM Configuration

The AWS Load Balancer Controller requires AWS permissions because it manages AWS resources from inside Kubernetes.

The implementation establishes:

```text
IAM Policy
      ↓
IAM Role
      ↓
Kubernetes Service Account
      ↓
AWS Load Balancer Controller
      ↓
AWS APIs
```

The controller policy defines permissions for resources such as:

- Application Load Balancers
- Target Groups
- Security Groups
- VPC-related information

The Kubernetes service account is linked to the IAM role using the EKS IAM service-account mechanism.

---

## 25. cert-manager Installation

cert-manager was installed before the AWS Load Balancer Controller because the controller's admission webhooks require certificates.

The dependency is:

```text
cert-manager
      ↓
Webhook certificates
      ↓
AWS Load Balancer Controller
```

The implementation waits for cert-manager to become ready before proceeding.

Example verification:

```bash
kubectl get all -n cert-manager
```

The expected state is that the cert-manager components are running.

---

## 26. AWS Load Balancer Controller Installation

The AWS Load Balancer Controller was installed through Helm.

Conceptually:

```text
Kubernetes Ingress
      ↓
AWS Load Balancer Controller
      ↓
AWS Application Load Balancer
```

The controller was configured with:

- EKS cluster name
- AWS region
- VPC ID
- existing IAM service account

The controller's service account was reused rather than creating an unrelated service account.

Verification:

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```

The controller webhook endpoints were also checked.

---

## 27. Argo CD Installation

Argo CD was installed into the EKS cluster using Helm.

The implementation uses a dedicated namespace:

```text
argocd
```

The installation follows:

```text
Helm Repository
      ↓
Argo CD Helm Chart
      ↓
argocd namespace
      ↓
Argo CD Server
Repo Server
Application Controller
Supporting Components
```

The practical pins the demonstrated Helm chart version rather than relying on an unbounded latest version.

Verification:

```bash
kubectl get all -n argocd
```

---

## 28. Argo CD Initial Access

The initial Argo CD administrator password is stored in a Kubernetes Secret.

It can be retrieved with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

The initial username is:

```text
admin
```

After initial access, the password should be changed.

Credentials themselves are not stored in the repository.

---

## 29. Argo CD HTTPS Ingress

An Ingress was configured to expose the Argo CD server through an AWS Application Load Balancer.

The traffic flow is:

```text
Browser
    ↓
argocd.<domain>
    ↓
DNS
    ↓
AWS ALB
    ↓
Argo CD Ingress
    ↓
argocd-server
```

Important configuration includes:

```text
ALB ingress class
Internet-facing scheme
IP target type
ACM certificate
HTTP / HTTPS listeners
HTTP → HTTPS redirect
HTTPS backend
```

The ACM certificate ARN and domain are environment-specific values and must not be hardcoded into generic documentation.

---

## 30. DNS Configuration

After the Ingress creates the ALB, its DNS endpoint is obtained from Kubernetes.

Example:

```bash
kubectl get ingress -n argocd
```

The ALB endpoint is then mapped to the desired hostname using a DNS CNAME record.

Conceptually:

```text
argocd.example.com
        ↓
CNAME
        ↓
AWS ALB DNS Name
```

The same pattern is used for the VProfile application endpoint.

---

## 31. Argo CD Repository Registration

Argo CD must be able to read the private `vprofile-helm` repository.

The implementation therefore establishes:

```text
Argo CD
    │
    ▼
GitHub SSH Authentication
    │
    ▼
vprofile-helm
```

The repository can be registered using the Argo CD CLI and the SSH private key.

Conceptually:

```bash
argocd login <argocd-endpoint> --username admin

argocd repo add git@github.com:<account>/vprofile-helm.git \
  --ssh-private-key-path ~/.ssh/<key>
```

Repository connectivity is verified before creating the Application.

---

## 32. EKS Node Access to ECR

The EKS worker nodes also require permission to pull private images from ECR.

The implementation therefore establishes:

```text
EKS Node Group
      ↓
Node IAM Role
      ↓
ECR Permissions
      ↓
Amazon ECR
```

This is independent of the Argo CD Git authentication path.

The two relationships are:

```text
Argo CD → GitHub
    = reads deployment configuration

EKS Nodes → ECR
    = pulls container images
```

A failure in either relationship produces a different class of deployment problem.

---

## 33. Argo CD AppProject

The deployment boundary is defined through an Argo CD `AppProject`.

The project specifies:

```text
Allowed source repository
        ↓
vprofile-helm

Allowed destination
        ↓
vprofile namespace
        ↓
Current EKS cluster

Allowed resource types
        ↓
Kubernetes resources required by the application
```

Example project structure:

```text
vprofile-helm/
└── argocd/
    └── projects/
        └── vprofile-project.yaml
```

The Project must exist before the Application that references it.

---

## 34. Argo CD Application

The Argo CD Application connects the Helm repository to the EKS cluster.

Conceptually:

```text
Application
    │
    ├── Repository
    ├── Branch
    ├── Helm path
    ├── Values file
    ├── Destination cluster
    ├── Destination namespace
    └── Synchronization policy
```

Example repository structure:

```text
vprofile-helm/
├── helm/
│   └── vprofile/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│
└── argocd/
    ├── projects/
    │   └── vprofile-project.yaml
    └── apps/
        └── vprofile-app.yaml
```

The Application source path must exactly match the actual Helm chart location.

---

## 35. Argo CD Synchronization Policy

The demonstrated Application uses automated synchronization:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```

### `prune`

Resources removed from the desired Git state can be removed from Kubernetes.

### `selfHeal`

Manual changes to managed Kubernetes resources can be reconciled back toward the Git-defined state.

### `CreateNamespace=true`

Allows the target application namespace to be created automatically when required.

### `ServerSideApply=true`

Enables Kubernetes server-side apply behavior for resource application.

---

## 36. Applying Argo CD Configuration

The AppProject is applied first:

```bash
kubectl apply \
  -f argocd/projects/vprofile-project.yaml
```

Then the Application:

```bash
kubectl apply \
  -f argocd/apps/vprofile-app.yaml
```

The order matters because the Application references the Project.

After the Application is created, Argo CD begins processing the Helm repository.

---

## 37. GitOps Deployment Flow

Once Argo CD is connected, the application deployment becomes Git-driven.

```text
vprofile-app
      │
      ▼
GitHub Actions
      │
      ├── Build
      ├── Test
      ├── Quality Gate
      └── Docker Build
              │
              ▼
           Amazon ECR
              │
              ▼
       Update Helm values
              │
              ▼
        vprofile-helm
              │
              ▼
           Argo CD
              │
              ▼
        Helm Rendering
              │
              ▼
         Kubernetes
              │
              ▼
        New Application Pod
```

No direct Kubernetes deployment command is required for normal application releases after GitOps has been configured.

---

## 38. Image Tag as the Deployment Trigger

The image tag is the critical connection between CI and CD.

Example:

```text
Before:

app:
  tag: old-version
```

After the application pipeline:

```text
app:
  tag: new-version
```

The resulting chain is:

```text
New image tag
      ↓
Git commit
      ↓
Argo CD detects Git change
      ↓
Helm renders new image
      ↓
Kubernetes Deployment changes
      ↓
New Pod created
      ↓
Old Pod replaced
```

This provides a traceable relationship between the application source revision, container image, Git deployment state, and running workload.

---

## 39. End-to-End Pipeline Test

The final pipeline test uses a small application change to exercise the complete workflow.

Conceptually:

```bash
# Create or switch to feature branch
git checkout -b feature-x

# Make a small change
vim README.md

# Commit
git add .
git commit -m "test pipeline"

# Push feature branch
git push origin feature-x
```

The feature branch push does not perform the deployment.

A Pull Request is then created:

```text
feature-x → main
```

This triggers the quality pipeline.

After the quality gate passes, the PR is merged.

The merge to `main` triggers:

```text
Docker Build
    ↓
ECR Push
    ↓
Helm values.yaml Update
```

---

## 40. Deployment Verification

After the application pipeline completes, the deployment chain is checked.

### ECR

Verify the new image exists.

```text
Amazon ECR
└── new image tag
```

### Helm

Verify the new tag exists in:

```text
vprofile-helm
└── helm/vprofile/values.yaml
```

### Argo CD

Verify the Application is:

```text
Synced
Healthy
```

### Kubernetes

Verify the new Pod:

```bash
kubectl get pods -n vprofile
```

Inspect the deployed image:

```bash
kubectl describe pod <pod-name> -n vprofile
```

---

## 41. Rolling Update

Changing the Deployment image causes Kubernetes to perform a rolling update.

Conceptually:

```text
Old Deployment
      │
      ▼
New image detected
      │
      ▼
New Pod created
      │
      ▼
New Pod becomes healthy
      │
      ▼
Old Pod removed
```

The final state should contain the new image version.

This is the runtime effect of the GitOps configuration change.

---

## 42. Application Verification

The deployed VProfile workload was validated through its externally accessible endpoint.

The verification chain includes:

```text
Application URL
      ↓
DNS
      ↓
ALB
      ↓
Ingress
      ↓
Service
      ↓
Application Pod
```

The application also depends on its backend Kubernetes services.

The practical validation therefore checks that the required workload components are running and reachable.

---

## 43. Common Implementation Failure Points

### Helm path mismatch

If the Argo CD Application points to:

```text
helm/vprofile-chart
```

while the actual chart is:

```text
helm/vprofile
```

the Application cannot render the expected chart.

Always verify the actual repository path.

---

### ECR image pull failure

If Pods show:

```text
ImagePullBackOff
```

check the EKS node IAM role and its ECR permissions.

---

### Argo CD repository connection failure

Check:

- repository URL
- SSH key
- GitHub account
- repository permissions
- Argo CD repository registration

---

### Application creation failure

If the Application references a Project that does not exist:

```text
AppProject
   ↓
must exist first
   ↓
Application
```

Apply the Project before the Application.

---

### Ingress does not create an ALB

Check:

- AWS Load Balancer Controller
- controller Pods
- controller IAM permissions
- Ingress class
- VPC ID
- annotations
- subnet/networking configuration

---

### Controller webhook failure

Check cert-manager first.

The dependency is:

```text
cert-manager ready
      ↓
Load Balancer Controller
      ↓
Ingress
```

---

## 44. Configuration and Secret Hygiene

The implementation does not place credentials directly into Git.

The following must never be committed:

```text
AWS access keys
AWS secret keys
GitHub PATs
SonarQube tokens
Slack webhooks
Private SSH keys
Passwords
Certificates containing private keys
```

Environment-specific values should be represented using placeholders when documenting configuration.

Example:

```text
<YOUR-AWS-ACCOUNT-ID>
<YOUR-VPC-ID>
<YOUR-ACM-CERTIFICATE-ARN>
<YOUR-DOMAIN>
<YOUR-GITHUB-ACCOUNT>
```

---

## 45. Environment-Specific Configuration

The following values are environment-specific:

```text
AWS account ID
AWS region
EKS cluster name
VPC ID
ECR repository
GitHub account/repository URLs
ACM certificate ARN
DNS domain
SonarQube endpoint
GitHub credentials
AWS credentials
```

These should not be treated as portable constants.

The implementation uses configuration and secrets to keep environment-specific information separate from the pipeline logic.

---

## 46. Cleanup Strategy

The environment contains AWS resources that must be removed in dependency order.

The practical cleanup sequence includes:

```text
1. Remove application ingress
        ↓
2. Remove Argo CD ingress
        ↓
3. Remove Load Balancer Controller IAM service account
        ↓
4. Remove Argo CD / Kubernetes resources
        ↓
5. Remove remaining AWS dependencies
        ↓
6. Terraform destroy
```

Ingress resources should be removed before destroying the underlying infrastructure because the AWS Load Balancer Controller may still manage AWS load balancers created from those resources.

---

## 47. Rebuild Strategy

The project also supports a clean rebuild workflow.

The intended pattern is:

```text
Clean Environment
      ↓
Terraform
      ↓
EKS
      ↓
Controllers
      ↓
Argo CD
      ↓
GitOps Application
      ↓
Validation
```

For the learning/demo environment, a fresh clone can also be used to avoid unnecessary local Git conflicts.

The purpose is to verify that the architecture is reproducible rather than dependent on an accidental local state.

---

## 48. Implementation Boundaries

This implementation demonstrates:

- Git repository separation
- SSH-based Git authentication
- Helm chart parameterization
- AI-assisted Helm generation
- GitHub Actions CI/CD
- SonarQube integration
- Amazon ECR integration
- GitHub Secrets and Variables
- cross-repository Git authentication
- Terraform-based EKS provisioning
- EKS access configuration
- IRSA for the AWS Load Balancer Controller
- cert-manager
- AWS Load Balancer Controller
- Argo CD installation
- Argo CD repository registration
- AppProject configuration
- Application configuration
- automated synchronization
- ALB/Ingress
- HTTPS
- DNS
- end-to-end GitOps deployment

The project does not claim:

- enterprise production hardening
- complete observability
- enterprise secrets management
- multi-cluster deployment
- blue/green deployment
- canary deployment
- disaster recovery
- automated multi-environment promotion
- enterprise policy enforcement

---

## 49. Implementation Summary

The complete implementation can be reconstructed as:

```text
GitHub Repositories
        ↓
SSH Authentication
        ↓
Helm Chart
        ↓
GitHub Actions
        ↓
SonarQube
        ↓
Docker
        ↓
Amazon ECR
        ↓
Terraform
        ↓
Amazon EKS
        ↓
cert-manager
        ↓
AWS Load Balancer Controller
        ↓
Argo CD
        ↓
Argo CD Project
        ↓
Argo CD Application
        ↓
Helm Repository
        ↓
Kubernetes
        ↓
VProfile Application
```

The most important implementation boundary is:

```text
CI
 ↓
Build + Quality + Image
 ↓
Git deployment-state update
 ↓
Argo CD
 ↓
Kubernetes
```

while:

```text
Terraform
 ↓
AWS / EKS infrastructure
```

remains a separate infrastructure lifecycle.

---

## 50. Implementation Memory Trigger

To reconstruct the project quickly:

> **Create the three repositories → build the Helm chart → configure CI and quality gates → publish images to ECR → update Helm image state → provision EKS with Terraform → configure EKS access and AWS Load Balancer Controller → install Argo CD → connect Argo CD to the Helm repository → create Project and Application → validate Git-to-EKS deployment.**
