# Limitations and Future Work

[← Back to README](../README.md) | [Architecture](architecture.md) | [Implementation](implementation.md) | [Validation](validation.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/96e6891d-f7ee-4237-bd79-750bf5aca9d7" />


## 1. Purpose

This document defines the current boundaries of the VProfile GitOps project and identifies the engineering improvements that would move the platform toward a more mature production-oriented implementation.

The project demonstrates an end-to-end GitOps delivery workflow:

```text
Application Source
      ↓
GitHub Actions
      ↓
Quality Gate
      ↓
Docker Image
      ↓
Amazon ECR
      ↓
Helm Deployment State
      ↓
Argo CD
      ↓
Amazon EKS
      ↓
Running Application
```

The project should not be interpreted as a complete enterprise production platform.

The purpose of this document is therefore to distinguish clearly between:

```text
Completed / Demonstrated
        ↓
Current Project Boundary
        ↓
Future Engineering Work
```

---

# 2. Current Project Boundary

The current project demonstrates:

- Git-based application source management
- GitHub Actions application CI/CD
- Maven build and testing
- Checkstyle validation
- SonarQube quality analysis
- quality-gate validation
- Docker image creation
- Amazon ECR image publication
- Helm-based Kubernetes deployment configuration
- Terraform-based AWS/EKS infrastructure
- Amazon EKS
- Kubernetes workload deployment
- AWS Load Balancer Controller
- ALB / Ingress integration
- HTTPS
- DNS
- Argo CD
- Argo CD AppProject
- Argo CD Application
- automated GitOps synchronization
- Kubernetes rolling update behavior
- end-to-end deployment validation

The core engineering model is:

```text
CI
 ↓
Build / Test / Quality
 ↓
Container Image
 ↓
Git Deployment State
 ↓
Argo CD
 ↓
Kubernetes
```

Infrastructure remains a separate lifecycle:

```text
Terraform
 ↓
AWS
 ↓
EKS
```

---

# 3. Limitation — Application Repository Directly Updates Deployment State

The current application delivery model allows the application CI pipeline to update the Helm deployment repository after the image is published.

The current flow is:

```text
Application CI
      ↓
Docker Build
      ↓
ECR
      ↓
Update Helm values.yaml
      ↓
Argo CD
      ↓
EKS
```

This works for the demonstrated GitOps workflow, but it reduces the explicit human approval boundary between:

```text
Build Artifact
```

and:

```text
Deployment
```

A production-oriented workflow could introduce a pull request in the deployment repository.

---

# 4. Future Work — PR-Based Helm Promotion

The deployment promotion process could evolve to:

```text
Application CI
      ↓
Build + Test + Quality Gate
      ↓
Docker Image
      ↓
Amazon ECR
      ↓
Create Helm Pull Request
      ↓
Human Review
      ↓
Merge
      ↓
Argo CD
      ↓
EKS
```

Instead of directly modifying:

```text
vprofile-helm
```

the application pipeline would create a deployment change that can be reviewed before it becomes the desired state.

### Benefits

This would provide:

- explicit deployment approval
- stronger change review
- clearer separation between CI and CD
- improved auditability
- safer production promotion

PR-based Helm promotion is a future maturity improvement rather than a completed capability.

---

# 5. Limitation — Infrastructure Pipeline Maturity

Terraform provides the infrastructure-as-code foundation for the AWS/EKS environment.

However, the current project should not claim a fully production-hardened Terraform delivery system merely because Terraform is used.

A mature infrastructure lifecycle would include:

```text
Terraform Change
      ↓
Pull Request
      ↓
terraform validate
      ↓
terraform plan
      ↓
Review / Approval
      ↓
terraform apply
      ↓
Drift Detection
      ↓
Notification
```

---

# 6. Future Work — Terraform CI/CD

The infrastructure repository could be extended with a dedicated GitHub Actions workflow.

A mature workflow could contain:

```text
Pull Request
     ↓
terraform fmt
     ↓
terraform validate
     ↓
terraform plan
     ↓
Plan Review
     ↓
Manual Approval
     ↓
terraform apply
```

The workflow could also provide controlled teardown:

```text
Destroy Request
      ↓
Approval
      ↓
terraform destroy
```

### Expected Benefits

- consistent infrastructure execution
- repeatable infrastructure changes
- reviewable Terraform plans
- reduced manual infrastructure operations
- clearer infrastructure audit history

---

# 7. Limitation — Infrastructure Drift Handling

GitOps introduces desired-state management, but infrastructure and Kubernetes environments can still experience drift.

The conceptual model is:

```text
Desired State
      │
      ▼
Git / Terraform
      │
      │ compare
      ▼
Actual AWS State
```

A mature platform should continuously detect differences between these states.

---

# 8. Future Work — Terraform Drift Detection

A future infrastructure pipeline could periodically perform drift detection.

Conceptually:

```text
Terraform State
      +
AWS Actual State
      │
      ▼
Drift Detection
      │
      ├── No Drift
      │
      └── Drift Detected
              ↓
          Notification
              ↓
        Human Investigation
```

The important principle is:

> Detect infrastructure drift rather than allowing manual changes to remain invisible.

---

# 9. Limitation — Notification Coverage

The project can validate pipeline results through GitHub Actions, Argo CD, Kubernetes, and other interfaces.

However, a mature operational platform should actively notify engineers when important events occur.

Examples include:

```text
Pipeline Failure
      ↓
Notification

Quality Gate Failure
      ↓
Notification

Terraform Drift
      ↓
Notification

Deployment Failure
      ↓
Notification

Application Health Problem
      ↓
Notification
```

---

# 10. Future Work — Pipeline Notifications

Slack can be extended as a notification channel for:

- application CI failures
- application deployment failures
- Terraform failures
- Terraform drift
- important deployment events
- infrastructure changes

The goal is to reduce the amount of manual polling required to discover failures.

---

# 11. Limitation — Secrets Management

The current implementation uses repository secrets and runtime configuration to avoid placing sensitive credentials directly in source control.

However, this should not be treated as the final enterprise secrets architecture.

The project currently does not establish comprehensive enterprise-grade secrets management.

Sensitive information can include:

```text
AWS credentials
GitHub tokens
SonarQube tokens
SSH private keys
Application secrets
Certificates / private keys
Webhook credentials
```

---

# 12. Future Work — Production Secrets Management

A more mature platform should introduce stronger controls around:

- secret storage
- secret rotation
- access control
- least privilege
- secret lifecycle
- separation between environments
- auditing of secret access

The goal would be to reduce the number of long-lived credentials and improve the overall security boundary.

---

# 13. Limitation — Credential-Based CI Authentication

The demonstrated application pipeline uses AWS credentials supplied through GitHub Actions secrets.

This is functional for the project, but a production-oriented implementation should evaluate stronger workload identity approaches.

The future objective is:

```text
GitHub Actions
      ↓
Short-lived / controlled identity
      ↓
AWS
```

rather than depending unnecessarily on long-lived credentials.

The exact identity architecture should be selected based on the organization's security model and operational requirements.

---

# 14. Limitation — Single-Environment Focus

The project demonstrates the GitOps deployment workflow primarily around a single EKS environment.

A production delivery platform normally needs environment promotion.

For example:

```text
Development
    ↓
Testing
    ↓
Staging
    ↓
Production
```

Each environment may have different:

- resource sizes
- networking
- scaling requirements
- secrets
- domains
- approval requirements

The important GitOps principle is configuration parity rather than identical infrastructure scale.

---

# 15. Future Work — Multi-Environment Promotion

The deployment architecture could evolve toward:

```text
Application CI
      ↓
Image
      ↓
Development
      ↓
Validation
      ↓
Staging
      ↓
Approval
      ↓
Production
```

Possible Git structures include:

```text
Environment-specific values
```

or:

```text
Separate environment overlays
```

or:

```text
Separate environment repositories
```

The final choice should depend on organizational scale and deployment governance.

The key requirement is that promotion remains Git-driven and auditable.

---

# 16. Limitation — Advanced Deployment Strategies

The current project demonstrates Kubernetes workload updates through the normal Deployment rollout mechanism.

It does not establish advanced production deployment strategies such as:

- blue/green deployment
- canary deployment
- progressive delivery
- traffic splitting
- automated rollback based on application metrics

These should not be claimed as implemented capabilities.

---

# 17. Future Work — Progressive Delivery

A future version could introduce:

```text
New Version
     ↓
Small Percentage of Traffic
     ↓
Observe Metrics
     ↓
Healthy?
   /     \
 Yes      No
  ↓        ↓
Increase   Rollback
Traffic
```

Possible strategies include:

```text
Blue / Green
Canary
Progressive Rollout
```

The important principle would be to make deployment risk proportional to the maturity of the validation and observability system.

---

# 18. Limitation — Rollback Maturity

GitOps provides a strong audit trail because deployment state is represented in Git.

However, a production rollback strategy requires more than simply knowing that a previous commit exists.

A mature rollback process should define:

```text
Failure
  ↓
Detection
  ↓
Decision
  ↓
Rollback
  ↓
Verification
  ↓
Incident Review
```

The current project does not establish a comprehensive automated rollback system.

---

# 19. Future Work — Automated Rollback

A future implementation could combine:

```text
Git
+
Argo CD
+
Kubernetes Health
+
Observability
```

to support safer rollback decisions.

For example:

```text
Deployment
    ↓
Health Metrics
    ↓
Failure Detected
    ↓
Rollback Decision
    ↓
Previous Known-Good Version
    ↓
Argo CD
    ↓
Kubernetes
```

Rollback policy should remain explicit and controlled rather than relying on assumptions about application health.

---

# 20. Limitation — Observability

The current project validates deployment state and application availability.

It does not establish a complete production observability platform.

A mature system should provide visibility into:

```text
Logs
Metrics
Traces
```

and connect those signals to:

```text
Alerts
   ↓
Incident Response
   ↓
Root Cause Analysis
```

---

# 21. Future Work — Production Observability

A future observability architecture could be:

```text
Application
    │
    ├── Logs
    ├── Metrics
    └── Traces
            │
            ▼
      Observability
            │
            ▼
          Alerts
            │
            ▼
      Incident Response
```

This extends the current delivery loop:

```text
Code
 ↓
Build
 ↓
Deploy
 ↓
Observe
 ↓
Detect
 ↓
Respond
 ↓
Improve
```

This would move the platform beyond deployment automation into operational reliability.

---

# 22. Limitation — Security Hardening

The project demonstrates AWS IAM, Kubernetes service-account integration, private ECR access, HTTPS, and other security-related mechanisms.

However, this does not constitute complete enterprise security hardening.

Areas requiring deeper treatment include:

- least-privilege access
- credential lifecycle
- secrets management
- network segmentation
- Kubernetes RBAC
- container security
- dependency security
- image scanning
- policy enforcement
- audit controls

---

# 23. Future Work — Security Maturity

A more mature platform could introduce:

```text
Source Security
      ↓
Dependency Security
      ↓
Container Image Security
      ↓
Infrastructure Security
      ↓
Kubernetes Policy
      ↓
Runtime Security
```

Security should become a continuous pipeline property rather than a final manual check.

---

# 24. Limitation — Disaster Recovery

The architecture is reproducible because infrastructure and deployment configuration are represented as code.

However, reproducibility alone is not the same as a fully tested disaster-recovery strategy.

A production disaster-recovery capability requires explicit validation of:

```text
Failure
  ↓
Recovery Procedure
  ↓
Rebuild Infrastructure
  ↓
Restore Required Data
  ↓
Restore Application
  ↓
Validate Service
```

The current project does not establish an enterprise disaster-recovery program.

---

# 25. Future Work — Disaster Recovery Validation

A future iteration could test:

```text
Environment Lost
      ↓
Terraform Rebuild
      ↓
EKS Recreated
      ↓
Controllers Installed
      ↓
Argo CD Restored
      ↓
Application Reconciled
      ↓
Data Restored
      ↓
Application Validated
```

The important future capability is not simply having Terraform code.

It is proving that the environment can actually be reconstructed within an acceptable recovery objective.

---

# 26. Limitation — Multi-Cluster GitOps

The current project focuses on one EKS environment.

It does not demonstrate:

```text
Cluster A
Cluster B
Cluster C
```

managed through a centralized multi-cluster GitOps strategy.

---

# 27. Future Work — Multi-Cluster GitOps

A mature platform could evolve toward:

```text
                 Git
                  │
                  ▼
              Argo CD
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
     Dev EKS   Stage EKS   Prod EKS
```

This would introduce additional considerations around:

- cluster registration
- environment isolation
- promotion
- access control
- secrets
- policy
- failure domains

---

# 28. Limitation — Enterprise Policy Enforcement

The current Argo CD AppProject provides deployment boundaries.

However, a production platform may require stronger policy enforcement.

Examples include:

```text
Allowed Images
Required Labels
Resource Limits
Security Context
Approved Registries
Namespace Rules
Network Policies
```

---

# 29. Future Work — Policy as Code

A future platform could introduce policy enforcement into the delivery path:

```text
Git Change
     ↓
CI
     ↓
Policy Validation
     ↓
Quality Gate
     ↓
Argo CD
     ↓
Kubernetes
```

This would prevent configurations that violate organizational standards from reaching the cluster.

---

# 30. Limitation — Production Scale

The project is a hands-on learning and portfolio implementation.

Its architecture demonstrates transferable DevOps patterns, but the environment should not automatically be interpreted as being sized or hardened for large-scale production workloads.

Important production concerns include:

- capacity planning
- autoscaling
- workload resource management
- cluster scaling
- cost management
- failure-domain design
- operational ownership

The project focuses on demonstrating the engineering workflow rather than establishing enterprise-scale capacity.

---

# 31. Future Work — Platform Scalability

A future iteration could evaluate:

```text
Application Load
      ↓
Horizontal Pod Autoscaling
      ↓
Cluster Capacity
      ↓
Node Scaling
      ↓
Multi-AZ Resilience
      ↓
Cost / Performance Optimization
```

The goal would be to connect deployment automation with actual operational requirements.

---

# 32. Limitation — Production Change Governance

The current project demonstrates automated deployment through GitOps.

However, production organizations may require stronger change-management controls.

A mature workflow could introduce:

```text
Code Change
    ↓
CI
    ↓
Quality Gate
    ↓
Deployment PR
    ↓
Human Review
    ↓
Approval
    ↓
Argo CD
```

This maintains the GitOps model while introducing an explicit governance boundary.

---

# 33. Future Work — Controlled Production Promotion

The future production model should make the promotion path explicit:

```text
Build Once
    ↓
Store Immutable Artifact
    ↓
Promote Through Environments
    ↓
Production Approval
    ↓
Deploy
```

The objective is to avoid rebuilding an application differently for each environment.

Instead, the same validated artifact should progress through controlled deployment states.

---

# 34. Current Project vs Future Platform

| Area | Current Project | Future Direction |
|---|---|---|
| Application CI | GitHub Actions | Continue / harden |
| Quality | SonarQube + Checkstyle | Expand security quality gates |
| Container Registry | Amazon ECR | Image scanning / governance |
| Deployment Config | Helm | PR-based promotion |
| GitOps | Argo CD | Multi-environment / multi-cluster |
| Infrastructure | Terraform | Terraform CI/CD + approval |
| Drift | GitOps reconciliation | Dedicated infrastructure drift detection |
| Notifications | Basic integration | Broader operational notifications |
| Secrets | Repository/runtime secrets | Production-grade secrets management |
| Deployment | Kubernetes rolling update | Progressive delivery |
| Rollback | Git-driven recovery | Automated / policy-driven rollback |
| Observability | Basic validation | Logs + metrics + traces + alerts |
| Security | IAM / HTTPS / access controls | Comprehensive security hardening |
| Recovery | Reproducible infrastructure | Tested disaster recovery |
| Environments | Primary environment | Dev → Stage → Production |
| Scale | Learning/demo environment | Production capacity engineering |

---

# 35. What This Project Should Not Claim

The repository should not claim the following as completed capabilities unless they are genuinely implemented later:

- production-grade high availability
- enterprise disaster recovery
- multi-cluster GitOps
- comprehensive secrets management
- blue/green deployment
- canary deployment
- automated rollback
- complete observability
- enterprise policy enforcement
- fully automated production promotion
- production-grade security hardening
- zero-downtime guarantees under all failure conditions
- enterprise-scale capacity management

These represent future capabilities or broader engineering concerns.

The distinction is important for both technical accuracy and interview credibility.

---

# 36. Interview Boundary

The project can confidently be described as:

> A hands-on GitOps implementation that takes an existing application through GitHub Actions CI, quality validation, container image creation, Amazon ECR publication, Helm-based deployment configuration, Argo CD reconciliation, and Kubernetes deployment on Amazon EKS.

A strong explanation should also acknowledge the maturity boundary:

> The implementation demonstrates the GitOps delivery pattern end to end. A production evolution would add stronger deployment approvals, Terraform CI/CD governance, infrastructure drift detection, production-grade secrets management, observability, security hardening, multi-environment promotion, and advanced deployment strategies.

This demonstrates engineering judgment rather than overstating the maturity of the project.

---

# 37. Future Maturity Roadmap

The platform can evolve in stages.

```text
CURRENT
  │
  ▼
End-to-End GitOps
  │
  ▼
PR-Based Deployment Promotion
  │
  ▼
Terraform CI/CD + Approval
  │
  ▼
Drift Detection + Notifications
  │
  ▼
Production Secrets Management
  │
  ▼
Observability
  │
  ▼
Security / Policy Enforcement
  │
  ▼
Multi-Environment Promotion
  │
  ▼
Advanced Deployment Strategies
  │
  ▼
Multi-Cluster GitOps
  │
  ▼
Production Platform Maturity
```

The progression should be driven by real engineering limitations rather than by simply adding more tools.

---

# 38. Guiding Principle for Future Work

The most important future-work principle is:

> **Do not treat every future technology as a missing feature of the current project. Introduce each capability when it solves a real limitation of the existing architecture.**

For example:

```text
Need safer deployment approval
        ↓
PR-based Helm promotion

Need controlled infrastructure changes
        ↓
Terraform CI/CD + approval

Need visibility into infrastructure drift
        ↓
Drift detection

Need faster incident awareness
        ↓
Notifications

Need operational visibility
        ↓
Observability

Need safer production releases
        ↓
Progressive deployment / rollback
```

This keeps the platform evolution deliberate.

---

# 39. Final Perspective

The project should be viewed as a foundation for progressively more mature DevOps engineering.

```text
                    CURRENT PROJECT
                           │
                           ▼
                   GitOps Delivery Core
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Quality       Artifacts      GitOps
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                         AWS
                           │
                           ▼
                          EKS
                           │
                           ▼
                     Running Workload
```

The next engineering progression is:

```text
Current GitOps
      ↓
Controlled Promotion
      ↓
Infrastructure Automation
      ↓
Drift Detection
      ↓
Security
      ↓
Observability
      ↓
Rollback
      ↓
Multi-Environment Delivery
      ↓
Advanced Deployment
      ↓
Multi-Cluster GitOps
```

The objective is not to accumulate tools.

The objective is to progressively improve:

```text
Safety
Auditability
Reproducibility
Security
Observability
Reliability
Scalability
```

---

# 40. Final Limitation Summary

The current project successfully demonstrates the core GitOps pattern:

```text
Git
 ↓
CI
 ↓
Image
 ↓
Deployment State
 ↓
Argo CD
 ↓
EKS
 ↓
Application
```

Its main limitations are around **production maturity**, not the fundamental GitOps workflow.

The highest-value next improvements are:

1. PR-based Helm promotion
2. Terraform CI/CD with controlled approval
3. Infrastructure drift detection
4. Broader pipeline notifications
5. Production-grade secrets management
6. Stronger security controls
7. Observability
8. Multi-environment promotion
9. Rollback and progressive delivery
10. Disaster recovery validation
11. Multi-cluster GitOps
12. Enterprise policy enforcement

These future capabilities should be introduced progressively as the platform's operational requirements become more demanding.
