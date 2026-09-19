# DevSecOps CI/CD Project — Context for Claude Code

## What this project is

A production-style DevSecOps CI/CD pipeline, built for portfolio/interview demonstration (LinkedIn, Medium, technical interviews). Full flow:

Developer → GitHub Repo → GitHub Actions CI → Review & Merge → Docker Build + Security Checks → Cosign Signing → Amazon ECR → Argo CD Image Updater (watches ECR, updates GitOps repo) → Argo CD → Self-Managed kubeadm Kubernetes Cluster on EC2 → App Deployment → Prometheus/Grafana/Alertmanager + OpenObserve (logs) → Slack/SNS Notifications

## Architecture diagram

A visual diagram of the full system architecture exists at 
`docs/architecture_diagram.png` (filename uses an underscore — renamed 
2026-09-19; the old `architecture-diagram.png` no longer exists). **View 
this image file directly at the start of Phase 1 (and any time the overall 
architecture needs re-confirming — it is treated as a live source of truth 
and gets replaced in place when the design changes)** — it shows the exact 
box-and-arrow flow, AWS VPC boundary, and the color-coded legend (CI / 
CD-GitOps / Runtime / Security / Observability) that the text description 
above only partially captures.

## Application

- Using Google's **Online Boutique** (`microservices-demo`) — 12 real microservices, multiple languages.
- **Decision already made:** keep Google's original Dockerfiles as-is. This project's value is the DevOps/security layer around the app, not app development — do not modify application source code or Dockerfiles unless explicitly asked.

## Current status (update this section as work progresses)

- [x] Phase 1: Discovery and design — repo created, source imported, ECR/visibility decisions locked in, CI matrix + OIDC design approved
- [ ] Phase 2: Application containerization
- [ ] Phase 3: CI pipeline (GitHub Actions) — design approved (matrix strategy, see below), YAML not yet written
- [ ] Phase 3.5: Plan the separate GitOps repository — adapt the existing Helm chart (already built for a different project, covers all 12 services) into it rather than rebuilding from scratch. Also plan that repo's own Kyverno-CLI lint pipeline (renders Helm output, lints on PRs touching `helm/**`) and its Argo CD Image Updater config (annotations per service telling it which ECR tag pattern to watch). Planning only; do not create the GitOps repo yet. This application repo stays source-only — no Helm charts or Kubernetes manifests here, per the existing repository-structure rule below.
- [ ] Phase 4: AWS infrastructure (Terraform)
- [ ] Phase 5: Kubernetes cluster (kubeadm on EC2)
- [ ] Phase 6: GitOps and CD (Argo CD + Argo CD Image Updater + Kyverno admission webhook for Cosign signature verification)
- [ ] Phase 7: Observability (Prometheus/Grafana/Alertmanager + OpenObserve/Fluent Bit for logs)
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
- **Logging stack — OpenObserve replaces ELK (confirmed 2026-09-19):** this reverses the original brief's exclusion of OpenObserve; it's a deliberate decision, not an oversight. Remove Elasticsearch/Logstash/Kibana from the design. Add OpenObserve (self-hosted, open-source edition, AGPL-3.0, free, single-binary) for log storage, search, and its own dashboards. Fluent Bit remains the log shipper/collector (unchanged role — only the destination changes, from ELK to OpenObserve). Prometheus, Grafana, and Alertmanager are unaffected — OpenObserve only replaces the logging piece, not metrics/alerting.
- **GitOps repo update mechanism — Argo CD Image Updater, not a PAT (confirmed 2026-09-19):** the app repo's CD pipeline stops after pushing the image to ECR — it never touches the GitOps repo and holds no credential for it. A separate, dedicated **Argo CD Image Updater** controller is installed once during Phase 6 (alongside Argo CD itself) and runs continuously in-cluster. It watches ECR directly for new tags matching each service's pattern (`frontend-*`, `cartservice-*`, etc.) and updates `values.yaml` in the GitOps repo using Argo CD's own already-configured Git credentials. This removes the last long-lived PAT from the pipeline — do not implement the old "CI clones GitOps repo and pushes with a stored PAT" approach.
- **Confirmed exact GitHub Actions to use (2026-09-19):** `gitleaks/gitleaks-action@v3` (not `@v2` — v2 stopped working on GitHub-hosted runners as of 2026-09-16 due to Node 20 runtime deprecation), `aquasecurity/trivy-action@master`, `hadolint/hadolint-action` (official), `sigstore/cosign-installer` (keyless signing via GitHub OIDC), `hermanbanken/kubeconform-action@v1`, the official Dependency-Check GitHub Action for OWASP Dependency-Check. All are self-fetching, ephemeral per-run actions — no pre-installation needed.
- **Cosign signature record:** keyless signing publishes to Sigstore's public Rekor transparency log automatically (searchable at search.sigstore.dev) — this is the permanent record, independent of the CI runner. No separate signature-history storage needed in this project.
- **Per-service Dockerfile paths (verified directly against the repo, 2026-09-19):** 11 of 12 services use `src/<service>/Dockerfile` (flat). `cartservice` is the one exception — `src/cartservice/src/Dockerfile` (nested, because it's a full .NET solution with its own internal `src/` and `tests/` folders). The CI matrix must handle this as a per-service path override, not a single uniform pattern.
- **GHCR PR-time image cleanup:** any throwaway/ephemeral GHCR image built for a PR is deleted on PR close via `actions/delete-package-versions` — these must not accumulate indefinitely.
- **SonarCloud Automatic Analysis:** SonarCloud's GitHub App integration is already auto-scanning this repo out of the box (no CI YAML involved). Once the real CI pipeline's own SonarQube step is added, go to SonarCloud → Administration → Analysis Method and switch OFF Automatic Analysis, to avoid two parallel, slightly-different scan results existing simultaneously.
- **Kyverno has two separate roles — do not conflate them:** (1) Kyverno CLI, used to lint the GitOps repo's rendered Helm output, in that repo's own separate CI pipeline (triggered by PRs touching `helm/**`) — belongs to the GitOps repo, doesn't exist yet, not part of this app repo. (2) Kyverno as an admission webhook, a permanent Helm install into the live Kubernetes cluster enforcing Cosign signature verification on every Pod creation — this is Phase 6 work, installed only after Argo CD and the GitOps repo exist. Do not add Kyverno to this app repo's CI.
- **GitHub OIDC provider + IAM role for CD (confirmed 2026-09-19):** created manually via the AWS Console (not Terraform — Terraform/Phase 4 hasn't happened yet; this role can be imported into Terraform state later without changing its behavior). Role ARN: `arn:aws:iam::566279697030:role/github-actions-cd-frontend`. The account ID is included here deliberately, not redacted like the earlier ECR note — this ARN must appear in full in `cd.yml` for OIDC federation to actually work, so hiding it in CLAUDE.md would provide no real protection once the workflow file is public. What actually secures this role is its trust policy, not obscurity of the ARN. Permissions are scoped to ECR push on the single `online-boutique` repository only, nothing else. The role name (`...-cd-frontend`) is service-specific for this session's frontend-only scope; when the CD matrix expands to all 12 services, decide then whether to rename/broaden this one role or create per-service roles.
- **OIDC trust policy `sub` claim gotcha (discovered 2026-09-19, cost ~2 failed CD runs to diagnose):** GitHub's OIDC token `sub` claim for this repo is **not** the plain `repo:<owner>/<repo>:ref:refs/heads/main` form — it includes stable numeric IDs appended to both owner and repo: `repo:abhinayyalla1997@127104532/microservices-demo-devsecops@1375690658:ref:refs/heads/main` (`127104532` = the GitHub account's user ID, `1375690658` = this repo's ID). A trust policy `StringLike` condition written with the plain name-only form fails every `AssumeRoleWithWebIdentity` call with "Not authorized" — confirmed via CloudTrail `LookupEvents`, not a propagation-delay issue (waited 10+ minutes with no change). The trust policy on `github-actions-cd-frontend` now matches the actual `@id`-qualified form above. **If this resurfaces on a new role** (e.g. when creating per-service roles later), check the actual `sub` value via CloudTrail (`cloudtrail:LookupEvents` filtered on `EventName=AssumeRoleWithWebIdentity`) rather than assuming the plain-name form or re-guessing propagation delay.
- **Argo CD `argocd-repo-server` crash-loop bug (found + fixed 2026-09-20):** the `copyutil` init container's `ln -s /var/run/argocd/argocd /var/run/argocd/argocd-cmp-server` command is not idempotent — it fails with "File exists" if the symlink already exists in the container's persistent `emptyDir` volume (which survives container restarts, only cleared on pod recreation), causing an unbounded crash loop (observed at 1088 restarts on this cluster). This is a known, already-fixed upstream bug: **argoproj/argo-cd#26595** — newer Argo CD releases replace the `ln -s` with an idempotent `cp -n`. Fix applied: upgraded Argo CD to current stable via `kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml` (rather than just restarting/deleting the affected pod, which only masks the bug until the next crash). **Exact version now running: `quay.io/argoproj/argocd:v3.5.3` (confirmed 2026-09-20 via `kubectl -n argocd get deployment argocd-repo-server -o jsonpath='{.spec.template.spec.containers[0].image}'`).** If this crash loop resurfaces after a future cluster restart on an *older* Argo CD version, this is the known cause — upgrade, don't just restart the pod.

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
- No long-lived Git credentials (PATs) anywhere either — the app repo's CD pipeline never touches the GitOps repo directly (see Architecture decisions: GitOps update mechanism).
- GitHub Actions must not deploy directly via unrestricted kubectl — always go through the GitOps flow (GitHub Actions → ECR → Argo CD Image Updater → GitOps repo → Argo CD → Kubernetes).
- Excluded on purpose (do not add): Amazon RDS PostgreSQL, Amazon SES, Semgrep/separate SAST stage, Syft/separate SBOM stage.

## Repository structure

This repo is the **application repository** (source, Dockerfile, CI workflows). The Kubernetes manifests/Helm/Argo CD Application definitions live in a **separate GitOps repository** — do not mix the two.

## Full CI pipeline stages (in order)

Checkout → Secret scanning (Gitleaks/TruffleHog) → Dependency scanning → Lint/format → Unit tests → SonarQube → Hadolint → Trivy (filesystem/config) → kubeval/kubeconform → Kyverno/OPA policy validation → Docker build → Trivy (image) → Cosign (image signing) → PR review/approval.

## Full CD pipeline stages (in order, on merge to main)

Build prod image → tag with Git SHA → Trivy scan → Cosign sign → GitHub OIDC auth to AWS → push to ECR. **The app repo's CD pipeline stops here.** From this point on it's out-of-band, handled by cluster-side controllers: Argo CD Image Updater detects the new ECR tag → updates `values.yaml` in the GitOps repo (using Argo CD's own Git credentials, no PAT in this repo) → Argo CD detects the GitOps repo change → syncs to cluster → rolling update → verify health → notify (Slack/SNS) → rollback via Git revert or Argo CD rollback if failed.
