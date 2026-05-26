# K8s Deployment Runbook — Issues & Fixes

This document captures every problem hit while moving the Boutique app from
`docker-compose` to EKS, what the root cause was, and the exact change that
fixed it. Written so a future-you (or a teammate) can avoid stepping on the
same rakes.

---

## 1. CI: `IMAGE_URI` split across two lines, push fails with exit 127

### Symptom
The `Push Docker Image` step in `.github/workflows/ci.yml` failed with:

```
IMAGE_URI=***.dkr.ecr.
  ***.amazonaws.com/order-service:<sha>: No such file or directory
Error: Process completed with exit code 127.
```

### Root cause
The `AWS_REGION` GitHub secret had a **trailing newline** baked into it.
GitHub masks the visible characters as `***` but preserves whitespace/newlines.
When the value got substituted into the unquoted `IMAGE_URI=...` assignment,
the newline split the command across two lines — the second half
(`.amazonaws.com/...`) was interpreted as a separate command.

### Fix
Re-saved the `AWS_REGION` secret in GitHub UI → Settings → Secrets → Actions.
Typed `us-east-1` directly (no paste, which is what usually drags in a newline).

### Hardening (optional)
Quote the assignment in `ci.yml` so a stray newline becomes a visible parse
error instead of silently splitting:
```yaml
IMAGE_URI="${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/${{ matrix.service }}:${{ github.sha }}"
```

---

## 2. CI: `update-manifests` job succeeded but never committed anything

### Symptom
The `update-manifests` job in CI completed with no error, but no commit
appeared on `main` and image tags in `gitops/k8s/*.yml` were never updated.

### Root cause
The `sed` command in `ci.yml` looked for the literal account ID in the
manifest files:

```bash
sed -i "s|${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/${SERVICE}:.*|...|g" $MANIFEST
```

But the manifests contained the **placeholder** `<AWS_ACCOUNT_ID>` instead of
the real account number. The pattern never matched → no file change →
`git diff --staged --quiet` was true → commit was skipped → push had nothing
to do.

### Fix
Anchored the `sed` on the service name only, so it matches regardless of
what's on the left side of the image URL:

```yaml
sed -i -E "s|image: .*/${SERVICE}:.*|image: ${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/${SERVICE}:${NEW_TAG}|g" $MANIFEST
```

Files touched: `.github/workflows/ci.yml` (commit `7167b05`).

---

## 3. CI: `git push` rejected as non-fast-forward

### Symptom
```
! [rejected]        main -> main (fetch first)
hint: Updates were rejected because the remote contains work that you do not
hint: have locally.
```

### Root cause
The `update-manifests` job was kicked off via **"Re-run failed jobs"** on the
old workflow run. GitHub re-runs against the *original trigger SHA*, not the
latest `main`. So the job's local `HEAD` was at the old commit, but remote
`main` had advanced to the regex-fix commit. Push was non-fast-forward.

### Fix
Triggered a **fresh `workflow_dispatch`** instead of re-running the old run.
A fresh dispatch checks out the current tip of `main` and pushes
fast-forward.

### Long-term safeguard (not applied yet)
Add `git pull --rebase origin main` before `git push` in the workflow, plus a
small retry loop. Makes the job robust to both this re-run scenario and
genuine concurrent pushes.

---

## 4. K8s: backend pods in `CrashLoopBackOff` — `database "auth_db" does not exist`

### Symptom
All 5 backend services (auth, orders, order-service, product-service,
user-service) stuck in `CrashLoopBackOff`. Logs:

```
FATAL: database "auth_db" does not exist
```

### Root cause
**Docker Compose** uses Postgres's built-in init mechanism — it mounts
`./database/init` into `/docker-entrypoint-initdb.d`, and the Postgres
image auto-runs any SQL there on first boot. That's how `auth_db`,
`orders_db`, etc. get created in Compose.

**K8s** does not do this automatically. There's a separate
`gitops/k8s/database/restore-job.yml` that runs `psql -f boutique_full.sql`
to create the databases — but it was **missing from `gitops/kustomization.yml`**,
so `kubectl apply -k gitops/` never created the Job. Postgres came up with
only the default `postgres` DB; services crashed.

### Fix
Added the Job to `gitops/kustomization.yml`:
```yaml
resources:
  - k8s/database/statefulset.yml
  - k8s/database/service.yml
  - k8s/database/restore-job.yml    # <-- added
```

Re-applied with `kubectl apply -k gitops/`. The Job ran, the databases were
created, and the services recovered on their next backoff cycle (no manual
rollout needed).

---

## 5. Frontend: products page empty, "Network Error" on `http://localhost:3003`

### Symptom
After port-forwarding `svc/frontend` to `localhost:3000`, the page loaded but
the products section was empty. Browser console:

```
[ProductService] Fetching products from: http://localhost:3003
AxiosError: Network Error
```

### Root cause — two compounding issues

1. **The console log was lying.** Actual API calls go through `apiClient`
   in `frontend/src/services/api.ts`, which defaults to
   `http://localhost:3001/api`. The `localhost:3003` was just a misleading
   `console.log` in `productService.ts`.

2. **`REACT_APP_*` env vars are baked in at build time, not runtime.**
   The K8s frontend Deployment had:
   ```yaml
   env:
     - name: REACT_APP_API_URL
       value: "http://gateway:3001/api"
   ```
   This does **nothing** — the JS bundle was already built inside the Docker
   image, with whatever value `REACT_APP_API_URL` had during `npm run build`.
   Since CI doesn't pass that var at build, the bundle fell back to the
   hardcoded `||` defaults (`localhost:3001`, `localhost:3003`).

3. **No nginx proxy in K8s.** In Compose, an nginx sidecar fronted the app
   and routed `/api/*` to gateway, `/` to frontend. The Compose setup
   actually bypassed nginx for product calls (browser hit
   `localhost:3003` directly via Compose port mapping). In K8s there's no
   such host mapping, so the browser had nothing to connect to.

### Fix (all in one commit, `b2da9af`)

**Frontend code — use relative URLs:**
- `frontend/src/services/api.ts`:
  ```ts
  const API_BASE_URL = process.env.REACT_APP_API_URL || '/api';
  ```
- `frontend/src/services/productService.ts`: changed the log fallback to
  `'/api'` to match.

**New nginx in K8s** — `gitops/k8s/frontend/nginx.yml` containing:
- `ConfigMap` `nginx-config` with `nginx.conf` (mirrors the Compose one):
  - `/api/*` → `http://gateway:3001/api/`
  - `/`      → `http://frontend:3000`
- `Deployment` running `nginx:alpine` with the ConfigMap mounted at
  `/etc/nginx/conf.d/default.conf`.
- `Service` (ClusterIP) on port 80.

**Cleanup** — removed the no-op `REACT_APP_API_URL` env from
`gitops/k8s/frontend/deployment.yml`.

**Kustomization** — added `k8s/frontend/nginx.yml` to `resources:`.

---

## End-to-end deployment flow (current state)

1. **Push code to `main`.**
2. **Manually dispatch CI** (Actions → Boutique CI Pipeline → Run workflow).
   - Builds and pushes 7 Docker images to ECR (tag = commit SHA).
   - Auto-commits new image tags into `gitops/k8s/*.yml`.
3. **Pull the manifest commit locally:** `git pull`.
4. **Argo CD picks up the diff** (auto-sync currently OFF) — click **Sync**
   in the UI, or `argocd app sync boutique`.
5. **Access Argo CD UI:**
   ```powershell
   kubectl -n argocd port-forward svc/argocd-server 8080:443
   ```
   Login: `admin` / password from
   `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | <base64 decode>`.
6. **Access the app:**
   ```powershell
   kubectl -n boutique port-forward svc/nginx 80:80
   ```
   Open http://localhost:80. (Use a different local port like `9090:80` if
   `80` or `8080` is busy — Argo CD's port-forward already holds `8080`.)

---

## Things still worth doing

- [ ] Turn on Argo CD auto-sync (and self-heal) once you trust the pipeline.
- [ ] Switch CI trigger from `workflow_dispatch` to `push` on `main`, gated
      by path filters (don't rebuild on doc-only commits).
- [ ] Add `git pull --rebase origin main` + retry loop to the
      `update-manifests` job so the workflow survives concurrent pushes /
      re-runs against stale SHAs.
- [ ] Quote `IMAGE_URI=` in CI to make whitespace bugs in secrets fail loud.
- [ ] Replace per-service `kubectl port-forward` with an actual Ingress
      (ALB / nginx-ingress) so the app is reachable without tunnels.
- [ ] Move secret values out of `gitops/secrets.yml` into something
      managed (SSM Parameter Store + external-secrets, or sealed-secrets).
