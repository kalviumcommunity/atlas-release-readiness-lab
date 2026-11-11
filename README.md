# Atlas Release Readiness Lab

Small containerised Flask service with broken packaging, Kubernetes routing and workflow secret handling.

## ⚠️ This repository is intentionally broken

This repository contains **exactly three planted release-blocking issues** (one Docker
issue, one Kubernetes issue, one release-workflow secret-safety issue). Your task is to
find and fix them, then produce a release-readiness pack.

**Do not work in this source repository.** Fork it first, then clone your fork:

```bash
git clone https://github.com/<your-username>/atlas-release-readiness-lab.git
cd atlas-release-readiness-lab
git checkout -b fix/release-readiness
```

## Expected healthy behaviour (target state)

When the three issues are fixed, the following should be true:

- Docker image builds and is tagged `atlas-service:v1.4.0`.
- Container listens on `0.0.0.0:5000` inside the image.
- Running the container with `-p 8080:5000` exposes the health endpoint at
  `http://localhost:8080/health`, returning:

  ```json
  {"status": "ok", "version": "1.4.0"}
  ```

  with HTTP 200.

- Kubernetes Deployment `atlas-deployment` (namespace: default) runs one replica of
  container port `5000`, with Pod label `app: atlas`.
- Kubernetes Service `atlas-service`, type `ClusterIP`, exposes port `80` targeting
  container port `5000`, and its selector **must match** the Deployment Pod label
  `app: atlas` for traffic to route correctly.
- `.github/workflows/release.yml` triggers on tags matching `v*.*.*` and on
  `workflow_dispatch`. It has a `release-check` job and a `production-deploy` job
  that `needs: release-check` and runs under a GitHub `environment: production`
  gate (approval additionally requires separately configured environment reviewers).
- Any reference to the registry token must use `${{ secrets.REGISTRY_TOKEN }}` only
  where required, and the value must never be printed or logged.
- Release is tagged `v1.4.0`. If the release must be rolled back, the documented
  rollback action is to redeploy the previous known-good image, `atlas-service:v1.3.0`.

## Repository layout

```
app.py                        Flask app, GET /health
requirements.txt              pinned dependencies
Dockerfile                    container build
k8s/deployment.yaml           Deployment atlas-deployment
k8s/service.yaml              Service atlas-service
k8s/secret.yaml               placeholder Secret (stringData, fake value only)
.github/workflows/release.yml release workflow with production gate
release-readiness.md          template you must complete
```

## Prerequisites and checks

Git, Docker Engine, curl and internet access to download python:3.11-slim and Flask==3.0.3. Reserve host port 8080. No host Python installation or cloud account needed for Docker validation. kubectl is needed for Kubernetes validation; server-backed validation, kubectl diff and apply require a configured reachable cluster. Optional Kind/Minikube cluster must be isolated; do not apply to a real production context. Client dry-run is not proof of service routing and may still contact an API server for schema/discovery. If cluster validation is unavailable, state that limit rather than claiming deployment.

```bash
docker build -t atlas-service:v1.4.0 .
docker run -d --name atlas-release -p 127.0.0.1:8080:5000 atlas-service:v1.4.0
curl --max-time 5 -i http://localhost:8080/health
# Only with a disposable configured cluster/context:
kubectl apply --dry-run=client -f k8s/
kubectl diff -f k8s/
```

Starter build fails intentionally. Compare Service selectors against Pod labels. Repair workflow secret handling without printing values or removing validation/production review. The deploy job is a placeholder: an environment name alone does not enforce approval; required reviewers must be configured separately in GitHub Settings. Do not add a real registry token or run the unsafe original token-printing step with real credentials. Keep the Kubernetes secret placeholder fake.

Tag and inspect the release:

```bash
git status
git tag v1.4.0
git show v1.4.0 --stat
git log -1 --oneline
```

## Notes

- Never commit real secrets, tokens, private keys, or cloud credentials.
  `k8s/secret.yaml` must only ever contain a placeholder value.
- Do not delete checks, remove the `production` environment gate, or bypass
  failing steps to "fix" an issue — find the actual root cause instead.
- Complete `release-readiness.md` with observed local results, risks and explicitly unverified controls.

Cleanup only your container: `docker rm -f atlas-release`.
