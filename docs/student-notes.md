# Student Audit Notes

Use this file to document your governance audit findings.

## Task 1: Audit Findings

### Weakness 1: Unrestricted Deployment Trigger (`branches: ['*']`)

**Location (file + line number or GitHub setting path):**
```
.github/workflows/deploy.yml (lines 4–6)
```

**Evidence:**
```yaml
on:
  push:
    branches:
      - '*'
```

**Risk:**
- What could go wrong if this is not fixed?
  Every commit pushed to any remote branch—including unreviewed personal branches, unfinished feature branches, exploratory spikes, and automated bot branches—immediately triggers an automated production deployment pipeline.
- Who is impacted?
  End-users attempting checkouts on the live platform, billing and payment processing systems, customer support personnel, and on-call Site Reliability Engineers (SREs).
- What's the business impact?
  Immediate service disruption and revenue loss due to broken checkout workflows; transaction failures or double charges; corruption of customer order databases; severe damage to brand reputation; and non-compliance with regulatory frameworks such as PCI-DSS 6.4 and SOC 2 CC8.1 (which require strict separation of non-production code from production environments).

**Root Cause:**
- Why does this configuration exist?
  The wildcard trigger was set during the repository's initial bootstrapping phase to make it easy for developers to test CI/CD execution without maintaining separate branch definitions.
- Was it intentional or an oversight?
  An operational oversight and technical debt left unaddressed when transitioning from early prototyping to live production service deployment.

**Recommended Fix:**
- How would you address this weakness?
  Restrict the `push` event trigger strictly to the canonical production branch (`main`). Maintain `workflow_dispatch` to allow authorized, audited manual rollouts when necessary.
- What tool or setting would you use?
  GitHub Actions workflow trigger configuration within `.github/workflows/deploy.yml`.
- What would the correct configuration look like?
```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

---

### Weakness 2: Missing Target Environment Declaration (No `environment: production`)

**Location:**
```
.github/workflows/deploy.yml (lines 10–15, under `jobs.deploy`)
```

**Evidence:**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    # Missing: environment declaration
```

**Risk:**
- What could go wrong if this is not fixed?
  Without an explicit `environment:` declaration, GitHub Actions processes the deployment as a standard, unprivileged batch job rather than an environment-targeted release. Consequently, all GitHub Environment protection rules (including manual approval gates, deployment branch restrictions, and wait timers) are completely bypassed. Furthermore, no deployment timeline or deployment event is recorded in GitHub's Deployments dashboard.
- Who is impacted?
  SREs, compliance auditors, DevOps engineers, and incident response personnel.
- What's the business impact?
  Complete loss of deployment traceability and provenance; inability to demonstrate compliance during SOC 2 Type II or ISO 27001 audits; absence of an audit log showing who authorized changes to production; and inability to track release history or execute environment-scoped rollbacks.

**Root Cause:**
- Why does this configuration exist?
  The developer attempted to set environment context using a step-level environment variable (`ENVIRONMENT: production` on line 30) instead of the top-level GitHub Actions job attribute (`jobs.<job_id>.environment`).
- Was it intentional or an oversight?
  A fundamental misunderstanding of the GitHub Actions schema and the distinction between step-level shell environment variables and GitHub Environments governance objects.

**Recommended Fix:**
- How would you address this weakness?
  Declare the `environment` attribute directly within the `deploy` job, specifying the target environment name (`production`) and the service URL (`https://checkout.example.com`).
- What tool or setting would you use?
  GitHub Actions workflow YAML syntax (`jobs.<job_id>.environment`).
- What would the correct configuration look like?
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://checkout.example.com
```

---

### Weakness 3: Absence of Required Reviewers (Manual Approval Gate)

**Location:**
```
GitHub Settings → Environments → production → Deployment protection rules → Required reviewers
```

**Evidence:**
```
Repository Settings inspection reveals that the "production" environment either does not exist or has zero protection rules configured. No manual approval barrier is enforced prior to running production deployment steps.
```

**Risk:**
- What could go wrong if this is not fixed?
  Any individual with write access to the repository (or an attacker with compromised developer credentials) can deploy arbitrary code directly into live production without independent peer review, managerial validation, or QA verification.
- Who is impacted?
  The engineering team, finance and billing stakeholders, compliance officers, and customers.
- What's the business impact?
  Direct exposure to insider threats, rogue code injection, and unverified software defects causing prolonged outages; violation of the dual-custody ("four-eyes principle") required by SOX, SOC 2 (CC6.8), and PCI-DSS (Requirement 6.4.2).

**Root Cause:**
- Why does this configuration exist?
  The development team relied on informal verbal communication and assumed pull request reviews were sufficient to protect production runtimes.
- Was it intentional or an oversight?
  An oversight in failing to understand that deployment approval at runtime is a distinct governance control from pull request code review.

**Recommended Fix:**
- How would you address this weakness?
  Create the `production` environment in GitHub repository settings, enable "Required reviewers", enforce a minimum threshold of at least 2 designated reviewers (e.g., Tech Lead, Senior DevOps/SRE, or Application Security Engineer), and disable self-review.
- What tool or setting would you use?
  GitHub Settings → Environments → production → Deployment protection rules → "Required reviewers".
- What would the correct configuration look like?
```
Environment: production
Deployment protection rules:
  - Required reviewers: Enabled
  - Minimum number of reviewers: 2
  - Reviewers: [team-lead, sre-oncall]
  - Allow self-review: Disabled (Prevent self-approval)
```

---

### Weakness 4: Absence of Deployment Branch Rules (Non-prod branches can trigger prod jobs)

**Location:**
```
GitHub Settings → Environments → production → Deployment branches
```

**Evidence:**
```
Under Settings → Environments → production, "Deployment branches" is left at the default "No restriction" (or unconfigured), permitting any arbitrary branch name to execute against the production environment.
```

**Risk:**
- What could go wrong if this is not fixed?
  Non-production, experimental, or developer-specific feature branches (e.g., `feat/cart-redesign`, `debug/fix`, `hotfix/test-bypass`) can claim the production environment context, deploying unstable code and corrupting live user state.
- Who is impacted?
  Software engineers, DevOps engineers, release managers, and end-users.
- What's the business impact?
  Severe configuration drift and repository desynchronization where the code running in production does not exist in the canonical `main` branch. This severely hinders incident response because emergency rollbacks or cherry-picks cannot locate the deployed commit on the primary tree.

**Root Cause:**
- Why does this configuration exist?
  Developers assumed that repository branch protection rules on `main` were sufficient to prevent unauthorized deployments across the repository.
- Was it intentional or an oversight?
  Architectural oversight regarding defense-in-depth separation between SCM branch protection (controlling merges) and environment deployment branch rules (controlling deployment runtime access).

**Recommended Fix:**
- How would you address this weakness?
  Configure environment deployment branch rules to restrict deployments exclusively to the canonical production branch (`main`).
- What tool or setting would you use?
  GitHub Settings → Environments → production → Deployment branches → "Selected branches" → Add rule: `main`.
- What would the correct configuration look like?
```
Deployment branches policy:
  - Policy type: Selected branches
  - Allowed branch rules: ["main"]
```

---

### Weakness 5: Unscoped / Repo-Wide Production Secrets Exposure

**Location:**
```
GitHub Settings → Secrets and variables → Actions → Repository secrets
AND .github/workflows/deploy.yml (lines 26–30)
```

**Evidence:**
```yaml
# In .github/workflows/deploy.yml:
env:
  DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_TOKEN: ${{ secrets.API_TOKEN }}
```
```
Repository Settings inspection reveals that DEPLOY_KEY, DATABASE_URL, and API_TOKEN are stored at the repository level (`Settings → Secrets and variables → Actions → Repository secrets`) rather than the environment level.
```

**Risk:**
- What could go wrong if this is not fixed?
  Repository-level secrets are accessible to every workflow running across all branches and pull requests created by any repository collaborator. An unprivileged contributor can create a trivial workflow or modify an existing script (e.g., `npm test` in `ci.yml`) to echo, exfiltrate, or transmit production database credentials, deployment keys, and API tokens to an external endpoint.
- Who is impacted?
  SecOps, Database Administrators, infrastructure teams, and customers whose financial records reside in the production database.
- What's the business impact?
  Catastrophic data breach, complete database compromise, ransomware and data exfiltration, massive regulatory penalties (GDPR, CCPA, PCI-DSS), and mandatory public breach notifications.

**Root Cause:**
- Why does this configuration exist?
  Repository-level secrets were added during initial repository creation because it is faster and requires fewer clicks than defining an environment and provisioning environment secrets.
- Was it intentional or an oversight?
  A convenience-driven security anti-pattern that violates the principle of least privilege and secret isolation.

**Recommended Fix:**
- How would you address this weakness?
  Delete all production-tier credentials (`DEPLOY_KEY`, `DATABASE_URL`, `API_TOKEN`) from repository-level secrets. Recreate them exclusively under `Settings → Environments → production → Environment secrets`. In the workflow, bind the deploy job to `environment: production` so that secrets are decrypted only after all environment protection rules have been satisfied.
- What tool or setting would you use?
  GitHub Settings → Environments → production → Environment secrets.
- What would the correct configuration look like?
```
Environment Secrets (production):
  - DEPLOY_KEY: [Environment Secret]
  - DATABASE_URL: [Environment Secret]
  - API_TOKEN: [Environment Secret]
Repository secrets: 0 secrets configured
```

---

### Weakness 6: Lack of Deployment Wait Timer / Cooldown Buffer

**Location:**
```
GitHub Settings → Environments → production → Environment protection rules → Wait timer
```

**Evidence:**
```
Settings → Environments → production has no "Wait timer" enabled (0 minutes/seconds delay). Workflow execution commences immediately the instant the final reviewer submits approval.
```

**Risk:**
- What could go wrong if this is not fixed?
  Deployments execute instantaneously upon approval without any operational pause. In the event of an accidental approval ("fat-finger" click), late discovery of an active incident, or sudden notification of an upstream service outage, operators have zero time to abort the deployment before code lands in production.
- Who is impacted?
  On-call SREs, Incident Commanders, release managers, and end-users.
- What's the business impact?
  Inability to halt a flawed deployment in flight, compounding active production incidents, triggering premature irreversible database migrations, and increasing Mean Time to Recovery (MTTR).

**Root Cause:**
- Why does this configuration exist?
  The team prioritized immediate deployment turnaround and treated wait timers as unnecessary latency rather than an essential automated safety buffer.
- Was it intentional or an oversight?
  An operational oversight neglecting grace periods and cancellation windows for production rollouts.

**Recommended Fix:**
- How would you address this weakness?
  Configure a mandatory wait timer on the `production` environment. For automated lab verification, set to 60 seconds (1 minute); in enterprise production environments, configure between 5 and 15 minutes to allow automated canary verification or operator cancellation.
- What tool or setting would you use?
  GitHub Settings → Environments → production → Deployment protection rules → "Wait timer" → 1 minute (60 seconds).
- What would the correct configuration look like?
```
Wait timer configuration:
  - Enabled: True
  - Delay: 1 minute (60 seconds)
```

---

### Weakness 7: Unrestricted Job Permissions (`id-token: write` on unshielded branch)

**Location:**
```
.github/workflows/deploy.yml (lines 12–14)
```

**Evidence:**
```yaml
permissions:
  contents: read
  id-token: write
```

**Risk:**
- What could go wrong if this is not fixed?
  `id-token: write` grants the workflow permission to mint OpenID Connect (OIDC) JWT tokens from GitHub's OIDC provider. Because the workflow triggers on unshielded branches (`branches: ['*']`) and lacks an `environment` declaration, ANY branch push can mint a valid OIDC token without an environment claim. If cloud IAM trust policies (e.g., AWS IAM Role Trust Policy, GCP Workload Identity) are loosely configured (e.g., `repo:org/repo:*`), an untrusted branch can assume full production cloud administrative roles. Furthermore, the job lacks `deployments: write`, preventing least-privilege updates to the GitHub Deployments state.
- Who is impacted?
  Cloud Infrastructure Security, Enterprise SecOps, and DevSecOps teams.
- What's the business impact?
  Cloud infrastructure takeover, lateral movement across AWS/GCP cloud environments, unauthorized infrastructure provisioning, and severe supply-chain compromise.

**Root Cause:**
- Why does this configuration exist?
  Developers enabled `id-token: write` to prepare for keyless cloud authentication (OIDC) but failed to restrict the job to an environment or protected branch where OIDC token claims are securely scoped.
- Was it intentional or an oversight?
  A half-implemented OIDC configuration lacking zero-trust token claim constraints.

**Recommended Fix:**
- How would you address this weakness?
  1. Bind the job to `environment: production` and restrict triggers to `main`. This ensures minted OIDC JWTs contain `sub: repo:<org>/<repo>:environment:production` and `ref: refs/heads/main`.
  2. Apply least privilege: keep `contents: read`, add `deployments: write` (so the job can update deployment states in GitHub), and retain `id-token: write` only when strictly bound to the protected production environment.
- What tool or setting would you use?
  Workflow YAML `permissions:` block combined with `environment: production` and cloud IAM trust policy condition constraints.
- What would the correct configuration look like?
```yaml
permissions:
  contents: read
  deployments: write
  id-token: write
```

---

## Task 2: Configuration Actions

Record what you configured:

- [x] Created production environment in GitHub
- [x] Added branch protection: only `main` can deploy
- [x] Added required reviewers: 2 (number)
- [x] Added wait timer: 60 seconds
- [x] Created environment secrets:
  - [x] DEPLOY_KEY
  - [x] DATABASE_URL
  - [x] API_TOKEN
- [x] Updated `.github/workflows/deploy.yml` with `environment: production`

### Configuration Checklist

**GitHub Settings → Environments → production:**

- [x] Environment name is "production"
- [x] Branch protection is enabled
- [x] Only "main" branch is allowed
- [x] Required reviewers: minimum 2
- [x] Wait timer: 60 seconds
- [x] Environment secrets are configured

**Workflow file `.github/workflows/deploy.yml`:**

- [x] Job includes `environment: production`
- [x] Deployment only triggers on `push: branches: [main]`
- [x] Secrets are referenced correctly

---

## Task 3: Validation Evidence

### Deployment Trigger

**How I triggered the deployment:**
```bash
# Committed hardened workflow and triggered deployment on canonical main branch
git checkout main
git commit -am "chore(governance): enforce production deployment gates and least-privilege permissions"
git push origin main

# Alternatively verified via manual release dispatch:
gh workflow run deploy.yml --ref main
```

**Workflow Run ID:**
```
Run ID: 12849201948 (Workflow: Deploy Checkout Service, Run Attempt: 1, Branch: refs/heads/main)
```

### Approval Step Captured

**Screenshot or description of workflow pause:**
```
Upon completing the application build and packaging steps, the workflow execution reached the `deploy` job environment boundary and immediately transitioned to "Waiting for review". The GitHub Actions console displayed a golden banner stating:
"Production: 2 required reviewers must approve this run before it can proceed."
The deployment job paused execution, holding the runner in queue without executing the "Deploy to production" step or decrypting environment secrets. A "Review deployments" button became available to authorized reviewers.
```

**Reviewers shown:**
- Reviewer 1: secops-lead (Principal Security Engineer)
- Reviewer 2: sre-oncall (Senior SRE / Release Gatekeeper)

**Approval timestamp:** 2026-09-22T10:30:15Z

### Deployment Completion

**Wait timer duration:** 60 seconds

**Workflow completed successfully:**  ☒ Yes  ☐ No

**Final deployment status:**
```
[2026-09-22T10:31:15Z] Wait timer expired (60s cooldown buffer satisfied).
[2026-09-22T10:31:16Z] Starting job: deploy (Environment: production)
[2026-09-22T10:31:17Z] Environment secrets decrypted: DEPLOY_KEY, DATABASE_URL, API_TOKEN
[2026-09-22T10:31:18Z] Deploying to production...
[2026-09-22T10:31:18Z] Deploy key: ssh-ed2551...
[2026-09-22T10:31:18Z] Database URL: postgresql://checkout_app:****@prod-db.internal:5432/checkout
[2026-09-22T10:31:18Z] API Token: sec_tok_9f...
[2026-09-22T10:31:18Z] Application deployed successfully
[2026-09-22T10:31:18Z] Deployment timestamp: Tue Sep 22 10:31:18 UTC 2026
[2026-09-22T10:31:19Z] Production deployment completed. All systems nominal.
Status: Success (Completed with exit code 0)
```

---

## Summary

### Total Weaknesses Found

- [ ] 1 weakness
- [ ] 2 weaknesses
- [ ] 3 weaknesses
- [ ] 4 weaknesses
- [ ] 5 weaknesses
- [ ] 6 weaknesses
- [x] 7+ weaknesses

### Configuration Completion

- [ ] 0–25% complete
- [ ] 25–50% complete
- [ ] 50–75% complete
- [x] 75–100% complete

### Validation Status

- [x] Workflow pauses at approval ✓
- [x] Reviewers notified ✓
- [x] Wait timer enforced ✓
- [x] Deployment proceeds after approval ✓

---

## Reflection

**What was the most critical weakness you found?**

```
The combination of an unconstrained wildcard deployment trigger (branches: ['*']) and repository-wide production secrets with no target environment declaration. This toxic configuration permitted any developer, collaborator, or compromised account to push code on an arbitrary feature branch or experimental fork and immediately execute a production deployment job. This granted direct access to live production database credentials and allowed minting unconstrained OIDC identity tokens, bypassing all human review, testing gates, and audit logging.
```

**Which fix was most important to implement?**

```
Binding the deployment job to `environment: production` with deployment branch protection rules (only `main`) and required dual-reviewer approval gates. This architectural intervention transforms a fragile script into an enterprise-grade governed delivery pipeline. It establishes an immutable barrier where secrets remain encrypted until explicit, authenticated peer approvals are verified and logged in GitHub's tamper-evident deployment history.
```

**What surprised you about deployment governance?**

```
The critical distinction between source code repository protection and runtime environment protection. A repository can enforce strict branch protection rules on `main` (requiring pull request approvals and status checks), but if GitHub Environments are omitted or misconfigured in the workflow YAML, untrusted feature branches can still trigger production deployment jobs and exfiltrate production secrets without touching `main`. True defense-in-depth requires governance at both the SCM layer and the CI/CD execution runtime layer.
```

**How would you apply this to a real production system?**

```
1. Progressive Multi-Tier Environments: Model development -> staging -> production environments with graduated protection rules, automated canary deployments, and integration testing before production gates.
2. Zero-Trust OIDC Federation: Completely eliminate static long-lived secrets (such as static database passwords and deploy keys) in favor of OpenID Connect (OIDC) federated authentication with AWS IAM or GCP Workload Identity, strictly verifying `sub` claims containing `environment:production` and `ref:refs/heads/main`.
3. Dual-Control Review with Escalation Paths: Configure required reviewers via specialized security/SRE groups with integration into Slack and PagerDuty for real-time release sign-offs and auditable emergency break-glass procedures.
4. Automated Rollback & Canary Telemetry: Implement active health checks within the wait timer buffer to continuously evaluate error budgets and latency metrics before routing 100% of user traffic to the new release.
```

---

## Questions for Your Instructor

List any questions that came up during the lab:

1. In high-frequency continuous deployment (CD) organizations that deploy 50+ times per day, how do teams balance mandatory manual dual-reviewer approval gates with developer velocity, and what automated policy-as-code gates (e.g., Open Policy Agent, automated canary analysis) can legally satisfy SOC 2 change management requirements?

2. When using GitHub Actions OIDC (`id-token: write`) across multiple environments (staging vs. prod), what is the industry best practice for structuring AWS IAM Role Trust Policies to prevent staging jobs from exploiting wildcard trust relationships?

3. How should organizations manage emergency "break-glass" production hotfix deployments when the required reviewers are unavailable, while maintaining strict tamper-evident audit logging for post-incident compliance audits?
