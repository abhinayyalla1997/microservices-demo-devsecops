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
- [ ] Phase 2: Application containerization (N/A per decision — Google's own Dockerfiles used as-is, nothing to build here)
- [x] Phase 3: CI pipeline (GitHub Actions) — **frontend only.** `ci.yml` + `build-scan-sign` composite action live and working (build → GHCR push → Trivy → Cosign, PR-time). The other 11 services are not yet in the matrix — see "Next session" below.
- [x] Phase 3.5: GitOps repository created — `abhinayyalla1997/online-boutique-gitops` (public), chart adapted from the existing one, **frontend only enabled**, working end to end (see Phase 6).
- [ ] Phase 4: AWS infrastructure (Terraform) — still not done; the OIDC role/provider and ECR repo that exist were created manually via Console, not Terraform. They can be imported into Terraform state later without behavior changes.
- [ ] Phase 5: Kubernetes cluster (kubeadm on EC2) — cluster itself pre-existed this project (not built by it); no Terraform/IaC for it yet.
- [~] Phase 6: GitOps and CD — **frontend proven working end to end** (CI → ECR via OIDC → Argo CD Image Updater → GitOps repo → Argo CD auto-sync → cluster → confirmed reachable in a browser, 2026-09-20). Argo CD itself pre-existed (184 days old) but had a crash-looping `repo-server` (see Architecture decisions) — upgraded to fix. Image Updater newly installed this session. Not yet done: Kyverno admission webhook, the other 11 services, ingress controller (NodePort used as a stopgap), a durable ECR pull-credential mechanism (see Architecture decisions — current one expires in 12h).
- [ ] Phase 7: Observability (Prometheus/Grafana/Alertmanager + OpenObserve/Fluent Bit for logs)
- [ ] Phase 8: Documentation

**Last session left off at:** 2026-09-20 — Got `frontend` working end to end and confirmed reachable in a browser (`http://<worker-public-ip>:30000`), proving the entire pipeline for real: CI (`ci.yml`) → CD (`cd.yml`, OIDC to AWS) → ECR (`online-boutique` repo) → Argo CD Image Updater → `online-boutique-gitops` repo → Argo CD auto-sync → pod running in the `boutique` namespace. Along the way, fixed real bugs (all documented in Architecture decisions below): an OIDC trust-policy `sub`-claim mismatch, an upstream Argo CD `repo-server` crash-loop bug (upgraded to v3.5.3), a Helm chart image-tag double-prefix bug, a wrong liveness/readiness probe path (`/healthz` vs `/_healthz`), and a missing ECR pull-credential (added a 12h imagePullSecret as a stopgap). **Next session:** (1) make the ECR pull credential durable — either a CronJob to refresh the imagePullSecret every ~6h, or (better) the kubelet `ecr-credential-provider` backed by an EC2 instance IAM role, so this doesn't silently break in 12 hours. (2) Decide whether/how to expand the CI matrix, CD workflow, and GitOps chart to the other 11 services (all scaffolding already supports it — matrix arrays and `enabled` flags just need extending). (3) Correct the stale infra note below (3 workers, not 4). (4) Consider hardening the wide-open security group (currently `0.0.0.0/0` all ports).

## Existing infrastructure (do not rebuild from scratch)

- Kubernetes cluster already exists: 1 control plane + **3** worker EC2 instances (corrected 2026-09-20 — previously said 4 here, verified wrong via `aws ec2 DescribeInstances`: `K8s-MN` control plane, `K8s-WN1/2/3` workers), kubeadm-bootstrapped, Kubernetes v1.34.x.
- Control plane has a permanent Elastic IP attached (54.85.169.119) — public IP no longer changes on stop/start. Worker public IPs (for NodePort access): `54.172.250.49` (WN1), `3.89.3.180` (WN2), `3.86.66.91` (WN3).
- **Argo CD was already installed in this cluster before this project started** (namespace `argocd`, 184+ days old) — this project didn't install it, just fixed a crash-looping component and added Image Updater on top. See Architecture decisions.
- The shared security group (`K8s-sg`) allows all inbound traffic from `0.0.0.0/0` on all ports/protocols — pre-existing, not changed by this project, but worth hardening eventually.
- Helm chart already built at `helm/online-boutique/` (in a *different* local project directory, `microservices-demo-main/helm/online-boutique` — not in this repo) covering all 12 services (NetworkPolicy, HPA, Ingress, RBAC, PodDisruptionBudget, ConfigMap, Secret, Namespace) — passes `helm lint`. This is the chart that was adapted into the `online-boutique-gitops` repo (see below).
- `scripts/build-push.sh` exists to build all 12 images and push to ECR.
- AWS account: `cognativ` profile (read-only MCP access already configured separately in Claude Desktop/Code — not part of this repo).
- GitHub repo (app): `abhinayyalla1997/microservices-demo-devsecops` (public), created and pushed 2026-09-18. Contains `src/` (12 services, unmodified from Google's repo), `.github/workflows/` (`ci.yml`, `cd.yml`, frontend-only), and this `CLAUDE.md`.
- GitHub repo (GitOps): `abhinayyalla1997/online-boutique-gitops` (public), created 2026-09-19. Contains the adapted Helm chart (frontend enabled only) and `argocd/application.yaml`.

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
- **Argo CD Image Updater + Application deployed (2026-09-20):** installed via `kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml`. Argo CD's write access to the GitOps repo uses a repo-scoped SSH deploy key (not a broad PAT), registered via `argocd repo add`. The `online-boutique` Application (`argocd/application.yaml` in the GitOps repo) has `syncPolicy.automated` (prune + selfHeal) and Image Updater annotations watching for `frontend-*` ECR tags, writing back to `frontend.image.tag` in the GitOps repo's `values.yaml` via Git.
- **No durable ECR pull-credential mechanism yet (gap, as of 2026-09-20):** this is a self-managed kubeadm cluster (not EKS), so nodes have no built-in ECR auth. Pods were hitting `ImagePullBackOff` ("no basic auth credentials") until a `kubernetes.io/dockerconfigjson` secret (`ecr-pull-secret`, in the `boutique` namespace) was manually created from a local `aws ecr get-login-password` token and attached to the `frontend-sa` ServiceAccount's `imagePullSecrets`. **This token expires in 12 hours** — it is a stopgap for this session's validation goal, not a production fix. Before relying on this longer-term, replace it with either (a) a CronJob that refreshes the secret every ~6h, or (b) the kubelet `ecr-credential-provider` binary backed by an EC2 instance IAM role (no manual credentials, the standard approach for self-managed clusters pulling from ECR) — decide which in Phase 5/6 hardening.
- **Two bugs found and fixed in the adapted Helm chart (2026-09-20), both in `online-boutique-gitops`, not this app repo:** (1) `_helpers.tpl`'s image helper re-prepended the service name onto the image tag; since our ECR tagging convention already includes the service name in the tag itself (`frontend-<sha>`), this produced a broken double-prefixed ref (`frontend-frontend-<sha>`) — fixed by not re-prepending. (2) The frontend Deployment's liveness/readiness probes targeted `/healthz`, but Google's actual frontend source registers the route at `/_healthz` (see `src/frontend/main.go` in the app repo) — the probe path mismatch caused every liveness check to 404 and the container to be killed (exit 143) shortly after each successful start, a permanent crash loop despite the app working fine. Both are chart bugs, not application bugs — the app source was never touched, per the project rule.

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
