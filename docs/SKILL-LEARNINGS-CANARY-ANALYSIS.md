# Skill Learnings: Canary Analysis & Progressive Delivery

## Session Summary

**Base Commit:** `03bb84be208d7f255a8e2523cf098beb51565f5c`
**Session Focus:** Testing and fixing canary analysis with New Relic and Prometheus providers
**Key Achievement:** Fixed `secretKeyRef` issue in Argo Rollouts web provider by creating Job-based analysis template

---

## 1. User Prompts (Chronological)

| # | Prompt | Category |
|---|--------|----------|
| 1 | "get the latest from git" | Setup |
| 2 | "lets do a small cosmetic change and go through code to dev to qa" | E2E Testing |
| 3 | "after ECR push, kustomize files need to updated in the git to trigger gitops .. why is that not happening" | Pipeline Issue |
| 4 | "yes, lets chain them" | Pipeline Enhancement |
| 5 | "is QA setup to do canary? if so is it using prometheus or newrelic" | Discovery |
| 6 | "i would like to test both newrelic and also prometheus .. lets test one after another" | Testing Request |
| 7 | "can use GHA for all work including testing and validation, u can prefix those GHA as tmp or something like that, NEVER NEVER NEVER USE LOCAL" | Constraint |
| 8 | "i fixed GHA Pat lets try again" | Retry |
| 9 | "give me list of variable for Newrelic and Jira" | Configuration |
| 10 | "added NR environment variable, lets continue" | Configuration |
| 11 | "can u explain me in greater detail with more logs why the rollout entered Degraded state because the New Relic analysis template is failing..." | Debug Request |
| 12 | "Create a fixed Job-based New Relic template that properly handles secrets" | Fix Request |
| 13 | "lets do another cosmetic change and test end to end and get detailed summary .. of canary deployment as well" | E2E Testing |
| 14 | "we should be able to watch better how canary is progressing ... any creative ideas without leaving this editor" | UX Enhancement |
| 15 | "simulate canary latency issue and roll-back scenario" | Simulation Request |

---

## 2. Issues Discovered & Fixed

### Issue 1: Build → Deploy Not Chained

**Problem:** After ECR push, the Deploy DEV workflow wasn't triggered automatically. GitOps requires kustomization.yaml to be updated with new image tags.

**Root Cause:** The Build & Push workflow ended after pushing to ECR without triggering the next stage.

**Solution:** Added `trigger-deploy` job to `20-ci-build-push.yaml`:

```yaml
trigger-deploy:
  name: "🚀 Trigger Deploy"
  needs: [prepare, build-vote, build-result, build-worker, build-summary]
  if: |
    always() &&
    (inputs.auto_deploy == true || inputs.auto_deploy == null) &&
    needs.build-vote.result == 'success' &&
    needs.build-result.result == 'success' &&
    needs.build-worker.result == 'success'
  steps:
    - name: Trigger Deploy to DEV
      run: |
        gh workflow run "30-cd-deploy-dev.yaml" \
          --repo ${{ github.repository }} \
          -f image_tag="${{ needs.prepare.outputs.image_tag }}"
      env:
        GH_TOKEN: ${{ secrets.GH_PAT }}
```

---

### Issue 2: `secretKeyRef` in Argo Rollouts Web Provider

**Problem:** New Relic analysis template using web provider failed with "Degraded" status.

**Root Cause:** Argo Rollouts' web provider doesn't properly resolve `secretKeyRef` in headers. The secret value is passed as literal text instead of the actual secret.

**Original (Broken) Code:**
```yaml
provider:
  web:
    url: "https://api.newrelic.com/..."
    headers:
    - key: Api-Key
      valueFrom:
        secretKeyRef:        # ❌ NOT RESOLVED
          name: newrelic-api-key
          key: api-key
```

**Solution:** Created Job-based analysis template that properly injects secrets via environment variables:

```yaml
provider:
  job:
    spec:
      template:
        spec:
          containers:
          - name: newrelic-check
            image: curlimages/curl:8.5.0
            env:
            - name: NR_API_KEY
              valueFrom:
                secretKeyRef:    # ✅ WORKS IN PODS
                  name: newrelic-api-key
                  key: api-key
            command: ["/bin/sh", "-c"]
            args:
            - |
              curl -H "Api-Key: $NR_API_KEY" $NR_API_URL/...
              # Exit 0 = pass, Exit 1 = fail
```

**File Created:** `.opsera-voting-app/k8s/base/rollouts/analysis-template-newrelic-job.yaml`

---

### Issue 3: Analysis Template Not Found in QA

**Problem:** Rollout failed with "AnalysisTemplate not found" error.

**Root Cause:** QA kustomization.yaml was explicitly deleting New Relic templates:

```yaml
- target:
    kind: AnalysisTemplate
    name: newrelic-combined-analysis
  patch: |-
    $patch: delete
```

**Solution:** Kept the new Job-based template (`newrelic-job-analysis`) which wasn't in the deletion list, and removed `success-rate` (Prometheus) from deletion to allow testing.

---

### Issue 4: GitHub Workflow Dispatch Not Recognized

**Problem:** `gh workflow run` returned HTTP 422 "Workflow does not have 'workflow_dispatch' trigger" even though it was defined.

**Root Cause:** GitHub's workflow indexing has caching issues with newly created/modified workflows.

**Solutions Tried:**
1. Wait 20-60 seconds - didn't work
2. Rename workflow file to force reindex - didn't work
3. Simplify workflow structure - didn't work

**Final Solution:** Add `push` trigger watching a config file:

```yaml
on:
  workflow_dispatch:
    inputs: ...
  push:
    paths:
      - ".github/canary-test-config.txt"  # Trigger via file change
```

---

### Issue 5: YAML Parsing Error in Workflow Files

**Problem:** Workflows with heredoc YAML generation failed with parsing errors.

**Root Cause:** Python/shell code inside `run:` blocks with heredocs confused GitHub's YAML parser.

**Solution:** Use Python for YAML generation:

```yaml
- name: Update Config
  run: |
    python3 << 'EOF'
    import yaml
    docs = [{"apiVersion": "...", ...}]
    with open("file.yaml", "w") as f:
        yaml.dump_all(docs, f)
    EOF
```

---

### Issue 6: Config File Parsing

**Problem:** Provider config file contained comments/timestamps that broke parsing.

**Solution:** Extract first word only:

```bash
PROVIDER=$(cat .github/canary-test-config.txt | awk '{print $1}' | head -1 || echo "newrelic")
```

---

## 3. Files Added/Modified by Area

### A. GitHub Actions Workflows (`.github/workflows/`)

| File | Type | Purpose |
|------|------|---------|
| `20-ci-build-push.yaml` | Modified | Added trigger-deploy job for auto-chaining |
| `ops-canary-analysis-test.yaml` | New | Test different analysis providers |
| `tmp-debug-analysis.yaml` | New | Debug rollout and analysis status |
| `tmp-canary-watch.yaml` | New | Live canary monitoring dashboard |
| `tmp-latency-simulation.yaml` | New | Simulate canary failure scenarios |
| `tmp-simulate-canary-failure.yaml` | New | Comprehensive failure simulation |
| `tmp-check-cluster.yaml` | New | Verify cluster state |

### B. Kubernetes Manifests (`.opsera-voting-app/k8s/`)

| File | Type | Purpose |
|------|------|---------|
| `base/rollouts/analysis-template-newrelic-job.yaml` | New | Job-based NR template (fixes secretKeyRef) |
| `base/rollouts/analysis-template-http.yaml` | New | HTTP health check fallback |
| `base/rollouts/analysis-template-newrelic.yaml` | Existing | Web-based NR templates (has issues) |
| `base/rollouts/kustomization.yaml` | Modified | Added new templates |
| `overlays/qa/kustomization.yaml` | Modified | Manage template deletions |
| `overlays/qa/patch-rollout-replicas.yaml` | Modified | Analysis template configuration |

### C. Config Files (`.github/`)

| File | Type | Purpose |
|------|------|---------|
| `canary-test-config.txt` | New | Trigger canary test workflow |
| `simulate-failure-config.txt` | New | Trigger failure simulation |
| `trigger-latency-sim.txt` | New | Trigger latency simulation |

### D. Application Code (`vote/`)

| File | Type | Purpose |
|------|------|---------|
| `templates/index.html` | Modified | Version display for testing |
| `app.py` | Modified | (During simulation only) |

---

## 4. Key Technical Decisions

### Decision 1: Job Provider vs Web Provider

**Context:** Need to use secrets in Argo Rollouts analysis.

**Options:**
1. Web provider with secretKeyRef (documented but buggy)
2. Job provider with environment variables
3. Hardcoded secrets (security risk)

**Decision:** Use Job provider.

**Rationale:**
- Kubernetes properly handles secretKeyRef in pod environment variables
- More control over analysis logic
- Can include multiple checks in one metric
- Better logging and debugging

---

### Decision 2: GitHub Actions for All Cluster Operations

**Constraint:** User explicitly requested "NEVER USE LOCAL" for kubectl commands.

**Implementation:**
- All kubectl operations via GitHub Actions workflows
- Created `tmp-*` prefixed workflows for testing
- Used `gh workflow run` to trigger from terminal
- Workaround: Push-based triggers when workflow_dispatch fails

---

### Decision 3: Analysis Provider Selection

**Context:** QA environment needed analysis provider configuration.

**Implementation:**
```
Available Providers:
├── newrelic-job-analysis (Job-based, handles secrets properly) ✅
├── newrelic-combined-analysis (Web-based, has secretKeyRef issue) ❌
├── success-rate (Prometheus-based) ✅
└── http-health-check (HTTP-based, simple health check) ✅
```

**Default:** `newrelic-job-analysis` for full APM analysis, `http-health-check` as fallback.

---

### Decision 4: Canary Steps Configuration

**QA Canary Configuration:**
```yaml
steps:
  - setWeight: 10
  - pause: {duration: 1m}
  - setWeight: 30
  - pause: {duration: 1m}
  - setWeight: 60
  - pause: {duration: 1m}
  - setWeight: 100
```

**Analysis Starts:** Step 1 (after 10% traffic)

---

## 5. Verification Commands (via GHA)

### Check Rollout Status
```bash
kubectl-argo-rollouts get rollout vote-rollout -n voting-app-qa
```

### List Analysis Templates
```bash
kubectl get analysistemplates -n voting-app-qa
```

### Get Analysis Run Details
```bash
kubectl get analysisruns -n voting-app-qa --sort-by='.metadata.creationTimestamp' -o wide
```

### Check Secret Exists
```bash
kubectl get secret newrelic-api-key -n voting-app-qa -o jsonpath='{.data}' | jq 'keys[]'
```

---

## 6. Learnings to Integrate Back to Skill

### 6.1 Add to Analysis Template Section

```markdown
### Analysis Provider Selection

#### Web Provider Limitations
- `secretKeyRef` in headers NOT properly resolved by Argo Rollouts
- Use Job provider instead for any authentication needs

#### Job Provider Pattern (Recommended)
```yaml
provider:
  job:
    spec:
      template:
        spec:
          containers:
          - name: analysis-check
            image: curlimages/curl:8.5.0
            env:
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: secret-name
                  key: api-key
            command: ["/bin/sh", "-c"]
            args:
            - |
              RESPONSE=$(curl -s -H "Api-Key: $API_KEY" "$URL")
              # Parse response and determine pass/fail
              if [ "$METRIC" -le "$THRESHOLD" ]; then
                exit 0  # Pass
              else
                exit 1  # Fail
              fi
```
```

### 6.2 Add to CI/CD Pipeline Section

```markdown
### Auto-Chain Build to Deploy

Add trigger job at end of build workflow:

```yaml
trigger-deploy:
  needs: [build-jobs]
  if: success()
  steps:
    - run: |
        gh workflow run "deploy.yaml" \
          --repo ${{ github.repository }} \
          -f image_tag="${{ needs.prepare.outputs.image_tag }}"
      env:
        GH_TOKEN: ${{ secrets.GH_PAT }}
```

**Requires:** `GH_PAT` secret with `workflow` scope.
```

### 6.3 Add to Troubleshooting Section

```markdown
### GitHub Workflow Dispatch Not Working

**Symptom:** `gh workflow run` returns HTTP 422 "Workflow does not have 'workflow_dispatch' trigger"

**Cause:** GitHub's workflow indexing caches old state

**Solutions:**
1. Wait 30-60 seconds after push
2. Add push trigger as backup:
   ```yaml
   on:
     workflow_dispatch:
     push:
       paths:
         - ".github/trigger-file.txt"
   ```
3. Modify file to trigger via push instead
```

### 6.4 Add to Testing Section

```markdown
### Canary Failure Simulation

To test rollback scenarios:

1. **Inject Latency:**
   - Modify app to add `time.sleep(2.0)` in main route
   - Build and deploy bad image
   - Watch canary fail and rollback

2. **Inject Error Rate:**
   - Add random 500 errors (50% failure rate)
   - Analysis will detect high error rate

3. **Inject Crash:**
   - Add delayed `sys.exit(1)` after 30 seconds
   - Pod restarts will trigger CrashLoopBackOff

**Workflow:** `tmp-latency-simulation.yaml`
```

---

## 7. Summary Statistics

| Metric | Value |
|--------|-------|
| User Prompts | 15 |
| Issues Fixed | 6 |
| Files Added | 12 |
| Files Modified | 8 |
| Workflows Created | 6 |
| Key Decision Points | 4 |

---

## 8. Integration Points with unified-code-to-cloud-v2

### Files to Add/Update in Base Skill:

1. **Analysis Templates:**
   - Add `analysis-template-newrelic-job.yaml` as the default NR template
   - Document web provider limitations

2. **Pipeline Workflows:**
   - Add `trigger-deploy` job pattern to Build workflow
   - Document GH_PAT requirement

3. **Testing Workflows:**
   - Include canary watch dashboard
   - Include failure simulation capabilities

4. **Troubleshooting Guide:**
   - Add secretKeyRef issue and solution
   - Add workflow dispatch caching issue

5. **Configuration Reference:**
   - Document required secrets for canary analysis
   - Document analysis provider options

---

*Generated from session ending 2026-01-28*
