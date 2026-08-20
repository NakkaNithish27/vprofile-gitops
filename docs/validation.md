# Validation

[← Back to README](../README.md) | [Architecture](architecture.md) | [Implementation](implementation.md) | [Limitations & Future Work](limitations-and-future-work.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/45bf4daf-13e2-46b8-85dc-5265d6ad9bcd" />


## 1. Validation Overview

Validation proves that the VProfile GitOps platform works as an integrated delivery system rather than merely proving that individual commands completed successfully.

The primary validation objective is:

```text
Source Change
     ↓
CI Validation
     ↓
Quality Gate
     ↓
Container Image
     ↓
Amazon ECR
     ↓
Helm Deployment State
     ↓
Argo CD
     ↓
Amazon EKS
     ↓
Kubernetes Rolling Update
     ↓
Running Application
```

The project is considered successfully validated when the important state transitions in this chain can be observed and verified.

The validation therefore focuses on:

1. Application CI
2. SonarQube quality validation
3. Container image publication
4. Helm deployment-state update
5. Argo CD synchronization
6. Kubernetes workload update
7. Application availability
8. End-to-end GitOps behavior

---

## 2. Validation Philosophy

A successful command is not automatically proof that the system works.

For example:

```text
Docker build succeeded
        ≠
Application deployed successfully
```

Similarly:

```text
Argo CD installed
        ≠
Argo CD successfully deploying the application
```

The validation model therefore uses multiple levels of evidence:

```text
Component State
      +
Configuration State
      +
Integration State
      +
Runtime State
      =
End-to-End Validation
```

---

## 3. Validation Environment

The validation environment consists of the following major components:

| Component | Validation Role |
|---|---|
| GitHub | Source and deployment-state history |
| GitHub Actions | CI/CD execution |
| SonarQube | Code-quality validation |
| Docker | Container image creation |
| Amazon ECR | Image storage |
| `vprofile-helm` | Desired deployment state |
| Argo CD | GitOps reconciliation |
| Amazon EKS | Kubernetes runtime |
| AWS Load Balancer Controller | External application routing |
| ALB | HTTP/HTTPS traffic entry point |
| DNS | Application hostname resolution |
| VProfile | Final application workload |

---

# 4. End-to-End Acceptance Criteria

The complete project passes its primary acceptance test when the following sequence succeeds:

```text
Feature Branch
      ↓
Pull Request
      ↓
GitHub Actions
      ↓
Build / Test / Checkstyle
      ↓
SonarQube Analysis
      ↓
Quality Gate PASS
      ↓
Merge to main
      ↓
Docker Image Build
      ↓
Amazon ECR
      ↓
Helm values.yaml image-tag update
      ↓
Argo CD detects Git change
      ↓
Argo CD synchronization
      ↓
Kubernetes Deployment update
      ↓
New Pod created
      ↓
New Pod becomes healthy
      ↓
Old Pod replaced
      ↓
Application remains accessible
```

This is the highest-value validation path for the project.

---

# 5. Validation Phase 1 — Pull Request CI

## Objective

Verify that a source-code change passes the CI quality checks before it is merged.

The expected flow is:

```text
Feature Branch
      ↓
Pull Request
      ↓
GitHub Actions
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
```

## Expected Result

The GitHub Actions workflow completes successfully.

The SonarQube quality gate must pass.

The pull request becomes eligible for merge according to the configured repository rules.

---

## What This Proves

A successful PR validation demonstrates that:

- GitHub Actions can access the application repository.
- The application can be built.
- Tests can execute.
- Checkstyle can execute.
- SonarQube analysis can execute.
- The SonarQube quality gate can return successfully.
- The PR validation stage is functioning.

This is the **quality-control boundary** before the application becomes a deployable artifact.

---

# 6. Validation Phase 2 — Merge to Main

## Objective

Verify that merging a validated change into `main` starts the application build-and-delivery workflow.

The expected sequence is:

```text
Pull Request
      ↓
Quality Gate PASS
      ↓
Merge
      ↓
main branch
      ↓
GitHub Actions
```

The merge must trigger the build/delivery phase.

---

## Expected Result

The main-branch workflow starts automatically.

The workflow should proceed to:

```text
Docker Build
      ↓
Amazon ECR
      ↓
Helm Repository Update
```

---

# 7. Validation Phase 3 — Docker Image Build

## Objective

Verify that the application source is converted into a deployable container image.

The expected flow is:

```text
vprofile-app
      ↓
Dockerfile
      ↓
GitHub Actions
      ↓
Docker Build
      ↓
Container Image
```

## Expected Result

The Docker image is built successfully.

The image receives the expected tag.

---

## What This Proves

A successful image build proves that the CI system can transform the application source into the container artifact required by the Kubernetes deployment.

The image is the artifact that connects:

```text
Application CI
        ↓
Kubernetes Deployment
```

---

# 8. Validation Phase 4 — Amazon ECR

## Objective

Verify that the newly built container image is published to Amazon ECR.

The expected flow is:

```text
Docker Image
      ↓
GitHub Actions
      ↓
Amazon ECR
```

## Verification

Check the ECR repository and confirm that the newly generated image tag exists.

Conceptually:

```text
Amazon ECR
└── VProfile application
    ├── previous tag
    └── new tag
```

---

## Expected Result

The new image is visible in the ECR repository.

The image tag must correspond to the image version produced by the pipeline.

---

## What This Proves

This validates:

- AWS authentication from GitHub Actions
- ECR repository access
- Docker image publication
- image availability for EKS

It also establishes the artifact that Kubernetes will later consume.

---

# 9. Validation Phase 5 — Helm Deployment-State Update

## Objective

Verify that the application pipeline updates the Git-managed deployment configuration.

The expected change is:

```text
vprofile-helm
      ↓
helm/vprofile/values.yaml
      ↓
app image tag
```

Conceptually:

```text
Before:

app:
  tag: old-version
```

After:

```text
app:
  tag: new-version
```

---

## Verification

Inspect the `vprofile-helm` repository and confirm:

```text
helm/vprofile/values.yaml
```

contains the new image tag.

The Git history should also show the deployment-state update.

---

## What This Proves

This is one of the most important validation points in the entire project.

It proves that:

```text
Application CI
      ↓
Container Image
      ↓
Git Deployment State
```

has successfully occurred.

The pipeline does not need to directly execute a Kubernetes deployment command.

Instead, it changes the desired state in Git.

---

# 10. Validation Phase 6 — Argo CD Repository Detection

## Objective

Verify that Argo CD can see the deployment-state change in the Helm repository.

The expected flow is:

```text
vprofile-helm
      ↓
Git change
      ↓
Argo CD detects change
```

---

## Expected Result

Argo CD identifies that the desired deployment state differs from the currently deployed state.

The Application should transition toward synchronization.

Conceptually:

```text
Git State
    │
    │ new image tag
    ▼
Argo CD
    │
    │ detects difference
    ▼
OutOfSync
    │
    ▼
Synchronization
```

If automated synchronization is enabled, the transition occurs without a manual deployment command.

---

# 11. Validation Phase 7 — Argo CD Synchronization

## Objective

Verify that Argo CD reconciles the Git-defined Helm state into Kubernetes.

The expected flow is:

```text
Git
 ↓
Helm
 ↓
Argo CD
 ↓
Kubernetes API
 ↓
EKS
```

## Expected Result

The Argo CD Application reaches:

```text
Synced
Healthy
```

---

## What This Proves

A successful synchronized/healthy state demonstrates that:

- Argo CD can access the Helm repository.
- The Application points to the correct repository.
- The Helm path is correct.
- Helm rendering succeeds.
- The destination cluster is reachable.
- Kubernetes resources can be applied.
- The desired state has been reconciled.

This is the primary GitOps validation point.

---

# 12. Validation Phase 8 — Kubernetes Deployment Update

## Objective

Verify that the Kubernetes Deployment reflects the new image version.

The expected sequence is:

```text
New Helm image tag
      ↓
Argo CD
      ↓
Kubernetes Deployment
      ↓
New Pod
```

---

## Verification

Check the application namespace:

```bash
kubectl get pods -n vprofile
```

Inspect the Deployment:

```bash
kubectl get deployment -n vprofile
```

Inspect the deployed image:

```bash
kubectl describe pod <pod-name> -n vprofile
```

The new Pod should reference the expected ECR image and tag.

---

# 13. Validation Phase 9 — Rolling Update

## Objective

Verify that Kubernetes performs the expected workload update after the image changes.

The demonstrated rollout follows:

```text
Old Pod
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

---

## Expected Result

The new Pod becomes healthy.

The previous Pod is eventually replaced.

The Deployment reaches its expected ready state.

---

## What This Proves

This validates the runtime portion of the GitOps chain:

```text
Git desired state
      ↓
Argo CD
      ↓
Kubernetes Deployment
      ↓
Pod rollout
```

The image tag therefore acts as the deployment trigger.

---

# 14. Validation Phase 10 — Application Availability

## Objective

Verify that the application remains accessible after the deployment.

The external traffic path is:

```text
Browser
   ↓
Application Domain
   ↓
DNS
   ↓
AWS Application Load Balancer
   ↓
Kubernetes Ingress
   ↓
Application Service
   ↓
Application Pod
```

---

## Expected Result

The VProfile application loads through its configured external endpoint.

The application should remain available after the new Pod is deployed.

---

## What This Proves

Application-level validation demonstrates that the deployment is not merely syntactically valid.

It demonstrates that the resulting runtime system is reachable through the intended network path.

---

# 15. Application Dependency Validation

The VProfile workload depends on multiple backend services.

The validation should therefore distinguish:

```text
Pod Running
```

from:

```text
Application Functioning
```

The backend dependencies must be reachable by the application.

The important application-level dependency chain is:

```text
VProfile Application
      │
      ├── Database
      ├── RabbitMQ
      └── Memcached
```

The application should be tested through functionality that exercises the required backend services.

---

# 16. Final End-to-End Validation

The final acceptance test connects the entire system:

```text
Developer
    │
    ▼
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
    └── SonarQube
            │
            ▼
      Quality Gate PASS
            │
            ▼
          Merge
            │
            ▼
       Docker Build
            │
            ▼
         Amazon ECR
            │
            ▼
    Helm values.yaml
       image tag
            │
            ▼
         Argo CD
            │
            ▼
       Kubernetes
            │
            ▼
        New Pod
            │
            ▼
      Running Application
```

This is the primary project acceptance criterion.

---

# 17. What Gets Updated Where

A useful way to validate the pipeline is to inspect each system after a successful deployment.

| System | Expected Change |
|---|---|
| GitHub application repository | New application commit |
| GitHub Actions | Successful workflow |
| SonarQube | Successful analysis / quality gate |
| Docker | New image built |
| Amazon ECR | New image tag |
| `vprofile-helm` | `values.yaml` image tag updated |
| Argo CD | Application synchronized |
| EKS | New Pod using new image |
| Kubernetes Deployment | New image reference |
| Application | New version available |

The expected state relationship is:

```text
ECR
  ↓
new Docker image

vprofile-helm
  ↓
new image tag

EKS
  ↓
new Pod

Argo CD
  ↓
Synced / Healthy
```

These states should agree.

---

# 18. Two-Repository GitOps Validation

The project uses a two-repository application/deployment model:

```text
vprofile-app
      │
      │ application source
      ▼
GitHub Actions
      │
      │ builds image
      ▼
Amazon ECR
      │
      │ deployment version
      ▼
vprofile-helm
      │
      │ desired state
      ▼
Argo CD
      │
      ▼
Amazon EKS
```

Argo CD watches the Helm repository rather than the application repository.

The validation therefore needs to confirm the bridge:

```text
vprofile-app
      ↓
GitHub Actions
      ↓
vprofile-helm
```

and then:

```text
vprofile-helm
      ↓
Argo CD
      ↓
EKS
```

---

# 19. PR-Gated Quality Validation

The pipeline deliberately separates:

```text
PR Phase
    ↓
Quality

Merge Phase
    ↓
Artifact + Deployment State
```

The validation model is therefore:

```text
Pull Request
      ↓
Build / Test / Quality
      ↓
Quality Gate PASS
      ↓
Merge
      ↓
Docker Image
      ↓
ECR
      ↓
Helm Update
```

This prevents the merge-to-main delivery phase from being the first point at which code quality is evaluated.

---

# 20. Manual Synchronization Validation

Although automated synchronization is enabled, Argo CD can also be used to inspect or trigger synchronization manually.

The validation purpose is to distinguish:

```text
Argo CD detects desired-state change
```

from:

```text
Argo CD successfully applies desired state
```

The Application should ultimately reach:

```text
Synced
Healthy
```

regardless of whether synchronization is observed automatically or initiated manually during troubleshooting.

---

# 21. GitOps Reconciliation Validation

GitOps is not validated merely by seeing an Argo CD dashboard.

The meaningful validation is:

```text
Git Desired State
        │
        ▼
Argo CD
        │
        ▼
Kubernetes Actual State
```

A successful reconciliation means the Kubernetes deployment reflects the Git-defined Helm state.

With automated self-healing enabled, a further validation scenario is possible:

```text
Git
 ↓
Desired State

Manual Kubernetes Change
 ↓
Drift

Argo CD
 ↓
Reconciliation
 ↓
Desired State Restored
```

This demonstrates the reconciliation model rather than only the initial deployment.

---

# 22. Troubleshooting Decision Tree

When the application does not update:

```text
Application did not update
        │
        ▼
Check GitHub Actions
        │
        ├── FAIL → Fix CI
        │
        ▼
Check ECR
        │
        ├── Missing image → Fix image publication
        │
        ▼
Check Helm repository
        │
        ├── Old tag → Fix Helm update
        │
        ▼
Check Argo CD
        │
        ├── Repository error → Fix Git access
        │
        ├── Path error → Fix Helm path
        │
        ├── OutOfSync → Inspect sync
        │
        ▼
Check Kubernetes
        │
        ├── ImagePullBackOff → Check ECR permissions
        │
        ├── CrashLoopBackOff → Check workload configuration
        │
        ├── Pending → Check cluster/resources
        │
        ▼
Check Ingress / ALB
        │
        ├── No endpoint → Check controller / ingress
        │
        ▼
Check Application
```

---

# 23. Common Failure Conditions

## GitHub Actions Failure

Check:

- workflow syntax
- repository permissions
- GitHub Secrets
- GitHub Variables
- SonarQube connectivity
- AWS credentials
- Docker build

---

## SonarQube Quality Gate Failure

Check:

- SonarQube server availability
- project configuration
- authentication token
- analysis output
- quality-gate result

A failed quality gate should stop the PR from being treated as successfully validated.

---

## ECR Image Missing

Check:

- AWS credentials
- AWS region
- ECR repository name
- Docker login
- image tag
- workflow execution logs

---

## Helm Tag Not Updated

Check:

- cross-repository authentication
- GitHub PAT
- Helm repository URL
- repository branch
- `values.yaml` path
- workflow permissions

---

## Argo CD Repository Error

Check:

- repository URL
- SSH key
- repository credentials
- repository permissions
- Argo CD repository registration

---

## Argo CD OutOfSync

Check:

- Helm repository commit
- chart path
- values file
- Helm rendering
- destination cluster
- target namespace
- Kubernetes resource errors

---

## `ImagePullBackOff`

Check:

- ECR image exists
- image repository is correct
- image tag is correct
- EKS node IAM role
- ECR permissions
- AWS region

---

## Pod Does Not Become Ready

Check:

```bash
kubectl get pods -n vprofile
kubectl describe pod <pod-name> -n vprofile
kubectl logs <pod-name> -n vprofile
```

The goal is to identify whether the failure is caused by:

```text
Image
Configuration
Dependency
Application
Resource
```

---

## Ingress / ALB Failure

Check:

- AWS Load Balancer Controller
- controller Pods
- controller IAM permissions
- Ingress class
- Ingress annotations
- VPC/subnet configuration
- DNS
- ACM certificate configuration

---

# 24. Evidence Strategy

Evidence should prove the project's important claims without turning the repository into a collection of screenshots.

The recommended approach is:

```text
Claim
  ↓
Validation Step
  ↓
High-Signal Evidence
```

Only evidence from the personally completed environment should be treated as execution evidence.

Course screenshots, lecture screenshots, or copied examples should not be used as proof of personal execution.

---

# 25. Recommended Evidence Set

## Evidence 1 — GitHub Actions PR Validation

Capture a successful workflow showing:

```text
Build
Test
Checkstyle
SonarQube
Quality Gate
```

### Proves

The PR quality stage executed successfully.

---

## Evidence 2 — ECR Image

Capture the ECR repository showing the newly published image tag.

### Proves

The application image was successfully published.

---

## Evidence 3 — Helm Repository Change

Capture the relevant Git commit or file change showing:

```text
values.yaml
    ↓
new image tag
```

### Proves

CI successfully transferred the deployable version into Git-managed deployment state.

---

## Evidence 4 — Argo CD Application

Capture the Argo CD Application showing:

```text
Synced
Healthy
```

and, where useful, the resource tree.

### Proves

Argo CD successfully reconciled the desired state.

---

## Evidence 5 — Kubernetes Pod

Capture:

```bash
kubectl get pods -n vprofile
```

and, where useful:

```bash
kubectl describe pod <pod-name> -n vprofile
```

### Proves

The new image is running inside EKS.

---

## Evidence 6 — Application

Capture the running VProfile application through its configured external endpoint.

### Proves

The final runtime path is working.

---

# 26. Validation Evidence Matrix

| Claim | Validation | Evidence | Expected Result |
|---|---|---|---|
| PR validation works | GitHub Actions | Workflow screenshot | Successful |
| Quality gate works | SonarQube | Quality gate result | PASS |
| Image build works | GitHub Actions | Build result | Success |
| Image publication works | ECR | Repository/image | New tag exists |
| Helm state changes | Git | Commit/file diff | New tag |
| Argo CD detects change | Argo CD | Application status | Sync triggered |
| GitOps reconciliation works | Argo CD | Application state | Synced / Healthy |
| EKS receives new image | Kubernetes | Pod description | New image tag |
| Rolling update works | Kubernetes | Pod transition | New Pod healthy |
| Application is reachable | Browser / endpoint | Application screenshot | Application loads |
| End-to-end delivery works | Combined evidence | Evidence chain | Complete |

---

# 27. Validation Claim Matrix

| Claim | Supported By |
|---|---|
| GitHub Actions CI works | Successful PR workflow |
| SonarQube quality gate works | Quality gate result |
| Docker image is produced | Successful image build |
| ECR stores the image | ECR repository |
| Helm is the deployment-state bridge | `values.yaml` update |
| Argo CD watches deployment state | Argo CD Application |
| Argo CD reconciles Kubernetes | Synced / Healthy state |
| EKS runs the new image | Kubernetes Pod |
| Kubernetes performs rolling update | New Pod replacing old Pod |
| Application remains accessible | External application test |
| Complete GitOps flow works | Combined end-to-end evidence |

---

# 28. What Validation Does Not Prove

Successful end-to-end validation does **not** prove:

- enterprise production readiness
- complete security hardening
- disaster recovery
- multi-cluster GitOps
- enterprise observability
- blue/green deployment
- canary deployment
- multi-environment promotion
- automated Terraform approval
- comprehensive secrets management
- enterprise-scale platform governance

The validation proves the capabilities actually demonstrated by the project.

It should not be used to infer capabilities that were not implemented.

---

# 29. Validation Boundaries

The project should distinguish between:

```text
Demonstrated
```

and:

```text
Possible Future Capability
```

### Demonstrated

- GitHub Actions CI/CD
- SonarQube quality validation
- Docker image creation
- Amazon ECR publication
- Helm deployment-state update
- Argo CD synchronization
- Kubernetes rollout
- EKS application deployment
- ALB/Ingress application access
- end-to-end pipeline validation

### Not Demonstrated as Completed

- PR-based Helm approval workflow
- Terraform CI/CD with manual apply approval
- enterprise observability
- comprehensive secrets management
- advanced deployment strategies
- multi-environment promotion
- disaster recovery

---

# 30. Final Validation Checklist

Use this checklist when reconstructing or revalidating the project.

### Source and CI

- [ ] Feature branch created
- [ ] Change pushed
- [ ] Pull Request created
- [ ] GitHub Actions triggered
- [ ] Build passed
- [ ] Tests passed
- [ ] Checkstyle passed
- [ ] SonarQube analysis passed
- [ ] Quality Gate passed
- [ ] Pull Request merged

### Container

- [ ] Docker image built
- [ ] Image tagged
- [ ] ECR authentication succeeded
- [ ] Image pushed to ECR
- [ ] New image tag visible in ECR

### Helm / GitOps

- [ ] `values.yaml` updated
- [ ] New image tag committed
- [ ] Helm repository push succeeded
- [ ] Argo CD detected the change
- [ ] Application synchronized
- [ ] Application became Healthy

### Kubernetes

- [ ] Deployment updated
- [ ] New Pod created
- [ ] New Pod became Ready
- [ ] Old Pod replaced
- [ ] New Pod uses expected ECR image

### Networking

- [ ] Ingress exists
- [ ] ALB exists
- [ ] DNS resolves
- [ ] HTTPS endpoint works
- [ ] Application is reachable

### Application

- [ ] VProfile loads
- [ ] Application functionality works
- [ ] Required backend dependencies are reachable
- [ ] Final application state matches the deployed version

---

# 31. Complete Validation Model

The complete project can be remembered as:

```text
                    VALIDATION

Source Change
     │
     ▼
Pull Request
     │
     ▼
CI + Quality Gate
     │
     ▼
Merge
     │
     ▼
Docker Image
     │
     ▼
Amazon ECR
     │
     ▼
Helm values.yaml
     │
     ▼
Git Commit
     │
     ▼
Argo CD
     │
     ▼
Synced / Healthy
     │
     ▼
Kubernetes Deployment
     │
     ▼
New Pod
     │
     ▼
Pod Healthy
     │
     ▼
ALB / Ingress
     │
     ▼
Application
```

---

# 32. Validation Memory Trigger

To reconstruct the validation quickly:

> **Make a small feature-branch change → create PR → verify GitHub Actions and SonarQube quality gate → merge → verify Docker image in ECR → verify new Helm image tag → verify Argo CD detects and synchronizes → verify new EKS Pod → verify rolling update → verify the application through the external endpoint.**

The most important state chain to remember is:

```text
ECR
  ↓
new image

Helm
  ↓
new tag

Argo CD
  ↓
Synced / Healthy

EKS
  ↓
new Pod

Application
  ↓
new version running
```

That chain is the core proof that the GitOps platform works end to end.
