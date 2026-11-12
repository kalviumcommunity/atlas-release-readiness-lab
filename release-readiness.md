# Release Readiness Pack — Atlas Service

Fill in every section below as you work through the three tasks. Keep entries short
and factual. Do not paste secret values anywhere in this file.

## 1. Release metadata

- Release version: v1.4.0
- Git commit hash: 7fd1275
- Git tag: v1.4.0
- Docker image name and version: atlas-service:v1.4.0
- Date/time: 2026-09-27
- Prepared by: Suryaprakash

## 2. Risk assessment (Task 1 — Docker)

- Issue found (what failed and how you noticed): Docker build failed on step 5 because `requirements-prod.txt` was referenced but did not exist in the repository.
- Impact if shipped as-is: Image build completely fails, preventing artifact generation and blocking the entire deployment pipeline.
- Evidence (build error summary): `COPY failed: file not found in build context: stat requirements-prod.txt: file does not exist`.
- Fix applied: Updated Dockerfile from `COPY requirements-prod.txt requirements.txt` to `COPY requirements.txt .`.
- Residual risk after fix: Low; image builds cleanly and passes local `/health` check with HTTP 200.

## 3. Pipeline stages (Task 2 — Kubernetes)

Describe what happens at each stage and where it was validated:

- Validate: Manifest syntax and resource validation performed with `kubectl apply -f k8s/`.
- Build: Docker image build via `docker build -t atlas-service:v1.4.0 .` to containerize the Flask application.
- Deploy: Kubernetes Deployment applies `atlas-deployment`, `atlas-service`, and `atlas-secret` into the cluster.
- Smoke test: Verified container endpoint responds with HTTP 200 on `http://localhost:8080/health`.

## 4. Environment gate

- Where does human approval occur in the pipeline? Inside the `production-deploy` job before the deployment placeholder executes.
- Which GitHub environment enforces it? `production`
- Why is this gate placed before production and not before `release-check`? Automated testing and image builds in `release-check` run automatically without blocking. This ensures human reviewers only spend time approving builds that have already passed all automated verification steps.

## 5. Secret handling (Task 3 — Workflow)

- Issue found in `.github/workflows/release.yml`: The workflow explicitly logged `${{ secrets.REGISTRY_TOKEN }}` to the build console output via `echo`.
- Why was it unsafe? Plain-text logging exposes confidential registry tokens to anyone with repository build log access, risking unauthorized image pushes/pulls.
- Fix applied (describe the safe pattern used, do not paste secret values): Replaced the unsafe `echo` log statement with a safe verification step that does not print the secret.
- Confirm no password, token, or private key is committed anywhere in the repo: Confirmed; `k8s/secret.yaml` contains only a placeholder value (`replace-me`).

## 6. Validation evidence

List commands run and a one-line result for each (attach screenshots separately
in your PDF submission):

- `docker build ...`: Successfully built image `atlas-service:v1.4.0`.
- `curl http://localhost:8080/health`: Returned `{"status": "ok", "version": "1.4.0"}` with HTTP 200.
- `kubectl apply -f k8s/`: Created `atlas-deployment`, `atlas-secret`, and `atlas-service`.
- `kubectl get deployment,pods,service`: Pod running 1/1, service allocated ClusterIP `10.96.145.175`.
- `kubectl describe service atlas-service`: Selector `app=atlas` mapped to Endpoint `10.244.0.5:5000`.
- `git tag` / `git show v1.4.0 --stat`: Tag `v1.4.0` created and points to commit `7fd1275`.

## 7. Rollback plan

- Rollback trigger (what condition would cause a rollback): Failure of post-deployment `/health` check, Pods entering `CrashLoopBackOff`, or elevated error rates in production.
- Rollback action (previous known-good image tag and how it is redeployed): Redeploy previous known-good version `atlas-service:v1.3.0` using `kubectl rollout undo deployment/atlas-deployment` (or updating image tag to `v1.3.0`).

## 8. Kubernetes vs Terraform

- One sentence: why would Terraform manage infrastructure provisioning while
  Kubernetes manages the application workload here?
Terraform provisions and manages underlying cloud infrastructure (such as virtual networks, VMs, and managed Kubernetes clusters) via Infrastructure as Code, whereas Kubernetes manages application container lifecycles, service routing, scaling, and rolling updates on top of that infrastructure.

## 9. Final decision

- Decision: **Ready**
- Reason: All three release blockers (Docker build failure, Kubernetes service routing mismatch, and workflow secret exposure) are fully resolved and verified; release tag and rollback strategies are documented.
