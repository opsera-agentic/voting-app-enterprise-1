# Voting App Enterprise Deployment Journey

## Overview

This document captures the complete deployment journey of a multi-service voting application to AWS EKS using GitOps (ArgoCD), including all issues encountered and their solutions.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER BROWSER                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS Load Balancer (nginx-ingress)                │
│  opsera-voting-app-dev.agent.opsera.dev                             │
│  results.opsera-voting-app-dev.agent.opsera.dev                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
┌───────────────┐    ┌─────────────────┐    ┌────────────────┐
│   Vote UI     │    │   Result UI     │    │    Worker      │
│   (Python     │    │   (Node.js      │    │   (.NET Core)  │
│    Flask)     │    │    Socket.io)   │    │                │
│   Port 8080   │    │   Port 8080     │    │                │
└───────┬───────┘    └────────┬────────┘    └───────┬────────┘
        │                     │                     │
        │ Push votes          │ Poll results        │ Process votes
        ▼                     │                     ▼
┌───────────────┐             │             ┌────────────────┐
│    Redis      │◄────────────┼─────────────│   PostgreSQL   │
│   (Queue)     │             │             │   (Database)   │
└───────────────┘             └─────────────►────────────────┘
```

## Infrastructure Components

| Component | Technology | Purpose |
|-----------|------------|---------|
| Hub Cluster | EKS `argocd-usw2` | Runs ArgoCD for GitOps |
| Spoke Cluster | EKS `opsera-usw2-np` | Runs application workloads |
| Container Registry | AWS ECR | Stores Docker images |
| GitOps | ArgoCD | Automated deployments from Git |
| Ingress | nginx-ingress | Load balancing & TLS termination |
| CI/CD | GitHub Actions | Build, scan, push images |

---

## Issues Encountered & Solutions

### Issue 1: Matrix Job Outputs Not Propagating

**Problem:** The CI/CD pipeline used matrix jobs for building images, but the `IMAGE_TAG` variable wasn't available in subsequent jobs.

**Symptom:** `update-kustomize` and `verify-argocd-sync` steps were being skipped.

**Root Cause:** Matrix jobs create multiple parallel executions, and outputs from matrix jobs can't be easily aggregated.

**Solution:** Move `IMAGE_TAG` generation to the `verify-prerequisites` job (non-matrix):

```yaml
verify-prerequisites:
  outputs:
    image_tag: ${{ steps.tag.outputs.image_tag }}
  steps:
    - name: Generate Image Tag
      id: tag
      run: |
        IMAGE_TAG="${GITHUB_SHA:0:7}-$(date +%Y%m%d%H%M%S)"
        echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT
```

---

### Issue 2: ArgoCD Sync Error - Immutable Selector

**Problem:** ArgoCD failed to sync with error: `spec.selector is immutable`

**Symptom:**
```
the object has been modified; please apply your changes to the latest version
field is immutable: spec.selector
```

**Root Cause:** Kustomize `commonLabels` adds labels to both `metadata.labels` AND `spec.selector.matchLabels`. Once a Deployment is created, its selector cannot be changed.

**Solution:** Use the `labels` transformer with `includeSelectors: false`:

```yaml
# Before (WRONG)
commonLabels:
  environment: dev

# After (CORRECT)
labels:
  - pairs:
      environment: dev
    includeSelectors: false  # Don't modify immutable selectors
```

Also required: Delete existing deployments to allow recreation with new labels.

---

### Issue 3: Port 80 Permission Denied (EACCES)

**Problem:** Vote and Result containers crashed with "Permission denied" on port 80.

**Symptom:**
```
Error: EACCES: permission denied, bind port 80
```

**Root Cause:** Containers run as non-root user (UID 1000) for security. Non-root users cannot bind to ports below 1024.

**Solution:** Change application ports from 80 to 8080:

**Dockerfile.vote:**
```dockerfile
EXPOSE 8080
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:8080", ...]
```

**Dockerfile.result:**
```dockerfile
EXPOSE 8080
CMD ["node", "server.js"]
# Also updated server.js: var port = process.env.PORT || 8080;
```

**Deployment manifests:**
```yaml
containers:
- name: vote
  ports:
  - containerPort: 8080  # Changed from 80
  livenessProbe:
    httpGet:
      port: 8080  # Changed from 80
```

**Services** still expose port 80 externally, routing to targetPort 8080:
```yaml
spec:
  ports:
  - port: 80           # External port
    targetPort: 8080   # Container port
```

---

### Issue 4: PVC Binding Timeout (No EBS CSI Driver)

**Problem:** StatefulSets for db and redis stuck in Pending state.

**Symptom:**
```
pod has unbound immediate PersistentVolumeClaims
0/4 nodes are available: 4 pod has unbound immediate PersistentVolumeClaims
```

**Root Cause:** The EKS cluster didn't have the AWS EBS CSI driver installed, so PersistentVolumeClaims couldn't be provisioned.

**Solution:** For dev environment, replace StatefulSets with Deployments using `emptyDir` volumes:

**db-deployment.yaml (dev-only):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: db
        image: postgres:15
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: db-data
        emptyDir: {}  # No PVC required
```

**kustomization.yaml (dev overlay):**
```yaml
resources:
  - ../../base
  - db-deployment.yaml      # Dev-only deployment
  - redis-deployment.yaml   # Dev-only deployment

patchesStrategicMerge:
  - |-
    $patch: delete
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: db
  - |-
    $patch: delete
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: redis
```

**Trade-off:** Data is lost when pods restart. For production, install EBS CSI driver or use RDS/ElastiCache.

---

### Issue 5: SAML SSO Blocking Git Push

**Problem:** GitHub Actions couldn't push commits back to the repository.

**Symptom:**
```
fatal: unable to access 'https://github.com/...': The requested URL returned error: 403
```

**Root Cause:** Organization enforces SAML SSO authentication, and the PAT token wasn't SSO-authorized.

**Solution:**
1. User validated PAT token for SSO in GitHub settings
2. Added graceful fallback in workflow with manual instructions:

```yaml
- name: Update Kustomization
  run: |
    # ... make changes ...

    if git push origin main 2>&1; then
      echo "### Kustomize Updated Successfully" >> $GITHUB_STEP_SUMMARY
    else
      echo "### Manual Update Required (SAML SSO)" >> $GITHUB_STEP_SUMMARY
      echo "Run this command locally:" >> $GITHUB_STEP_SUMMARY
      echo "sed -i '' 's|newTag: .*|newTag: ${IMAGE_TAG}|g' ${KUSTOMIZE_FILE}"
    fi
```

---

### Issue 6: ArgoCD Initial Admin Secret Not Found

**Problem:** Couldn't retrieve ArgoCD password for UI access.

**Symptom:**
```
Initial admin secret not found. Password may have been changed.
```

**Root Cause:** The `argocd-initial-admin-secret` may have been deleted after first login (security best practice).

**Solution:** Created workflow to reset ArgoCD admin password:

```yaml
- name: Reset ArgoCD Admin Password
  run: |
    NEW_PASSWORD=$(openssl rand -base64 12 | tr -dc 'a-zA-Z0-9' | head -c 16)
    HASHED_PASSWORD=$(htpasswd -nbBC 10 "" "$NEW_PASSWORD" | tr -d ':\n' | sed 's/$2y/$2a/')

    kubectl -n argocd patch secret argocd-secret \
      -p "{\"stringData\": {
        \"admin.password\": \"$HASHED_PASSWORD\",
        \"admin.passwordMtime\": \"$(date +%FT%T%Z)\"
      }}"
```

---

### Issue 7: Results Page 404 - Browser Cache

**Problem:** User got 404 on voting app URL, but it worked from GitHub Actions.

**Symptom:** Browser showed 404, but `curl` returned 200.

**Root Cause:** Browser had cached a previous 404 response.

**Solution:** Hard refresh (`Cmd+Shift+R`) or use incognito mode.

---

### Issue 8: Results Page - "Cannot GET /results"

**Problem:** Accessing `/results` path returned Express.js 404 error.

**Symptom:**
```
Cannot GET /results
```

**Root Cause:** The nginx ingress routed `/results` to the result service, but the result app only serves routes at `/`. When result app received `/results`, it had no matching route.

**Initial Attempt (Failed):** Add rewrite annotation:
```yaml
nginx.ingress.kubernetes.io/rewrite-target: /$2
path: /results(/|$)(.*)
```
This rewrote the path but static assets (JS, CSS) still broke.

**Final Solution:** Use separate subdomain for results:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: voting-app-result
spec:
  rules:
  - host: results.opsera-voting-app-dev.agent.opsera.dev  # Separate subdomain
    http:
      paths:
      - path: /
        backend:
          service:
            name: result
```

---

### Issue 9: Socket.io 400 Bad Request

**Problem:** Results page loaded but showed raw Angular templates instead of data.

**Symptom:**
```
POST /socket.io/?EIO=4&transport=polling 400 (Bad Request)
```

**Root Cause #1:** Missing WebSocket support in ingress.

**Solution Part 1:** Add WebSocket annotations:
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
  nginx.ingress.kubernetes.io/configuration-snippet: |
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
```

**Root Cause #2:** Multiple replicas break socket.io sessions.

Socket.io uses session IDs. With 2 replicas:
1. Request 1 → Pod A → Creates session `abc123`
2. Request 2 → Pod B → Doesn't know session `abc123` → 400 error

**Solution Part 2:** Scale result to 1 replica:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result
spec:
  replicas: 1  # Single replica for socket.io session affinity
```

For production with multiple replicas, use:
- Redis adapter for socket.io
- Sticky sessions via ingress annotation

---

### Issue 10: Results Show 0% / 100%

**Problem:** User always saw extreme percentages (0/100 or 100/0).

**Symptom:** Voting multiple times didn't change the ratio.

**Root Cause:** By design! The app uses **one vote per voter** (cookie-based voter_id). The database stores ONE row per voter_id, updating (not inserting) on each vote.

**Solution:** Not a bug - it's a feature! To see real percentages:
- Use multiple incognito windows (each gets new voter_id)
- Use different browsers
- Clear cookies between votes

---

## Workflows Created

| Workflow | Purpose |
|----------|---------|
| `ci-cd-voting-app.yaml` | Full CI/CD with quality gates (Grype, SonarQube) |
| `diagnostics.yaml` | ArgoCD & K8s health checks |
| `cleanup-resources.yaml` | Delete stale K8s resources |
| `argocd-access.yaml` | Get ArgoCD URL |
| `argocd-reset-password.yaml` | Reset ArgoCD admin password |
| `argocd-sync.yaml` | Force ArgoCD sync |
| `force-sync.yaml` | Hard refresh + sync |
| `debug-ingress.yaml` | Ingress & connectivity debug |
| `debug-voting.yaml` | Voting flow debug |
| `check-votes.yaml` | Quick database/redis check |

---

## Key Learnings

### 1. Kubernetes Labels & Selectors
- `spec.selector` is **immutable** after creation
- Use Kustomize `labels` transformer with `includeSelectors: false`
- Delete deployments to change selectors

### 2. Non-Root Containers
- Ports < 1024 require root privileges
- Always use high ports (8080, 3000, etc.) for non-root containers
- Services can still expose port 80 externally

### 3. StatefulSets vs Deployments
- StatefulSets require working storage provisioner
- For dev without EBS CSI: use Deployments + emptyDir
- For prod: install proper CSI driver or use managed services

### 4. GitOps with ArgoCD
- ArgoCD watches Git repo for changes
- Use hard refresh to clear cache: `argocd.argoproj.io/refresh: hard`
- Sync can be triggered via kubectl patch or ArgoCD CLI

### 5. Ingress Path-Based Routing
- Path rewrites break SPA static assets
- Use subdomains for complex apps with their own routing
- WebSocket apps need special ingress annotations

### 6. Socket.io & Load Balancing
- Socket.io requires session affinity
- Multiple replicas need Redis adapter or sticky sessions
- For dev: single replica is simplest

### 7. CI/CD Pipeline Design
- Matrix job outputs don't propagate well
- Generate shared values in prerequisite jobs
- Always handle SAML/SSO gracefully with fallbacks

---

## Final URLs

| Service | URL |
|---------|-----|
| Vote UI | https://opsera-voting-app-dev.agent.opsera.dev/ |
| Results UI | https://results.opsera-voting-app-dev.agent.opsera.dev/ |
| ArgoCD | Check `argocd-access.yaml` workflow output |

---

## Deployment Summary

```
Timeline:
1. Initial CI/CD setup → Matrix job output issues
2. First deployment → Immutable selector errors
3. Port fix → Permission denied on port 80
4. Storage fix → PVC binding timeout
5. Auth fix → SAML SSO blocking push
6. Results routing → Cannot GET /results
7. WebSocket fix → Socket.io 400 errors
8. Replica fix → Session affinity issues
9. SUCCESS → Both apps working!
```

Total issues resolved: **10**
Workflows created: **10**
Time to production: Multiple iterations with systematic debugging

---

*Generated by Claude Code during voting-app-enterprise deployment session*
