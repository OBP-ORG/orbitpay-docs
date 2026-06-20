# Release Checklist — OrbitPay

This checklist must be completed for every release of the OrbitPay protocol. Each item includes the responsible role and required approval where applicable.

## Release Naming Convention

```
v<MAJOR>.<MINOR>.<PATCH>

MAJOR — Contract redeployment required (breaking change)
MINOR — New feature, backward-compatible
PATCH — Bug fix, backward-compatible
```

Example: `v0.2.0` — first feature release after initial `v0.1.0`.

---

## Pre-Release (1 Week Before)

### Planning

- [ ] Release scope defined and documented in release notes draft — **Product Owner**
- [ ] All targeted issues/PRs linked to the release milestone — **Release Manager**
- [ ] Backward compatibility impact assessed for each changed contract — **Smart Contract Lead**
- [ ] Database migration requirements identified — **Backend Lead**
- [ ] Deployment window scheduled and communicated to team — **Release Manager**

### Security

- [ ] Security review completed for all contract changes — **Security Reviewer** (approval required)
- [ ] Dependency audit run: `cargo audit`, `npm audit` — **Backend Lead**
- [ ] Secret scan run: `scripts/check-secrets.sh` — **Backend Lead**
- [ ] No S0 or S1 secrets in diff — **Security Reviewer** (approval required)

### Quality

- [ ] All tests passing: `cargo test --all` — **Smart Contract Lead**
- [ ] All tests passing: `npm run lint && npm run typecheck && npm run build` — **Frontend Lead**
- [ ] Contract integration tests passing (cross-contract flows) — **Smart Contract Lead**
- [ ] Backend integration tests passing — **Backend Lead**
- [ ] Indexer replay test passing (staging) — **Backend Lead**

### Documentation

- [ ] `CHANGELOG.md` updated with all changes since last release — **Release Manager**
- [ ] `CONTRACT_REGISTRY.md` prepared with new contract entries (if deploying new contracts) — **Smart Contract Lead**
- [ ] `CONFIGURATION.md` updated with any new or changed environment variables — **Backend Lead**
- [ ] API documentation updated (if endpoint changes) — **Backend Lead**
- [ ] SDK documentation updated (if SDK changes) — **SDK Lead**

---

## Staging Deployment (3 Days Before Production)

### Smart Contracts

- [ ] Contracts built: `cargo build --release --target wasm32-unknown-unknown` — **Smart Contract Lead**
- [ ] WASM hashes recorded — **Smart Contract Lead**
- [ ] Contracts deployed to staging (testnet) — **Smart Contract Lead**
- [ ] Contracts initialized with staging configuration — **Smart Contract Lead**
- [ ] Contract IDs recorded in `CONTRACT_REGISTRY.md` — **Smart Contract Lead**
- [ ] Contracts verified on block explorer — **Smart Contract Lead**

### Backend

- [ ] Database backup taken before migration — **Backend Lead**
- [ ] Database migrations run: `alembic upgrade head` — **Backend Lead**
- [ ] Backend services deployed to staging — **Backend Lead**
- [ ] Health checks verified: `/health`, `/health/db`, `/health/redis` — **Backend Lead**
- [ ] Indexer confirmed processing events from new contracts — **Backend Lead**
- [ ] API endpoints smoke-tested — **Backend Lead**

### Frontend

- [ ] Frontend built with staging environment variables — **Frontend Lead**
- [ ] Frontend deployed to staging URL — **Frontend Lead**
- [ ] Wallet connection verified on staging — **Frontend Lead**
- [ ] Key user flows smoke-tested — **Frontend Lead**

### Integration

- [ ] End-to-end flow tested: deposit → stream → claim — **QA / Backend Lead**
- [ ] End-to-end flow tested: deposit → vest → claim — **QA / Backend Lead**
- [ ] End-to-end flow tested: proposal → vote → execute — **QA / Backend Lead**
- [ ] All flows tested with different wallet accounts — **QA / Backend Lead**

### Approval

- [ ] Release Manager signs off on staging results — **Release Manager** (approval required)
- [ ] Security Reviewer signs off on staging deployment — **Security Reviewer** (approval required for mainnet)

---

## Production Deployment (Release Day)

### 0. Pre-Flight (30 min before)

- [ ] Deployment window opened — **Release Manager**
- [ ] Team notified in deployment channel — **Release Manager**
- [ ] Monitoring dashboards confirmed operational (Grafana) — **Backend Lead**
- [ ] Rollback plan reviewed with team — **Release Manager**
- [ ] Production database backup taken — **Backend Lead**

### 1. Smart Contracts (if applicable)

- [ ] Contracts deployed to mainnet (see `DEPLOYMENT.md` section "Smart Contract Deployment") — **Smart Contract Lead**
- [ ] Contracts initialized with production configuration — **Smart Contract Lead**
- [ ] Contract IDs and deploy transactions recorded in `CONTRACT_REGISTRY.md` — **Smart Contract Lead**
- [ ] Contracts verified on block explorer — **Smart Contract Lead**
- [ ] Cross-contract references updated (e.g., governance → treasury) — **Smart Contract Lead**
- [ ] Deprecated contracts marked in registry — **Smart Contract Lead**

### 2. Database Migration

- [ ] Write traffic paused (if required by migration) — **Backend Lead**
- [ ] Database migrations run: `alembic upgrade head` — **Backend Lead**
- [ ] Migration verified: `alembic current` — **Backend Lead**
- [ ] Migration rollback tested in staging beforehand — **Backend Lead**
- [ ] **Decision point:** If migration fails, execute rollback per `DEPLOYMENT.md` — **Backend Lead + Release Manager**

### 3. Backend Deploy

- [ ] New backend image built and pushed — **Backend Lead**
- [ ] Services deployed: `docker compose up -d` — **Backend Lead**
- [ ] Health checks verified: all endpoints return 200 — **Backend Lead**
- [ ] Indexer confirmed processing events — **Backend Lead**
- [ ] Prometheus metrics confirmed scraping — **Backend Lead**
- [ ] **Decision point:** If health checks fail, rollback to previous image — **Backend Lead + Release Manager**

### 4. Frontend Deploy

- [ ] Environment variables verified in deployment platform — **Frontend Lead**
- [ ] Contract IDs in env vars match `CONTRACT_REGISTRY.md` mainnet entries — **Frontend Lead**
- [ ] Production build deployed — **Frontend Lead**
- [ ] **Decision point:** If build fails or regressions found, revert to previous deployment — **Frontend Lead + Release Manager**

### 5. Post-Deploy Verification

- [ ] Wallet connection works on production — **Frontend Lead**
- [ ] Treasury balance displays correctly — **Frontend Lead**
- [ ] Stream creation and claiming functional — **QA**
- [ ] Vesting progress displays correctly — **QA**
- [ ] Governance proposal flow functional — **QA**
- [ ] API endpoints returning correct data — **Backend Lead**
- [ ] Indexer lag < 5 minutes — **Backend Lead**
- [ ] No error spikes in Sentry or logs — **Backend Lead**
- [ ] Mobile breakpoints intact — **Frontend Lead**
- [ ] Maintenance mode disabled (if was enabled) — **Frontend Lead**

### 6. Release Finalization

- [ ] Release tagged in git: `git tag -a vX.Y.Z -m "Release vX.Y.Z"` — **Release Manager**
- [ ] Tag pushed: `git push origin vX.Y.Z` — **Release Manager**
- [ ] Release notes published (GitHub Releases) — **Release Manager**
- [ ] `CHANGELOG.md` committed with release date — **Release Manager**
- [ ] Deployment channel notified: "vX.Y.Z deployed successfully" — **Release Manager**
- [ ] Deployment window closed — **Release Manager**

---

## Post-Release (Within 24 Hours)

- [ ] Monitor Sentry/Prometheus for unexpected errors — **Backend Lead**
- [ ] Monitor indexer lag stability — **Backend Lead**
- [ ] Monitor on-chain activity for anomalies — **Smart Contract Lead**
- [ ] Check community channels for user-reported issues — **Frontend Lead**
- [ ] Update SDK package if contract interfaces changed — **SDK Lead**
- [ ] Update public documentation if changed — **Release Manager**
- [ ] Retrospective scheduled (if any issues during deploy) — **Release Manager**

---

## Release Approval Matrix

| Component | Dev Lead Sign-off | Security Review | Release Manager Approval |
|-----------|------------------|-----------------|-------------------------|
| Smart contracts (any change) | Smart Contract Lead | **Required** | **Required** |
| Smart contracts (patch only) | Smart Contract Lead | Required if auth or token logic changed | **Required** |
| Backend (new endpoint) | Backend Lead | Not required | Not required |
| Backend (auth or DB change) | Backend Lead | **Required** | **Required** |
| Frontend (UI only) | Frontend Lead | Not required | Not required |
| Frontend (wallet/tx signing) | Frontend Lead | **Required** | **Required** |
| SDK (new wrapper) | SDK Lead | Not required | Not required |
| SDK (breaking change) | SDK Lead | Not required | Required (semver) |

---

## Emergency Hotfix Procedure

For critical fixes that cannot wait for the full release cycle:

1. **Declare:** Release Manager declares emergency hotfix
2. **Fix:** Developer creates fix on a branch from the release tag
3. **Review:** Security Reviewer reviews if auth, token, or funds logic is touched
4. **Deploy:** Follow production deployment steps for the affected component only
5. **Verify:** Run post-deploy verification for the affected component
6. **Backport:** Merge hotfix back to `main` after verification
7. **Tag:** Tag the hotfix release (patch version bump)
8. **Postmortem:** If the bug reached production, follow incident postmortem process

Hotfix releases **skip** staging deployment but **require** Security Reviewer sign-off for any contract or auth-related changes. No hotfix may bypass the approval matrix above.
