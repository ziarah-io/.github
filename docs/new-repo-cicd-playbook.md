# New Repo CI/CD Setup Playbook

Reference guide for replicating the standard CI/CD pipeline, branch protection, and
approval rules used across Ziarah services. Based on setup done for ZTAI-Frontend.

---

## 1. Workflow Files

Four workflow files go in `.github/workflows/`. All use reusable workflows from
`ziarah-io/.github/.github/workflows/`.

### `deploy-alpha.yml`

```yaml
name: Deploy Alpha
on:
  push:
    branches: [alpha]
concurrency:
  group: deploy-alpha
  cancel-in-progress: false
jobs:
  ci:
    uses: ziarah-io/.github/.github/workflows/reusable-ci.yml@main
    with:
      node-version: "20"
      package-manager: "npm"       # or pnpm
      language: "node"             # or python
      run-lint: false              # enable if eslint is configured
      run-tests: true
      run-build: false             # IMPORTANT: keep false — Docker build is the real gate
      run-codeql: false
    permissions:
      contents: read
      security-events: write
      actions: read
  security:
    uses: ziarah-io/.github/.github/workflows/reusable-security-scan.yml@main
    with:
      scan-filesystem: true
      scan-secrets: true
      generate-sbom: true
      language: "node"
      fail-on-vulnerability: false  # alpha is lenient
    permissions:
      contents: read
      security-events: write
    secrets: inherit
  build:
    needs: [ci, security]
    uses: ziarah-io/.github/.github/workflows/reusable-docker-build.yml@main
    with:
      image-name: "ztai-<service-name>"
      environment: "alpha"
      env-file-source: ""
    permissions:
      contents: read
      packages: write
    secrets:
      ACR_LOGIN_SERVER: ${{ secrets.ACR_LOGIN_SERVER }}
      ACR_USERNAME: ${{ secrets.ACR_USERNAME }}
      ACR_PASSWORD: ${{ secrets.ACR_PASSWORD }}
  deploy:
    needs: [build]
    uses: ziarah-io/.github/.github/workflows/reusable-deploy-aks.yml@main
    with:
      image-name: "ztai-<service-name>"
      environment: "alpha"
      cluster-name: "ztaiCluster"
      resource-group: "ztai"
      deployment-name: "ztai-<service-name>"
      container-name: "ztai-<service-name>"
    permissions:
      contents: read
    secrets:
      ACR_LOGIN_SERVER: ${{ secrets.ACR_LOGIN_SERVER }}
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
      MS_TEAMS_WEBHOOK_URI: ${{ secrets.MS_TEAMS_WEBHOOK_URI }}
      PAT_TOKEN: ${{ secrets.PAT_TOKEN }}
```

### `deploy-prod.yml`

Same structure as alpha with these differences:

```yaml
on:
  push:
    branches: [main]
concurrency:
  group: deploy-prod
  cancel-in-progress: false        # never cancel a prod deploy mid-flight
jobs:
  security:
    with:
      fail-on-vulnerability: true  # stricter than alpha
  build:
    with:
      environment: "prod"
  deploy:
    with:
      environment: "prod"
      cluster-name: "ztaiProdCluster"
```

### `pr-validation.yml`

```yaml
name: PR Validation
on:
  pull_request:
    branches: [main, alpha]
    types: [opened, synchronize, reopened]
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
jobs:
  validate:
    uses: ziarah-io/.github/.github/workflows/reusable-pr-validation.yml@main
    with:
      node-version: "20"
      package-manager: "npm"
      language: "node"
      run-lint: false              # disable if eslint not fully configured
      run-typescript: false        # disable if tsconfig has missing module issues
      run-build: false             # IMPORTANT: keep false — env vars not available in CI
    permissions:
      contents: read
      pull-requests: write
      security-events: write
```

> **Why `run-build: false`?** Next.js (and services using `@t3-oss/env-nextjs` or similar
> env validators) validate required env vars at build time. These vars aren't available in
> CI runners. The Docker build (which has access to secrets/env files) is the real gate.

### `ci.yml`

```yaml
name: CI
on:
  schedule:
    - cron: "0 6 * * 1"           # weekly Monday 6am — no push/PR triggers
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  ci:
    uses: ziarah-io/.github/.github/workflows/reusable-ci.yml@main
    with:
      node-version: "20"
      package-manager: "npm"
      language: "node"
      run-lint: false
      run-tests: true
      run-build: true              # full build runs here on schedule (secrets available via CI env)
    permissions:
      contents: read
      security-events: write
      actions: read
```

### `security-scan.yml`

```yaml
name: Security Scan
on:
  schedule:
    - cron: "0 2 * * *"           # nightly 2am — no push/PR triggers
concurrency:
  group: security-${{ github.ref }}
  cancel-in-progress: true
jobs:
  security:
    uses: ziarah-io/.github/.github/workflows/reusable-security-scan.yml@main
    with:
      scan-filesystem: true
      scan-container: false
      scan-secrets: true
      generate-sbom: true
      severity: "HIGH,CRITICAL"
      language: "node"
      node-version: "20"
      fail-on-vulnerability: false
    permissions:
      contents: read
      security-events: write
    secrets: inherit
```

> **Why schedule-only?** `deploy-alpha.yml` and `deploy-prod.yml` already run security
> scans as a gate before every build. Adding push/PR triggers causes duplicate runs.

---

## 2. Trigger Map — One Workflow at a Time

**Rule: when a deploy is running, no other workflow should independently trigger.**

`deploy-alpha.yml` and `deploy-prod.yml` already run CI + security internally as gates
before every build. If `security-scan.yml` or `ci.yml` also fire on push/PR events,
you get duplicate parallel runs — wasting minutes and cluttering the Actions tab.

| Event | Workflows triggered | Should NOT trigger |
|---|---|---|
| PR opened/updated to alpha or main | `pr-validation.yml` only | `security-scan.yml`, `ci.yml` |
| Push to alpha | `deploy-alpha.yml` (CI → Security → Build → Deploy) | `security-scan.yml`, `ci.yml`, `pr-validation.yml` |
| Push to main | `deploy-prod.yml` (CI → Security → Build → Deploy) | `security-scan.yml`, `ci.yml`, `pr-validation.yml` |
| Nightly 2am | `security-scan.yml` | — |
| Weekly Monday 6am | `ci.yml` | — |

### How to enforce this

**`security-scan.yml`** — schedule trigger only, never `push` or `pull_request`:

```yaml
on:
  schedule:
    - cron: "0 2 * * *"
# DO NOT add:
#   push:
#     branches: [main, alpha]
#   pull_request:
#     branches: [main, alpha]
```

**`ci.yml`** — schedule trigger only, never `push` or `pull_request`:

```yaml
on:
  schedule:
    - cron: "0 6 * * 1"
# DO NOT add:
#   push:
#     branches: [main]
#   pull_request:
#     branches: [main, alpha]
```

**`pr-validation.yml`** — `pull_request` only, never `push`:

```yaml
on:
  pull_request:
    branches: [main, alpha]
    types: [opened, synchronize, reopened]
# DO NOT add:
#   push:
#     branches: [main, alpha]
```

> **Why this matters:** merging a PR to alpha triggers both `deploy-alpha.yml` AND
> any workflow with `push: [alpha]`. With three devs merging throughout the day,
> duplicate security scans stack up and slow everyone down. The deploy pipeline is
> already the source of truth for CI + security gates.

---

## 3. Branch Protection Rulesets

Use the GitHub API to set rulesets. Replace user IDs as needed:
- `santhoshvijayan` → ID `3340521`
- `marimuthuraja-ziarah` → ID `181445198`

### Alpha ruleset

```bash
gh api repos/ziarah-io/<REPO>/rulesets -X POST --input - <<'EOF'
{
  "name": "alpha",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/alpha"],
      "exclude": []
    }
  },
  "bypass_actors": [
    {"actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always"},
    {"actor_id": 3340521, "actor_type": "User", "bypass_mode": "always"},
    {"actor_id": 181445198, "actor_type": "User", "bypass_mode": "always"}
  ],
  "rules": [
    {"type": "deletion"},
    {"type": "non_fast_forward"},
    {"type": "pull_request", "parameters": {
      "required_approving_review_count": 1,
      "dismiss_stale_reviews_on_push": false,
      "require_code_owner_review": false,
      "require_last_push_approval": false,
      "required_review_thread_resolution": false
    }}
  ]
}
EOF
```

### Prod/main ruleset

```bash
gh api repos/ziarah-io/<REPO>/rulesets -X POST --input - <<'EOF'
{
  "name": "Prod branch protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main"],
      "exclude": []
    }
  },
  "bypass_actors": [
    {"actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always"},
    {"actor_id": 3340521, "actor_type": "User", "bypass_mode": "always"},
    {"actor_id": 181445198, "actor_type": "User", "bypass_mode": "always"}
  ],
  "rules": [
    {"type": "deletion"},
    {"type": "non_fast_forward"},
    {"type": "required_deployments", "parameters": {
      "required_deployment_environments": ["alpha"]
    }},
    {"type": "pull_request", "parameters": {
      "required_approving_review_count": 1,
      "dismiss_stale_reviews_on_push": false,
      "require_code_owner_review": false,
      "require_last_push_approval": false,
      "required_review_thread_resolution": false
    }}
  ]
}
EOF
```

> **`required_deployments: ["alpha"]`** — main can only be merged after a successful
> deployment to the `alpha` GitHub environment. This ensures every prod release was
> first validated on alpha.

### Check for and disable `signed-commits` ruleset

Repos may have a pre-existing `signed-commits` ruleset (applies to `~ALL` branches)
that blocks merges entirely since most contributors don't GPG-sign commits.

```bash
# List all rulesets and look for signed-commits
gh api repos/ziarah-io/<REPO>/rulesets --jq '[.[] | {id, name}]'

# If found, disable it (replace ID)
gh api repos/ziarah-io/<REPO>/rulesets/<ID> -X PUT --input - <<'EOF'
{
  "name": "signed-commits",
  "target": "branch",
  "enforcement": "disabled",
  "conditions": {"ref_name": {"include": ["~ALL"], "exclude": []}},
  "rules": [{"type": "required_signatures"}, {"type": "non_fast_forward"}],
  "bypass_actors": []
}
EOF
```

> **Do NOT add `required_signatures` to your own rulesets.** The `signed-commits`
> global ruleset cannot be bypassed by anyone (not even bypass_actors). If commits
> are unsigned (most contributors), merges will be permanently blocked.

---

## 4. Kubernetes — imagePullPolicy

Ensure the deployment in the prod cluster pulls the latest image on every deploy:

```bash
kubectl patch deployment ztai-<service-name> -n default \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"ztai-<service-name>","imagePullPolicy":"Always"}]}}}}' \
  --context <prod-context>
```

Or set it directly in the deployment YAML:

```yaml
containers:
  - name: ztai-<service-name>
    imagePullPolicy: Always
```

---

## 5. Disable Dependabot Auto-fix PRs

Dependabot auto-fix PRs create noise. Disable for each repo (keep vulnerability alerts):

```bash
# Single repo
gh api repos/ziarah-io/<REPO>/automated-security-fixes -X DELETE

# All repos in org at once
python3 - <<'PYEOF'
import subprocess

repos = [
    "ZTAI-Frontend", "ZTAI-Hotel-Service", "ZTAI-Flight-Service",
    # ... add all repos
]

for repo in repos:
    r = subprocess.run(
        ["gh", "api", f"repos/ziarah-io/{repo}/automated-security-fixes", "-X", "DELETE"],
        capture_output=True, text=True
    )
    status = "disabled" if r.returncode == 0 else "skipped (not enabled)"
    print(f"{repo}: {status}")
PYEOF
```

---

## 6. Alpha ↔ Main Divergence Fix

If `alpha` and `main` diverge (different commit histories, not just ahead/behind):

```bash
# 1. Identify commits on alpha not in main
git log --oneline origin/main..origin/alpha --no-merges

# 2. Cherry-pick them onto main
git checkout main
git cherry-pick <sha1> <sha2> ...

# 3. Force-push with lease (safe — fails if someone else pushed)
git push origin main --force-with-lease

# OR: reset alpha to match main, then re-apply alpha-only commits
git checkout -B alpha origin/main
git cherry-pick <alpha-only-commits>
git push origin alpha --force-with-lease
```

---

## 7. Common Pitfalls

| Pitfall | Cause | Fix |
|---|---|---|
| `Invalid environment variables` in CI build | `@t3-oss/env-nextjs` validates at build time; env vars not in CI runners | Set `run-build: false` in `pr-validation.yml` and deploy workflow CI job |
| `Commits must have verified signatures` | `signed-commits` ruleset with no bypass actors | Disable the ruleset (see §3) |
| Duplicate workflow runs on push to alpha/main | `security-scan.yml` or `ci.yml` has `push:` or `pull_request:` trigger | Both must be schedule-only (see §2) — deploy workflows already run CI + security internally |
| 3 workflows firing when a PR is merged | `security-scan.yml` had `push: [main]` + deploy already runs security | Remove all push/PR triggers from `security-scan.yml` and `ci.yml` |
| PR branch workflow changes don't take effect | GitHub uses base branch workflow for `pull_request` events | Merge the fix to the base branch first (use bypass rights), then PRs pick it up |
| Push rejected: N unsigned commit violations | Feature branch created from local diverged branch | Always branch from `origin/alpha`: `git checkout -B <branch> origin/alpha` |
| `gh pr merge --admin` blocked by ruleset | `--admin` bypasses legacy branch protections, not rulesets | Use `gh api repos/.../pulls/<N>/merge -X PUT` as the bypass actor user |
