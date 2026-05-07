# 🚀 GitOps CI/CD Pipeline — GitHub × AWS

> A production-grade CI/CD pipeline that automatically builds, pushes, and deploys a **FastAPI application** to **Amazon EKS** using a fully automated GitOps workflow — with zero stored AWS credentials.

<img width="5320" height="3965" alt="architecture png" src="https://github.com/user-attachments/assets/e019f319-0847-40d1-a473-42e417261789" />

---

## 📌 Table of Contents

- [Overview](#overview)
- [Why GitHub × AWS Integration Matters](#why-github--aws-integration-matters)
- [Architecture](#architecture)
- [Pipeline Flow](#pipeline-flow)
- [Multi-Environment Strategy](#multi-environment-strategy)
- [Security — Keyless Auth with OIDC](#security--keyless-auth-with-oidc)
- [GitOps Approach](#gitops-approach)
- [Repository Structure](#repository-structure)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)

---

## Overview

This project implements an **end-to-end CI/CD pipeline** for deploying a Python FastAPI application to AWS EKS. The pipeline is fully automated — a single `git push` triggers the entire workflow from code to running pods, across three isolated environments (dev, staging, prod).

Key highlights:
- **Zero stored AWS credentials** — authentication via OpenID Connect (OIDC)
- **GitOps-driven deployments** — ArgoCD watches a dedicated GitOps repo and syncs state automatically
- **Multi-environment isolation** — separate namespaces and Helm value files per environment
- **Cross-account architecture** — CI infrastructure and workloads in separate AWS accounts
- **Helm-based Kubernetes manifests** — templated and version-controlled deployment configuration

---

## Why GitHub × AWS Integration Matters

Integrating GitHub with AWS is more than just connecting two platforms — it is the foundation of a **secure, auditable, and fully automated delivery pipeline**.

### The old way — stored credentials (dangerous)
```
AWS_ACCESS_KEY_ID     = AKIA...  ← stored in GitHub Secrets forever
AWS_SECRET_ACCESS_KEY = xxxx...  ← one leak = full AWS account access
```

Hardcoded or long-lived credentials are a major security risk. They never expire, are difficult to rotate, and if leaked they grant permanent access to your AWS environment.

### The modern way — OIDC (keyless)
```
GitHub runner → requests JWT from GitHub OIDC endpoint
             → sends JWT to AWS STS
             → STS verifies signature against GitHub's public keys
             → issues temporary credentials (valid 1 hour only)
             → credentials exist in runner memory only, never stored
```

With OIDC, **there are no credentials to steal**. Every workflow run gets a fresh, short-lived token tied to a specific repo and branch. This is the industry standard for GitHub × AWS integration.

### Benefits of tight GitHub × AWS integration
| Benefit | Description |
|---|---|
| Security | No long-lived credentials — OIDC tokens expire in 1 hour |
| Auditability | Every deployment is a git commit — full history and rollback |
| Automation | Push to a branch → environment is updated automatically |
| Separation of concerns | Developers own code, DevOps owns infrastructure |
| Scalability | Add new environments by adding a branch and a Helm values file |

---

## Architecture

The pipeline is split across two AWS accounts:

### Management Account (CI)
Handles all build and image storage infrastructure:
- **GitHub Actions** — detects push events, authenticates via OIDC, triggers CodeBuild
- **AWS IAM** — OIDC provider + scoped IAM role for GitHub Actions
- **AWS CodeBuild** — 3 separate projects (dev, staging, prod) that build Docker images
- **AWS ECR** — private image registry with cross-account pull access enabled

### Workloads Account (CD)
Handles all deployment and runtime infrastructure:
- **Amazon EKS** — Kubernetes cluster with managed node groups in private subnets
- **ArgoCD** — GitOps controller running in the `argocd` namespace
- **App namespaces** — `test-cicd-dev`, `test-cicd-staging`, `test-cicd-prod`

---

## Pipeline Flow

```
1.  Developer pushes code to GitHub (feature/*, develop, or main)
2.  GitHub Actions workflow triggers
3.  Runner sends OIDC request to GitHub token endpoint
4.  GitHub issues signed JWT (repo + branch claims)
5.  Runner calls AWS STS → AssumeRoleWithWebIdentity
6.  STS validates JWT → issues temporary credentials (1hr)
7.  GitHub Actions calls CodeBuild StartBuild API
        → passes GIT_SHA, APP_ENV, GITOPS_TOKEN as env vars
8.  CodeBuild runs buildspec.yml:
        → authenticates to ECR Public (no rate limits)
        → docker build
        → docker push to ECR (tagged: env-gitsha)
        → git clone gitops-repo
        → updates k8s/<env>/values.yaml image tag
        → git commit + push [skip ci]
9.  ArgoCD detects change in gitops-repo (webhook or poll)
10. ArgoCD syncs Helm chart to correct EKS namespace
11. EKS node pulls new image from ECR (via node IAM role)
12. Rolling update completes → new pods running
```

---

## Multi-Environment Strategy

Each branch maps to a dedicated environment with its own CodeBuild project, Helm values, Kubernetes namespace, and replica count:

| Branch | Environment | CodeBuild Project | Namespace | Replicas |
|---|---|---|---|---|
| `feature/*` | dev | `test-cicd-dev` | `test-cicd-dev` | 1 |
| `develop` | staging | `test-cicd-staging` | `test-cicd-staging` | 2 |
| `main` | prod | `test-cicd-prod` | `test-cicd-prod` | 3 |

Image tags follow the pattern `{env}-{git-sha}` — for example:
```
dev-9f9fc898b7c5b8c6d417b7fde36ea4b29e5ae8ed
staging-9f9fc898b7c5b8c6d417b7fde36ea4b29e5ae8ed
prod-9f9fc898b7c5b8c6d417b7fde36ea4b29e5ae8ed
```

This means every image is traceable to an exact commit and environment — `latest` is never used.

### Branch protection
- `main` is protected — requires PR from `develop` and passing CI checks
- Direct pushes to `main` are blocked
- All production deployments go through a PR review process

---

## Security — Keyless Auth with OIDC

```
┌─────────────────────────────────────────────────────────────┐
│  No AWS credentials stored anywhere in GitHub               │
│                                                             │
│  GitHub runner                                              │
│    │                                                         │
│    ├─ requests JWT ──► GitHub OIDC endpoint                 │
│    │                     (token.actions.githubusercontent.com)│
│    │                                                         │
│    ├─ sends JWT ────► AWS STS                               │
│    │                     verifies against GitHub public keys │
│    │                                                         │
│    └─ receives ◄──── Temp credentials (1hr, memory only)   │
│                                                             │
│  IAM trust policy scoped to:                                │
│  repo:mr-g00d-cat/test-cicd:*                               │
└─────────────────────────────────────────────────────────────┘
```

The IAM role trust policy ensures only this specific repository can assume the role — even a fork cannot obtain credentials:

```json
{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub":
        "repo:mr-g00d-cat/test-cicd:*"
    }
  }
}
```

---

## GitOps Approach

ArgoCD runs **inside the EKS cluster** and continuously monitors the GitOps repository. The desired state of each environment lives entirely in git — not in any CI system or person's head.

```
gitops-repo (source of truth)
    k8s/dev/values.yaml      ← ArgoCD syncs this to test-cicd-dev namespace
    k8s/staging/values.yaml  ← ArgoCD syncs this to test-cicd-staging namespace
    k8s/prod/values.yaml     ← ArgoCD syncs this to test-cicd-prod namespace
```

**Why GitOps?**
- Every deployment is a git commit — full audit trail
- Rolling back = reverting a commit
- Drift detection — ArgoCD automatically corrects manual `kubectl` changes
- Separation of CI (build) and CD (deploy) concerns

**ArgoCD sync policy:**
```yaml
syncPolicy:
  automated:
    prune: true      # removes resources deleted from git
    selfHeal: true   # reverts unauthorized manual changes
```

---

## Repository Structure

### App repo — `mr-g00d-cat/test-cicd`
```
test-cicd/
├── app/
│   └── main.py                    # FastAPI application
├── requirements.txt               # Python dependencies
├── Dockerfile                     # uses ECR Public base image
├── buildspec.yml                  # CodeBuild instructions
└── .github/
    └── workflows/
        └── ci.yml                 # GitHub Actions — OIDC + trigger
```

### GitOps repo — `mr-g00d-cat/gitops-repo`
```
gitops-repo/
├── argocd/
│   ├── app-dev.yaml               # ArgoCD Application — dev
│   ├── app-staging.yaml           # ArgoCD Application — staging
│   └── app-prod.yaml              # ArgoCD Application — prod
└── k8s/
    ├── dev/
    │   ├── Chart.yaml
    │   ├── values.yaml            # image tag updated by CodeBuild
    │   └── templates/
    │       ├── deployment.yaml
    │       └── service.yaml
    ├── staging/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    │       ├── deployment.yaml
    │       └── service.yaml
    └── prod/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            └── service.yaml
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Application | Python 3.12, FastAPI, Uvicorn |
| Containerisation | Docker (base: ECR Public — no rate limits) |
| Image registry | AWS ECR (private, immutable tags, scan on push) |
| CI trigger | GitHub Actions |
| CI build | AWS CodeBuild (3 projects) |
| Authentication | OIDC — zero stored credentials |
| Image delivery | AWS ECR cross-account pull |
| GitOps controller | ArgoCD (Helm install) |
| Kubernetes | Amazon EKS 1.32 (managed node groups) |
| Infra as code | Terraform (VPC, EKS, IAM, ECR) |
| Package manager | Helm 3 |

---

## Getting Started

### Prerequisites
```bash
# tools required
aws --version        # AWS CLI v2
terraform version    # Terraform 1.5+
kubectl version      # kubectl
helm version         # Helm 3
argocd version       # ArgoCD CLI
```

### 1. Clone both repos
```bash
git clone https://github.com/mr-g00d-cat/test-cicd.git
git clone https://github.com/mr-g00d-cat/gitops-repo.git
```

### 2. Provision AWS infrastructure
```bash
cd terraform
terraform init
terraform apply
```

### 3. Connect kubectl to EKS
```bash
aws eks update-kubeconfig --region ap-northeast-1 --name test-cicd-cluster
kubectl get nodes
```

### 4. Install ArgoCD
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set configs.params."server\.insecure"=true
```

### 5. Add GitHub secrets to `test-cicd` repo
| Secret | Description |
|---|---|
| `AWS_ROLE_ARN` | ARN of the github-actions-role |
| `AWS_REGION` | e.g. `ap-northeast-1` |
| `CODEBUILD_PROJECT_DEV` | CodeBuild project name for dev |
| `CODEBUILD_PROJECT_STAGING` | CodeBuild project name for staging |
| `CODEBUILD_PROJECT_PROD` | CodeBuild project name for prod |
| `GITOPS_TOKEN` | Fine-grained PAT — write access to gitops-repo only |

### 6. Deploy ArgoCD applications
```bash
cd gitops-repo
kubectl apply -f argocd/
```

### 7. Trigger the pipeline
```bash
git checkout -b feature/my-feature
# make changes
git add . && git commit -m "feat: my change"
git push origin feature/my-feature
# watch GitHub Actions → CodeBuild → ArgoCD → EKS
```

---

## Author

**Md Tanvir Rahman** — [Linkedin Profile](https://www.linkedin.com/in/tanvir-rahman-t006/)

---

> *Built to learn production-grade DevOps practices — GitOps, keyless security, multi-environment delivery, and Kubernetes-native deployments on AWS.*
