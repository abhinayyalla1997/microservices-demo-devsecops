# DevSecOps CI/CD Project — Context for Claude Code

## What this project is

A production-style DevSecOps CI/CD pipeline, built for portfolio/interview demonstration (LinkedIn, Medium, technical interviews). Full flow:

Developer → GitHub Repo → GitHub Actions CI → Review & Merge → Docker Build + Security Checks → Cosign Signing → Amazon ECR → GitOps Repo Update → Argo CD → Self-Managed kubeadm Kubernetes Cluster on EC2 → App Deployment → Prometheus/Grafana/Alertmanager + ELK → Slack/SNS Notifications

## Architecture diagram

A visual diagram of the full system architecture exists at 
`docs/architecture-diagram.png`. **View this image file directly at the 
start of Phase 1 (and any time the overall architecture needs 
re-confirming)** — it shows the exact box-and-arrow flow, AWS VPC boundary, 
and the color-coded legend (CI / CD-GitOps / Runtime / Security / 
Observability) that the text description above only partially captures.

## Application

- Using Google's **Online Boutique** (`microservices-demo`) — 12 real microservices, multiple languages.
- **Decision already made:** keep Google's original Dockerfiles as-is. This project's value is the DevOps/security layer around the app, not app development — do not modify application source code or Dockerfiles unless explicitly asked.

## Current status (update this section as work progresses)

- [x] Phase 1: Discovery and design — repo created, source imported, ECR/visibility decisions locked in, CI matrix + OIDC design approved
- [ ] Phase 2: Application containerization
- [ ] Phase 3: CI pipeline (GitHub Actions) — design approved (matrix strategy, see below), YAML not yet written
- [ ] Phase 3.5: Plan the separate GitOps repository — adapt the existing Helm chart (already built for a different project, covers all 12 services) into it rather than rebuilding from scratch. Planning only in the next session; do not create the GitOps repo yet. This application repo stays source-only — no Helm charts or Kubernetes manifests here, per the existing repository-structure rule below.
- [ ] Phase 4: AWS infrastructure (Terraform)
- [ ] Phase 5: Kubernetes cluster (kubeadm on EC2)
- [ ] Phase 6: GitOps and CD (Argo CD)
- [ ] Phase 7: Observability (Prometheus/Grafana/Alertmanager/ELK)
- [ ] Phase 8: Documentation

**Last session left off at:** 2026-09-18 — Created public GitHub repo `abhinayyalla1997/microservices-demo-devsecops` and pushed `src/` (12 services, sparse-checked-out from Google's `microservices-demo`, unmodified, no k8s manifests/Skaffold). Locked in ECR strategy (single `online-boutique` repo, `<service>-<sha>` tags) and repo-visibility rule (public, no AWS account IDs or secrets in this repo). Designed and approved (not yet implemented) the GitHub Actions matrix workflow: one job template fanned out per service via `strategy.matrix.service`, each doing build → Trivy scan → Cosign sign (OIDC keyless) → push to the single ECR repo via GitHub OIDC auth to AWS, followed by a single `update-gitops` job after the full matrix completes. **Next session:** (1) start Phase 3 — write `.github/workflows/ci.yml` and `.github/actions/build-scan-sign/action.yml` for real, starting with one service (e.g. `frontend`) before wiring the full 12-way matrix; verify each service's exact Dockerfile path (cartservice's is nested at `src/cartservice/src/Dockerfile`, not `src/cartservice/Dockerfile`). (2) Plan the separate GitOps repo (Phase 3.5 above) and how to adapt the existing Helm chart into it.

## Existing infrastructure (do not rebuild from scratch)

- Kubernetes cluster already exists: 1 control plane + 4 worker EC2 instances, kubeadm-bootstrapped, Kubernetes v1.34.x.
- Control plane has a permanent Elastic IP attached (54.85.169.119) — public IP no longer changes on stop/start.
- Helm chart already built at `helm/online-boutique/` covering all 12 services (NetworkPolicy, HPA, Ingress, RBAC, PodDisruptionBudget, ConfigMap, Secret, Namespace) — passes `helm lint`.
- `scripts/build-push.sh` exists to build all 12 images and push to ECR.
- AWS account: `cognativ` profile (read-only MCP access already configured separately in Claude Desktop/Code — not part of this repo).
- GitHub repo: `abhinayyalla1997/microservices-demo-devsecops` (public), created and pushed 2026-09-18. Contains `src/` (12 services, unmodified from Google's repo) and this `CLAUDE.md`.

## Architecture decisions

- **ECR strategy:** ONE ECR repository named `online-boutique` (region `us-east-1`; account ID intentionally omitted from this public repo — see `Repository structure`/visibility note below) holds all 12 service images. Tags follow `<service-name>-<git-sha>`, e.g. `frontend-a1b2c3d`, `cartservice-a1b2c3d`. Do not create separate ECR repos per service.
- **Repo visibility:** the GitHub repository is **PUBLIC** (portfolio/LinkedIn use). Never commit secrets, credentials, or internal AWS account details to it — double-check before every commit.

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
