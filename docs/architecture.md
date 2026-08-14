# Architecture

[← Back to README](../README.md)

## 1. Architecture Overview

This project implements an end-to-end GitOps delivery architecture for deploying the existing VProfile application workload to Amazon EKS.

The architecture separates three major concerns:

```text
┌───────────────────────────────────────────────────────────────┐
│                    SOURCE / DESIRED STATE                    │
│                                                               │
│  vprofile-app        vprofile-helm        vprofile-infra     │
│  Application         Deployment           Infrastructure     │
│  Source              Configuration        Terraform          │
└─────────────┬──────────────┬──────────────────┬──────────────┘
              │              │                  │
              │              │                  │
              ▼              │                  ▼
       GitHub Actions        │             Terraform
       Application CI        │                  │
              │              │                  ▼
              │              │             AWS / EKS
              │              │
              ▼              │
        Amazon ECR           │
              │              │
              │              ▼
              │           Argo CD
              │              │
              │              ▼
              └──────────► Amazon EKS
                             │
                             ▼
                       Kubernetes Workload
```

The central GitOps boundary is:

```text
CI
 │
 ├── builds and validates the application
 ├── creates the container image
 ├── publishes the image to ECR
 └── updates deployment state in Git
                         │
                         ▼
                       Git
                         │
                         ▼
                      Argo CD
                         │
                         ▼
                    Kubernetes
```

The key principle is:

> **CI changes the desired state in Git; Argo CD reconciles that desired state into Kubernetes.**

---

## 2. Project Ownership Boundary

The VProfile application is the existing workload used by the project.

This architecture therefore separates:

```text
Application Workload
        ≠
DevOps Platform Around the Workload
```

The application itself was not developed as part of this project.

The engineering architecture covered by this repository is:

- application CI/CD
- container image delivery
- Helm deployment configuration
- AWS infrastructure
- Amazon EKS
- Kubernetes deployment
- Argo CD GitOps reconciliation
- AWS Load Balancer Controller
- IAM integration
- HTTPS ingress
- DNS
- validation

The architecture documentation intentionally does not claim ownership of the VProfile application's business logic or Java application design.

---

## 3. Repository Architecture

The operational project uses three Git repositories.

```text
                    GitHub
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
vprofile-app     vprofile-helm    vprofile-infra
       │               │                │
       │               │                │
       ▼               ▼                ▼
 Application       Helm +           Terraform
 source + CI       Argo CD          infrastructure
                   manifests
```

### `vprofile-app`

Contains the application source and application CI/CD workflow.

Responsibilities include:

- application build
- unit tests
- Checkstyle
- SonarQube analysis
- quality gate
- Docker image build
- Amazon ECR publication
- deployment-state update

### `vprofile-helm`

Contains the Kubernetes deployment configuration.

Responsibilities include:

- Helm chart
- `values.yaml`
- deployment configuration
- Argo CD project manifest
- Argo CD application manifest

This repository is watched by Argo CD.

### `vprofile-infra`

Contains Terraform infrastructure configuration.

Its responsibility is the AWS/EKS infrastructure lifecycle.

The repository separation provides independent lifecycle and access boundaries for:

```text
Application
Deployment
Infrastructure
```

---

## 4. Source-of-Truth Model

Git is the source of the desired configuration.

The architecture can be divided into:

```text
GitHub
  │
  │ Desired State
  │
  ├── Application Source
  ├── Deployment Configuration
  └── Infrastructure Configuration
  │
  ▼
AWS
  │
  │ Actual Runtime State
  │
  ├── EKS
  ├── Kubernetes resources
  ├── ECR
  ├── ALB
  └── supporting AWS resources
```

This produces a clear relationship:

```text
Git
 ↓
Desired State

AWS / Kubernetes
 ↓
Actual State
```

Argo CD continuously compares the desired deployment state in Git with the actual Kubernetes state.

---

## 5. Application CI/CD Architecture

The application pipeline has two principal execution paths.

### Pull Request Path

```text
Pull Request
     │
     ▼
GitHub Actions
     │
     ├── Build
     ├── Unit Test
     ├── Checkstyle
     ├── SonarQube Analysis
     └── Quality Check
             │
             ▼
          PASS / FAIL
```

This path validates application changes before they are merged.

### Main Branch Path

After a successful merge:

```text
Merge to main
      │
      ▼
GitHub Actions
      │
      ├── Build Docker Image
      │
      ▼
   Amazon ECR
      │
      ▼
Update Helm image tag
      │
      ▼
vprofile-helm
```

The important architectural transition is:

```text
Application Source
        ↓
Container Image
        ↓
Deployment State
```

CI does not directly perform the Kubernetes deployment.

---

## 6. Container Image Architecture

The container image is the artifact connecting application CI to Kubernetes deployment.

```text
Application Source
       │
       ▼
GitHub Actions
       │
       ▼
Docker Build
       │
       ▼
Container Image
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

The image reference is stored in the Helm deployment configuration.

Conceptually:

```yaml
image:
  repository: <ECR repository>
  tag: <image-tag>
```

The tag therefore becomes part of the Git-managed desired state.

---

## 7. ECR → EKS Image Flow

The EKS nodes require AWS permissions to retrieve private container images from ECR.

The relationship is:

```text
EKS Cluster
    │
    ▼
Node Group
    │
    ▼
Node IAM Role
    │
    ▼
ECR Permissions
    │
    ▼
Amazon ECR
    │
    ▼
Container Image
    │
    ▼
Kubernetes Pod
```

Without the required ECR permissions, Pods attempting to use private ECR images can fail with image-pull errors.

This creates a separate AWS identity boundary from the GitOps deployment mechanism:

```text
Argo CD
    │
    └── manages Kubernetes desired state

EKS Node IAM Role
    │
    └── allows image retrieval from ECR
```

---

## 8. Helm Deployment Architecture

Helm provides the deployment packaging layer between Git and Kubernetes.

The repository contains a Helm chart with configuration such as:

```text
vprofile-helm
└── helm
    └── vprofile
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

The chart parameterizes Kubernetes resources rather than storing every deployment value directly in each manifest.

The important deployment parameter is the container image:

```text
values.yaml
     │
     └── image repository + tag
              │
              ▼
        Helm rendering
              │
              ▼
      Kubernetes manifests
```

This gives the CI pipeline a controlled place to update the deployed image version.

---

## 9. Argo CD Architecture

Argo CD runs inside the EKS cluster as the GitOps continuous deployment controller.

Its architectural role is:

```text
vprofile-helm
      │
      │ Git
      ▼
   Argo CD
      │
      │ Reconciliation
      ▼
 Kubernetes API
      │
      ▼
 EKS Workloads
```

Argo CD watches the Helm repository and compares:

```text
Desired State
     │
     ▼
Git / Helm
     │
     │ compare
     ▼
Actual State
     │
     ▼
Kubernetes
```

If the states differ, Argo CD can synchronize the Kubernetes resources toward the desired Git state.

This is the pull-based deployment model used by the project.

---

## 10. Argo CD Project and Application

The Argo CD configuration has two important logical objects:

```text
AppProject
    │
    └── defines boundaries

Application
    │
    ├── defines source
    ├── defines destination
    ├── defines Helm path
    └── defines synchronization behavior
```

### AppProject

The project defines boundaries such as:

- permitted source repository
- permitted destination namespace
- allowed resource types

Conceptually:

```text
vprofile-project
       │
       ├── Source
       │     └── vprofile-helm
       │
       └── Destination
             └── vprofile namespace
```

### Application

The Argo CD Application identifies:

```text
Repository
    ↓
Target revision
    ↓
Helm chart path
    ↓
Values file
    ↓
Kubernetes destination
```

The application is therefore the connection between:

```text
Git repository
       +
Helm configuration
       +
Kubernetes cluster
```

---

## 11. Argo CD Synchronization Policy

The demonstrated Application uses automated synchronization behavior.

The important settings are:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### `prune`

If a resource is removed from the desired Git state, Argo CD can remove the corresponding resource from Kubernetes.

Conceptually:

```text
Git
 │
 └── resource removed
          │
          ▼
       Argo CD
          │
          ▼
Kubernetes resource removed
```

### `selfHeal`

If the live Kubernetes state is manually changed away from the desired Git configuration, Argo CD can reconcile it back toward Git.

Conceptually:

```text
Git desired state
       │
       ▼
     Argo CD
       │
       │ compare
       ▼
Kubernetes actual state
       │
       ▼
Manual drift detected
       │
       ▼
Reconciliation
       │
       ▼
Desired state restored
```

This is the central reconciliation behavior of the GitOps model.

---

## 12. Kubernetes Resource Architecture

The VProfile workload is represented by Kubernetes resources managed through Helm and Argo CD.

The resource relationship is conceptually:

```text
Argo CD
   │
   ▼
Helm
   │
   ├── Deployment
   │      │
   │      ▼
   │    Pods
   │
   ├── Service
   │
   ├── PersistentVolumeClaim
   │
   ├── Ingress
   │
   └── Secret
```

The exact application behavior remains the responsibility of the supplied workload.

The DevOps architecture is responsible for packaging, deploying, exposing, and managing the workload on EKS.

---

## 13. AWS Load Balancer Controller Architecture

The project uses the AWS Load Balancer Controller to translate Kubernetes Ingress configuration into AWS Application Load Balancers.

The relationship is:

```text
Kubernetes Ingress
        │
        ▼
AWS Load Balancer Controller
        │
        ▼
AWS Application Load Balancer
        │
        ▼
Target Group
        │
        ▼
Kubernetes Pod
```

The controller uses Kubernetes Ingress annotations to determine the desired AWS load-balancer behavior.

The project uses an internet-facing ALB configuration with Pod IP targeting and HTTPS-related configuration.

---

## 14. AWS Load Balancer Controller Identity Architecture

The controller requires AWS permissions.

The project establishes this through an IAM-backed Kubernetes service account.

```text
IAM Policy
     │
     ▼
IAM Role
     │
     ▼
Kubernetes Service Account
     │
     ▼
AWS Load Balancer Controller Pods
     │
     ▼
AWS APIs
     │
     ├── Application Load Balancers
     ├── Target Groups
     └── Security Groups
```

This is the project's IRSA-based permission chain.

The important boundary is:

```text
Kubernetes Controller
        ↓
IAM-backed identity
        ↓
AWS API permissions
```

The controller does not need unrestricted AWS credentials embedded in the application workload.

---

## 15. cert-manager Dependency

The AWS Load Balancer Controller uses admission webhooks that require certificates.

The dependency is:

```text
cert-manager
      │
      ▼
Webhook certificates
      │
      ▼
AWS Load Balancer Controller
```

Therefore the controller installation depends on cert-manager being available and ready.

---

## 16. HTTPS / Ingress Architecture

The application access path is:

```text
User
 │
 ▼
Application Domain
 │
 ▼
DNS
 │
 ▼
AWS Application Load Balancer
 │
 ▼
Kubernetes Ingress
 │
 ▼
Application Service
 │
 ▼
Application Pod
```

The ALB configuration uses:

- HTTP listener
- HTTPS listener
- ACM certificate
- HTTP-to-HTTPS redirection
- Pod IP target routing

Conceptually:

```text
HTTP :80
   │
   ▼
Redirect
   │
   ▼
HTTPS :443
   │
   ▼
ACM Certificate
   │
   ▼
ALB
   │
   ▼
Ingress
   │
   ▼
Service
   │
   ▼
Pod
```

---

## 17. Argo CD Access Architecture

Argo CD itself is also exposed through an ingress and AWS ALB.

The access path is:

```text
Browser
   │
   ▼
Argo CD Domain
   │
   ▼
DNS
   │
   ▼
AWS ALB
   │
   ▼
Argo CD Ingress
   │
   ▼
Argo CD Service
   │
   ▼
Argo CD Server
```

This allows Argo CD to be accessed through a domain rather than requiring direct access to an internal Kubernetes service.

---

## 18. Complete CI → GitOps → Kubernetes Flow

The complete application delivery architecture can be represented as:

```text
                    APPLICATION DELIVERY

Developer
    │
    ▼
vprofile-app
    │
    ▼
GitHub Actions
    │
    ├── Build
    ├── Unit Test
    ├── Checkstyle
    ├── SonarQube
    └── Quality Gate
            │
            ▼
       Merge to main
            │
            ▼
       Docker Build
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
     Kubernetes API
            │
            ▼
        Amazon EKS
            │
            ▼
      Kubernetes Pods
            │
            ▼
        Application
```

This is the primary architecture of the project.

---

## 19. Infrastructure Architecture

The infrastructure side is represented separately:

```text
vprofile-infra
      │
      ▼
 Terraform
      │
      ▼
 AWS Infrastructure
      │
      ├── VPC / networking
      ├── EKS control plane
      ├── Managed node groups
      └── supporting infrastructure
              │
              ▼
          Amazon EKS
```

Terraform provides the infrastructure lifecycle.

The application GitOps lifecycle begins after the EKS platform is available.

This creates a useful separation:

```text
Infrastructure Lifecycle
        │
        ▼
     Terraform
        │
        ▼
      AWS / EKS


Application Lifecycle
        │
        ▼
   GitHub Actions
        │
        ▼
    Helm + Argo CD
        │
        ▼
      Kubernetes
```

---

## 20. Infrastructure and Application Boundary

The two major lifecycle domains are:

### Infrastructure

```text
Terraform
   ↓
AWS
   ↓
EKS
```

### Application Delivery

```text
GitHub Actions
   ↓
ECR
   ↓
Helm
   ↓
Argo CD
   ↓
EKS workload
```

Terraform establishes the platform.

GitOps manages the application deployment state within that platform.

This separation prevents application CI/CD from becoming responsible for directly provisioning the underlying Kubernetes infrastructure.

---

## 21. Complete System Architecture

The entire project can be compressed into four layers:

```text
┌───────────────────────────────────────────────────────────┐
│  1. SOURCE / DESIRED STATE                               │
│                                                           │
│  GitHub                                                   │
│   ├── vprofile-app                                       │
│   ├── vprofile-helm                                      │
│   └── vprofile-infra                                     │
└──────────────────────────────┬────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────┐
│  2. AUTOMATION                                            │
│                                                           │
│  GitHub Actions                                           │
│   ├── CI                                                  │
│   ├── Quality Gate                                       │
│   ├── Docker Build                                       │
│   └── ECR Publication                                    │
│                                                           │
│  Terraform                                                │
│   └── AWS / EKS infrastructure lifecycle                 │
└──────────────────────────────┬────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────┐
│  3. GITOPS RECONCILIATION                                 │
│                                                           │
│  Argo CD                                                  │
│   ├── Watches Helm repository                             │
│   ├── Compares desired vs actual state                    │
│   ├── Synchronizes resources                              │
│   ├── Prunes removed resources                             │
│   └── Self-heals detected drift                           │
└──────────────────────────────┬────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────┐
│  4. RUNTIME                                               │
│                                                           │
│  Amazon EKS                                               │
│   ├── Kubernetes workloads                                │
│   ├── Services                                            │
│   ├── Ingress                                             │
│   ├── AWS Load Balancer Controller                        │
│   ├── Argo CD                                             │
│   └── Application Pods                                    │
│                                                           │
│  AWS                                                     │
│   ├── ECR                                                 │
│   ├── ALB                                                 │
│   ├── IAM                                                 │
│   └── DNS / HTTPS supporting services                     │
└───────────────────────────────────────────────────────────┘
```

---

## 22. Architectural Decisions

### Decision 1 — Separate Application, Deployment, and Infrastructure Repositories

The project uses:

```text
vprofile-app
vprofile-helm
vprofile-infra
```

This separates application source, deployment configuration, and infrastructure lifecycle.

---

### Decision 2 — Use Git as the Deployment Source of Truth

Deployment configuration is stored in Git rather than being maintained only inside the cluster.

This provides:

- version history
- traceability
- reviewable configuration changes
- reproducibility
- GitOps reconciliation

---

### Decision 3 — Use Helm for Deployment Packaging

Helm provides parameterization around Kubernetes resources and provides a controlled location for image-version configuration.

---

### Decision 4 — Use Argo CD for Pull-Based Deployment

The CI pipeline does not directly push Kubernetes manifests into the cluster.

Instead:

```text
CI
 ↓
Git
 ↓
Argo CD
 ↓
Kubernetes
```

This keeps deployment reconciliation inside the cluster.

---

### Decision 5 — Use Amazon ECR for Container Images

ECR provides the AWS-native image registry used by the EKS workload.

The EKS node IAM role provides the permissions required for image retrieval.

---

### Decision 6 — Use AWS Load Balancer Controller

The project uses the AWS Load Balancer Controller to integrate Kubernetes Ingress with AWS Application Load Balancers.

This allows Kubernetes configuration to drive the required AWS load-balancer resources.

---

### Decision 7 — Use Terraform for EKS Infrastructure

Terraform provides declarative infrastructure configuration and lifecycle management for the AWS/EKS platform.

---

## 23. Drift and Reconciliation Model

The application deployment architecture intentionally distinguishes:

```text
Desired State
     │
     ▼
Git
     │
     ▼
Argo CD
     │
     ▼
Actual State
     │
     ▼
Kubernetes
```

If the actual Kubernetes state changes independently of Git:

```text
Manual Kubernetes Change
          │
          ▼
     Actual State
          │
          │ differs
          ▼
       Argo CD
          │
          ▼
     Reconciliation
          │
          ▼
   Desired Git State
```

With `selfHeal` enabled, the demonstrated configuration is intended to restore drift toward the Git-defined state.

---

## 24. Deployment Approval Boundary

The demonstrated course pipeline updates the Helm deployment state directly.

Therefore the current flow is:

```text
CI
 ↓
ECR
 ↓
Helm update
 ↓
Argo CD
 ↓
EKS
```

A more mature production-oriented architecture would introduce an explicit review point:

```text
CI
 ↓
ECR
 ↓
Helm Pull Request
 ↓
Human Approval
 ↓
Merge
 ↓
Argo CD
 ↓
EKS
```

This approval-based flow is a future improvement, not a completed capability of the current project.

---

## 25. Architectural Boundaries

This project demonstrates the complete GitOps workflow covered by the practical material.

It does not establish:

- enterprise multi-cluster GitOps
- complete observability
- enterprise secrets management
- disaster recovery
- blue/green deployment
- canary deployment
- complete environment promotion strategy
- enterprise policy enforcement
- automated Terraform approval workflow
- full automated infrastructure drift remediation
- enterprise-scale platform governance

These should be treated as future engineering work rather than completed architecture capabilities.

---

## 26. Architecture Summary

The project can ultimately be reduced to one engineering model:

```text
                 GIT
                  │
      ┌───────────┼───────────┐
      │           │           │
      ▼           ▼           ▼
 Application   Helm       Terraform
   Source      Desired     Infra
                State       State
      │           │           │
      ▼           ▼           ▼
 GitHub       Argo CD    Terraform
 Actions          │           │
      │           │           │
      ▼           ▼           ▼
     ECR       Kubernetes     AWS
                  │
                  ▼
              Amazon EKS
                  │
                  ▼
             Application
```

The core relationship is:

```text
Terraform
   ↓
Build the platform

GitHub Actions
   ↓
Build and publish the application

Helm
   ↓
Define deployment state

Argo CD
   ↓
Reconcile deployment state

Amazon EKS
   ↓
Run the workload
```

This separation of responsibilities is the central architectural idea demonstrated by the project.
