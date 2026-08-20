# VProfile GitOps — End-to-End CI/CD on AWS EKS

An end-to-end **GitOps delivery project** that deploys the existing VProfile application workload to **Amazon EKS** using **GitHub Actions, Docker, Amazon ECR, Helm, Argo CD, and Terraform**.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/f97188fc-ca7a-47ac-92dd-2960bb30d737" />


> **Project focus:** The engineering work in this repository is the infrastructure, CI/CD, Kubernetes deployment, GitOps integration, and validation surrounding the VProfile workload. The VProfile application itself was provided as the project workload and was not developed as part of this project.

---

## Overview

This project demonstrates how an application change can move from source control to a running Kubernetes workload through a GitOps-based delivery workflow.

```text
Developer
    │
    ▼
GitHub Application Repository
    │
    ▼
GitHub Actions
    │
    ├── Build
    ├── Test
    ├── Checkstyle
    └── SonarQube Quality Gate
            │
            ▼
      Docker Image
            │
            ▼
       Amazon ECR
            │
            ▼
       Helm Repository
            │
            ▼
          Argo CD
            │
            ▼
        Amazon EKS
            │
            ▼
   Kubernetes Workload
```

The infrastructure foundation is provisioned separately:

```text
Terraform
    │
    ▼
AWS Infrastructure
    │
    ▼
Amazon EKS
    │
    ├── Kubernetes Workloads
    ├── AWS Load Balancer Controller
    ├── Ingress
    └── Argo CD
```

---

## Project Objective

The objective was to implement and validate an end-to-end DevOps delivery platform in which:

1. Application changes are managed through Git.
2. GitHub Actions performs application CI.
3. SonarQube provides code-quality validation.
4. A container image is built and published to Amazon ECR.
5. The deployment image reference is maintained through Helm.
6. Argo CD watches the desired deployment state in Git.
7. Argo CD reconciles that state into Amazon EKS.
8. Kubernetes performs the resulting workload update.
9. The final application state is validated end to end.

The core GitOps principle demonstrated by the project is:

> **CI changes the desired state in Git; Argo CD reconciles that desired state into Kubernetes.**

---

## Application & Ownership Boundary

### VProfile Application

The VProfile application was used as the **existing application workload** for this project.

The application itself was not developed as part of this work.

The engineering focus was the DevOps platform surrounding the workload:

- AWS infrastructure
- Kubernetes platform
- Helm deployment configuration
- GitHub Actions CI/CD
- Docker image delivery
- Amazon ECR integration
- Argo CD GitOps deployment
- AWS Load Balancer Controller
- HTTPS ingress
- DNS configuration
- validation and troubleshooting

This distinction is intentional: the project demonstrates **DevOps engineering around an existing workload**, rather than application-development ownership.

---

## Architecture

![VProfile GitOps Architecture](architecture.png)

At a high level, the system separates three concerns:

| Repository / Component | Responsibility |
|---|---|
| `vprofile-app` | Application source and CI |
| `vprofile-helm` | Kubernetes deployment state |
| `vprofile-infra` | AWS/EKS infrastructure |
| GitHub Actions | CI and delivery automation |
| Amazon ECR | Container image registry |
| Argo CD | GitOps reconciliation |
| Amazon EKS | Kubernetes execution platform |

The operational repository separation allows application, deployment configuration, and infrastructure to have distinct responsibilities and lifecycles.

---

## My Engineering Contribution

The project involved hands-on implementation and configuration across the delivery platform.

### Infrastructure

- Provisioned the EKS environment using Terraform.
- Configured the AWS infrastructure required for the Kubernetes platform.
- Configured Kubernetes access and supporting AWS integrations.

### Kubernetes & AWS

- Configured the AWS Load Balancer Controller.
- Configured the IAM service account required by the controller.
- Configured Kubernetes Ingress resources.
- Configured HTTPS-related components.
- Configured DNS for application access.
- Integrated the Kubernetes workload with AWS networking.

### Helm

- Converted Kubernetes deployment configuration into a Helm-based deployment structure.
- Parameterized deployment values such as the container image.
- Used Helm values as the deployment-state interface consumed by Argo CD.

### CI/CD

- Configured GitHub Actions for the application workflow.
- Integrated build and validation stages.
- Integrated Checkstyle and SonarQube quality validation.
- Built the Docker image.
- Published the image to Amazon ECR.
- Updated the Helm deployment image reference after a successful application build.

### GitOps

- Installed and configured Argo CD.
- Connected Argo CD to the Helm repository.
- Configured the Argo CD Application.
- Enabled automated synchronization behavior.
- Validated reconciliation from Git state into the EKS cluster.

### Validation

- Validated the CI pipeline.
- Validated image publication to ECR.
- Validated Helm deployment-state changes.
- Validated Argo CD synchronization.
- Validated Kubernetes workload health.
- Validated application deployment and image updates.
- Validated the end-to-end delivery path.

---

## End-to-End Delivery Flow

A successful application change follows this general sequence:

```text
Feature Branch
      │
      ▼
Pull Request
      │
      ▼
GitHub Actions
      │
      ├── Build
      ├── Test
      ├── Checkstyle
      └── SonarQube Quality Gate
              │
              ▼
          Merge to main
              │
              ▼
        Docker Image Build
              │
              ▼
          Amazon ECR
              │
              ▼
       Helm values update
              │
              ▼
          Helm Git Repo
              │
              ▼
            Argo CD
              │
              ▼
        Desired State
              │
              ▼
           Amazon EKS
              │
              ▼
       Kubernetes rollout
```

The important architectural boundary is that Argo CD is responsible for the Kubernetes deployment side of the GitOps workflow.

---

## Key Engineering Capabilities Demonstrated

### GitOps

- Git as the desired-state source
- Pull-based deployment through Argo CD
- Automated reconciliation
- Synchronization between Git and Kubernetes
- Deployment-state versioning through Git

### CI/CD

- GitHub Actions
- Pull-request validation
- Build and test execution
- Static/code-quality validation
- Container image creation
- Image publication to ECR
- Deployment-state update

### AWS

- Amazon EKS
- Amazon ECR
- IAM
- IAM service accounts / IRSA
- AWS Load Balancer Controller
- Application Load Balancer
- DNS
- HTTPS-related configuration

### Kubernetes

- Deployments
- Services
- Ingress
- Namespaces
- Pods
- Rolling updates
- Kubernetes application validation

### Helm

- Helm charts
- `values.yaml`
- Parameterized deployment configuration
- Container image version management

### Infrastructure as Code

- Terraform-based EKS infrastructure provisioning

---

## Validation

The project was validated across multiple layers rather than relying only on successful command execution.

### CI Validation

Confirmed that:

- application changes triggered the expected GitHub Actions workflow
- build and validation stages completed
- SonarQube quality validation was executed
- the container image was produced
- the image was published to Amazon ECR

### GitOps Validation

Confirmed that:

- the Helm deployment state changed with the new image reference
- Argo CD detected the desired-state change
- the Application reached the expected synchronized/healthy state
- Kubernetes resources reflected the desired state

### Kubernetes Validation

Confirmed:

- workload resources were running
- Pods became healthy
- the new image was deployed
- the application remained accessible after the update

### End-to-End Validation

The final validation connected:

```text
Git change
    ↓
CI
    ↓
Quality Gate
    ↓
Container Image
    ↓
ECR
    ↓
Helm
    ↓
Argo CD
    ↓
EKS
    ↓
Running Application
```

---

## Project Boundaries

This project demonstrates a complete hands-on GitOps workflow, but it should **not** be interpreted as an enterprise production platform.

The implementation does not establish:

- enterprise-grade observability
- disaster recovery
- multi-cluster GitOps
- comprehensive secrets management
- advanced deployment strategies such as blue/green or canary deployment
- complete enterprise security hardening
- automated Terraform approval workflows
- automated infrastructure drift remediation
- multi-environment promotion
- enterprise policy enforcement

The course's final maturity tasks identify several of these areas as subsequent improvements, including PR-based Helm promotion, Terraform CI/CD with manual approval, Slack notifications, and drift detection.

---

## Current Deployment Approval Model

The demonstrated application pipeline updates the Helm deployment state directly after the CI workflow succeeds.

A more production-oriented evolution would be:

```text
Application CI
      │
      ▼
Build + Test + Quality Gate
      │
      ▼
Push Image → ECR
      │
      ▼
Create Helm Pull Request
      │
      ▼
Human Review / Approval
      │
      ▼
Merge
      │
      ▼
Argo CD
      │
      ▼
EKS
```

This approval-based promotion model is intentionally documented as **future work**, not as a capability already implemented by this project.

---

## Future Engineering Direction

The next maturity steps for this platform would include:

1. **PR-based Helm promotion**
   - Replace direct Helm repository updates with pull requests.
   - Introduce an explicit deployment approval point.

2. **Terraform CI/CD**
   - Run `terraform validate` and `terraform plan` for branches and pull requests.
   - Require manual approval before `terraform apply`.

3. **Drift detection**
   - Periodically compare infrastructure state against Terraform configuration.
   - Surface infrastructure changes requiring review.

4. **Pipeline notifications**
   - Extend Slack notifications across application and infrastructure pipelines.

5. **Production hardening**
   - Improve secrets management.
   - Add stronger security controls.
   - Introduce observability.
   - Add environment promotion controls.
   - Evaluate advanced deployment strategies.

These are future improvements and are **not represented as completed capabilities**.

---

## Technologies

| Area | Technologies |
|---|---|
| Version Control | Git, GitHub |
| CI/CD | GitHub Actions |
| Code Quality | SonarQube, Checkstyle |
| Containers | Docker |
| Registry | Amazon ECR |
| Infrastructure | Terraform |
| Cloud | AWS |
| Kubernetes | Amazon EKS, Kubernetes |
| Deployment | Helm |
| GitOps | Argo CD |
| Networking | AWS Load Balancer Controller, ALB, Ingress |
| Identity | AWS IAM, IAM Service Accounts / IRSA |
| DNS / HTTPS | DNS, cert-manager, HTTPS |
| Notifications | Slack |

---

## Repository Navigation

### Architecture

Understand the system architecture, repository boundaries, traffic flow, GitOps model, security boundaries, and major engineering decisions.

→ [Architecture](docs/architecture.md)

### Implementation

Understand how the environment and delivery platform were assembled, configured, and deployed.

→ [Implementation](docs/implementation.md)

### Validation

Understand the validation strategy, important checks, end-to-end results, and evidence supporting the project claims.

→ [Validation](docs/validation.md)

### Limitations & Future Work

Understand the project's current boundaries and the engineering improvements that would move it toward a more production-oriented platform.

→ [Limitations & Future Work](docs/limitations-and-future-work.md)

### Evidence

→ [Evidence](evidence/screenshots/)

---

## Project Summary

This project demonstrates the practical integration of:

```text
Git
+
GitHub Actions
+
Docker
+
Amazon ECR
+
Terraform
+
Kubernetes
+
Amazon EKS
+
Helm
+
Argo CD
=
End-to-End GitOps Delivery
```

The primary engineering outcome is not the VProfile application itself.

It is the **DevOps delivery system built around the application**:

> **Code change → CI validation → container image → Git deployment state → Argo CD reconciliation → Kubernetes workload**

That workflow forms the core of this project's engineering story.
