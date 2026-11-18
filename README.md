# fieldin-optimization

---
name: Fieldin Deployment Stability Plan
overview: A practical plan to stop regressions, cut costs, and make Fieldin's deployment process stable, secure, and not reliant on a single agent -- prioritized by impact and ordered so each phase builds on the previous one.
todos:
  -  branch-protection
    content: Enable branch protection rules on fld-app-new, fld-api, gsc-api (require PRs, reviews, up-to-date branches)
    status: pending
  -  ci-setup
    content: Set up GitHub Actions with build + lint checks for the 3 main repos
    status: pending
  -  critical-tests
    content: Write 10-15 targeted tests for the most fragile code paths (map blocks, timeline, shift replay)
    status: pending
  -  cost-audit
    content: Run AWS resource audit -- stale envs, node utilization, idle services, NAT gateway costs
    status: in_progress
  -  auto-cleanup
    content: Implement automatic cleanup of expired ticket environments
    status: pending
  -  helm-charts
    content: Create Helm charts for fld-app-new, fld-api, gsc-api
    status: pending
  -  cicd-pipeline
    content: Build GitHub Actions pipeline for automatic staging deploys on merge to develop
    status: pending
  -  deploy-manifests
    content: Create Git repo with deployment manifests for staging and production state
    status: pending
isProject: false
---

# Fieldin Deployment Stability and Cost Optimization Plan

## The Core Problem

Three issues feeding each other:

- **Regressions**: No automated tests + direct pushes to develop = bugs come back after every deploy
- **Cost waste**: No visibility into resource usage, possibly stale environments and oversized nodes
- **Fragile process**: Deployment knowledge lives in one Cursor plugin; no standard tooling, no CI/CD

## Phase 1: Stop the Bleeding (Regressions) -- Weeks 1-3

This is the highest-impact change. Everything else is secondary until deploys stop breaking things.

### 1a. Enable branch protection on `develop` (Day 1)

For `fld-app-new`, `fld-api`, and `gsc-api`:
- Require Pull Requests for all merges to `develop`
- Require at least 1 review approval
- Require branches to be up-to-date before merging (this is the key one -- it forces developers to rebase/merge `develop` into their branch before merging, which surfaces conflicts BEFORE they hit develop)

This alone eliminates the "last merge wins" problem where branch B overwrites branch A's fix.

### 1b. Add basic CI checks with GitHub Actions (Weeks 1-2)

Start minimal -- don't try to get 80% test coverage overnight. For each main repo:

- **Build check**: Does the code compile/build without errors? (`npm run build` for fld-app-new, `composer install` for PHP repos)
- **Lint check**: Catch obvious errors (ESLint for TypeScript, PHPStan for PHP)
- **Smoke test**: For fld-app-new, a basic `ng build --configuration=production` catches import errors, missing dependencies, and TypeScript compilation issues that manual QA misses

These run on every PR. If the build fails, the PR can't be merged. This catches ~30-40% of regressions with near-zero test-writing effort.

### 1c. Add critical-path tests (Week 2-3)

Write a small number of targeted tests for the areas that keep breaking:
- The map/block services that had the `name` vs `id` bug
- The timeline component that had the hours mismatch (PRODUCT-8646)
- The shift replay filtering (PRODUCT-8634)

Even 10-15 focused tests on the most fragile areas prevent the most common regressions. These run in CI on every PR.

**Cost of Phase 1:** Near zero (GitHub Actions free tier gives 2000 min/month for private repos). Developer time: ~1 week for setup, then 1-2 hours per sprint for maintenance.

---

## Phase 2: Cut Costs -- Week 2-4 (parallel with Phase 1)

### 2a. AWS resource audit (I can do this now)

Using kubectl and AWS CLI, I can check:
- **Stale ticket environments** (`env-*` namespaces past their TTL) -- each one consumes pod resources
- **Oversized node groups** -- are nodes running at 30% CPU? Could use smaller instances
- **Idle deployments in staging** -- services nobody is testing that still consume memory/CPU
- **NAT gateway costs** -- often the hidden expensive item in AWS
- **Data transfer costs** -- cross-AZ traffic adds up silently

### 2b. Implement automatic environment cleanup

The deploy plugin already has a "cleanup stale environments" flow, but it's manual. Add a scheduled check (cron job or GitHub Action) that:
- Lists all `env-*` namespaces
- Flags those past TTL
- Posts to Slack asking for confirmation before cleanup
- Auto-deletes after 48h with no response

### 2c. Right-size staging resources

After the audit, adjust resource requests/limits on staging deployments. Most services probably request more CPU/memory than they use, which wastes node capacity and may force unnecessary node scaling.

**Estimated savings:** Typically 15-30% of the staging cluster bill. Production savings require more careful analysis.

---

## Phase 3: Deployment Independence -- Weeks 4-8

Make the deployment process work without depending solely on the Cursor agent.

### 3a. Helm charts for core services

Now Helm makes sense -- not to replace the plugin, but to give the team a fallback and standard interface:
- Create charts for `fld-app-new`, `fld-api`, `gsc-api`
- The deploy plugin can call `helm install/upgrade` instead of raw kubectl
- Any developer can also deploy manually with `helm` if the agent is unavailable
- CI/CD pipelines can use the same charts

### 3b. GitHub Actions deployment pipeline

Build a CI/CD pipeline that:
1. On PR merge to `develop`: automatically builds the Docker image via Jenkins (or GitHub Actions)
2. Automatically deploys to staging
3. Runs smoke tests against staging
4. Posts result to Slack

This means staging deploys happen automatically on merge -- no human or agent needed. Production still requires manual promotion (as it should).

### 3c. Deployment manifest in Git

Store the desired state of staging and production in a Git repo:
```
fieldin-deploy/
  staging/
    fld-app-new.yaml    # branch, image tag, config overrides
    fld-api.yaml
    gsc-api.yaml
  production/
    fld-app-new.yaml
    fld-api.yaml
```

This creates an audit trail and makes it easy to see what's deployed where. Changes to production require a PR with approval.

---

## Phase 4: Optional Hardening -- Month 3+

Only pursue these if Phases 1-3 are stable:

- **Terraform for AWS infrastructure** -- codify EKS, VPC, IAM for disaster recovery
- **ArgoCD for GitOps** -- auto-sync cluster state from Git (only if team grows)
- **E2E test suite** -- Cypress/Playwright tests for critical user flows
- **Staging-to-production promotion gates** -- automated checks before prod deploy is allowed

---

## Priority and Impact Summary

- **Phase 1 (Branch protection + CI)**: HIGHEST impact, LOWEST cost. Stops regressions. Do this first.
- **Phase 2 (Cost audit + cleanup)**: Direct cost savings. Can run in parallel with Phase 1.
- **Phase 3 (Helm + CI/CD pipeline)**: Makes the process independent and professional. Builds on Phase 1.
- **Phase 4 (Terraform + GitOps)**: Nice-to-have. Only if the team grows or requirements demand it.

---

## What This Solves

| Problem | Solution | Phase |
|---------|----------|-------|
| Same bug comes back after deploy | Branch protection + "require up-to-date" | 1a |
| Broken code reaches develop | PR reviews + CI build checks | 1b |
| Regressions in fragile areas | Targeted tests for known-breaking code | 1c |
| AWS cost waste | Resource audit + stale env cleanup | 2 |
| Deployment depends on one agent | Helm + CI/CD pipeline | 3 |
| No audit trail | Deployment manifests in Git | 3c |
| No disaster recovery for infra | Terraform (optional) | 4 |
