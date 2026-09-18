# DevSecOps CI/CD Project — Context for Claude Code

## What this project is

A production-style DevSecOps CI/CD pipeline, built for portfolio/interview demonstration (LinkedIn, Medium, technical interviews). Full flow:

Developer → GitHub Repo → GitHub Actions CI → Review & Merge → Docker Build + Security Checks → Cosign Signing → Amazon ECR → GitOps Repo Update → Argo CD → Self-Managed kubeadm Kubernetes Cluster on EC2 → App Deployment → Prometheus/Grafana/Alertmanager + ELK → Slack/SNS Notifications

## Application

- Using Google's **Online Boutique** (`microservices-demo`) — 12 real microservices, multiple languages.
- **Decision already made:** keep Google's original Dockerfiles as-is. This project's value is the DevOps/security layer around the app, not app development — do not modify application source code or Dockerfiles unless explicitly asked.

## Current status (update this section as work progresses)

- [ ] Phase 1: Discovery and design
- [ ] Phase 2: Application containerization
- [ ] Phase 3: CI pipeline (GitHub Actions)
- [ ] Phase 4: AWS infrastructure (Terraform)
- [ ] Phase 5: Kubernetes cluster (kubeadm on EC2)
- [ ] Phase 6: GitOps and CD (Argo CD)
- [ ] Phase 7: Observability (Prometheus/Grafana/Alertmanager/ELK)
- [ ] Phase 8: Documentation

**Last session left off at:** _(update this line every session — this is the single most important line in this file)_

## Existing infrastructure (do not rebuild from scratch)

- Kubernetes cluster already exists: 1 control plane + 4 worker EC2 instances, kubeadm-bootstrapped, Kubernetes v1.34.x.
- Control plane has a permanent Elastic IP attached (54.85.169.119) — public IP no longer changes on stop/start.
- Helm chart already built at `helm/online-boutique/` covering all 12 services (NetworkPolicy, HPA, Ingress, RBAC, PodDisruptionBudget, ConfigMap, Secret, Namespace) — passes `helm lint`.
- `scripts/build-push.sh` exists to build all 12 images and push to ECR.
- AWS account: `cognativ` profile (read-only MCP access already configured separately in Claude Desktop/Code — not part of this repo).

## Working rules (from the original project brief — follow these strictly)

- Work in phases. Do not implement everything at once.
- Inspect before changing — check current repo/environment state first, especially at the start of a new session.
- Explain every major decision in beginner-friendly language.
- Do not make destructive changes without asking for confirmation first.
- Never expose secrets, tokens, private keys, or credentials — and never commit them to Git.
- Show the exact files that will be created or modified before doing so.
- Explain commands before executing them.
- Validate each phase before moving to the next.
- No long-lived AWS access keys anywhere — use GitHub OIDC federation for GitHub Actions → AWS auth.
- GitHub Actions must not deploy directly via unrestricted kubectl — always go through the GitOps flow (GitHub Actions → GitOps repo → Argo CD → Kubernetes).
- Excluded on purpose (do not add): Amazon RDS PostgreSQL, OpenObserve, Amazon SES, Semgrep/separate SAST stage, Syft/separate SBOM stage.

## Repository structure

This repo is the **application repository** (source, Dockerfile, CI workflows). The Kubernetes manifests/Helm/Argo CD Application definitions live in a **separate GitOps repository** — do not mix the two.

## Full CI pipeline stages (in order)

Checkout → Secret scanning (Gitleaks/TruffleHog) → Dependency scanning → Lint/format → Unit tests → SonarQube → Hadolint → Trivy (filesystem/config) → kubeval/kubeconform → Kyverno/OPA policy validation → Docker build → Trivy (image) → Cosign (image signing) → PR review/approval.

## Full CD pipeline stages (in order, on merge to main)

Build prod image → tag with Git SHA → Trivy scan → Cosign sign → GitHub OIDC auth to AWS → push to ECR → update image tag in GitOps repo → Argo CD detects change → syncs to cluster → rolling update → verify health → notify (Slack/SNS) → rollback via Git revert or Argo CD rollback if failed.
