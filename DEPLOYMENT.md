# Deployment Runbook — OrbitPay

This runbook covers deployment procedures for all OrbitPay components across every environment: local development, Stellar testnet, staging, and mainnet.

## Responsible Roles

| Role | Responsibility | Approval Required For |
|------|---------------|-----------------------|
| **Release Manager** | Oversees the release; opens deployment window | All mainnet deploys |
| **Smart Contract Lead** | Deploys Soroban contracts; verifies on-chain | Contract deploy / upgrade |
| **Backend Lead** | Deploys API + Indexer; runs DB migrations | Backend deploy |
| **Frontend Lead** | Deploys dashboard; verifies build | Frontend deploy |
| **Security Reviewer** | Reviews diff for secret leaks, auth changes, token handling | All mainnet deploys |

No single person may authorize and execute a mainnet deployment alone — production deploys require Release Manager approval and Security Reviewer sign-off.

---

## Environment Topology

### Local

| Component | Host | Notes |
|-----------|------|-------|
| Soroban contracts | `soroban-cli` local network (`standalone`) | `soroban network start` |
| PostgreSQL | `localhost:5432` | Docker Compose |
| Redis | `localhost:6379` | Docker Compose |
| API + Indexer | `localhost:8080` | `npm run dev` / `uvicorn` |
| Frontend | `localhost:3000` | `npm run dev` |

### Testnet (Stellar Testnet)

| Component | Details |
|-----------|---------|
| Network passphrase | `Test SDF Network ; September 2015` |
| RPC endpoint | `https://soroban-testnet.stellar.org` |
| Friendbot | `https://friendbot.stellar.org` |
| Block explorer | `https://stellar.expert/explorer/testnet` |
| API + Indexer | VPS / cloud VM (see `CONFIGURATION.md` for host) |
| Frontend | Vercel / Netlify preview deploy |

### Staging

| Component | Details |
|-----------|---------|
| Network | Stellar Testnet (separate contract instances from dev testnet) |
| API + Indexer | Staging VPS (dedicated) |
| PostgreSQL | Staging DB instance (dedicated) |
| Redis | Staging Redis instance (dedicated) |
| Frontend | Vercel preview / staging subdomain |
| Purpose | Mirror of production for rehearsal; all deploy steps dry-run here first |

### Mainnet (Stellar Public)

| Component | Details |
|-----------|---------|
| Network passphrase | `Public Global Stellar Network ; September 2015` |
| RPC endpoint | `https://soroban.stellar.org` (or dedicated archive node) |
| Block explorer | `https://stellar.expert/explorer/public` |
| API + Indexer | Production VPS / cloud |
| PostgreSQL | Managed DB (with replicas, automated backups) |
| Redis | Managed Redis (with persistence) |
| Frontend | Production Vercel / Netlify deployment |
| Monitoring | Prometheus + Grafana (see `INCIDENT_RESPONSE.md`) |

---

## Pre-Deployment Checklist

- [ ] All tests pass (`cargo test --all`, `npm run lint`, `npm run build`)
- [ ] Security review completed for changed contracts
- [ ] Database migration scripts reviewed and tested in staging
- [ ] Release notes drafted
- [ ] Rollback plan documented
- [ ] Deployment window communicated to team
- [ ] Monitoring dashboards confirmed operational
- [ ] Release Manager approval obtained (mainnet only)
- [ ] `CONTRACT_REGISTRY.md` prepared with new contract IDs/hashes

---

## Smart Contract Deployment

### 1. Build WASM Artifacts

```bash
cd contracts
cargo build --release --target wasm32-unknown-unknown
# Output: target/wasm32-unknown-unknown/release/<contract>.wasm
```

### 2. Verify Build Reproducibility

```bash
# Record the hash of each WASM artifact
sha256sum target/wasm32-unknown-unknown/release/*.wasm > deployment-hashes.txt
```

### 3. Deploy (Testnet)

```bash
# Deploy each contract. Order matters: Treasury first (dependencies reference it).
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/treasury.wasm \
  --network testnet \
  --source <DEPLOYER_SECRET_KEY>

# Save the contract ID; repeat for payroll_stream, vesting, governance
```

### 4. Initialize Contracts

```bash
# Treasury: admin = deployer, signers = [], threshold = 1 (for bootstrapping)
soroban contract invoke \
  --id <TREASURY_ID> \
  --network testnet \
  --source <DEPLOYER_SECRET_KEY> \
  -- initialize --admin <ADMIN_ADDRESS> --signers '[]' --threshold 1

# Repeat initialization for each contract per its constructor
```

### 5. Update Contract Registry

After deployment, immediately update `CONTRACT_REGISTRY.md` with:
- Network (testnet / mainnet)
- Contract name
- Contract ID
- WASM hash
- Deployment transaction hash
- Deployer address
- Timestamp

### 6. Verify on Explorer

Open each contract ID on the block explorer and verify:
- WASM bytecode is uploaded
- Initialization was successful
- Contract metadata matches expected

---

## Contract Upgrade Procedure

OrbitPay contracts are **immutable** — upgrades deploy a new contract instance. Migration steps:

### 1. Deploy New Contract

Follow the standard deployment steps above for the new version.

### 2. State Migration (if applicable)

If state must be carried forward:
1. Deploy a **migration helper** contract
2. Read state from old contract via cross-contract call
3. Write state to new contract
4. Verify migrated state integrity

### 3. Update Dependent Contracts

For any contract referencing the upgraded contract's address:

```bash
# Example: Update governance to reference new treasury
soroban contract invoke \
  --id <GOVERNANCE_ID> \
  --network mainnet \
  --source <ADMIN_SECRET_KEY> \
  -- set_treasury --treasury <NEW_TREASURY_ID>
```

### 4. Update All References

- [ ] Update `CONTRACT_REGISTRY.md`
- [ ] Update frontend environment config (`NEXT_PUBLIC_*_CONTRACT_ID`)
- [ ] Update backend indexer contract address filter
- [ ] Update SDK contract ID constants
- [ ] Deprecate old contract (emit `Deprecated` event, halt new interactions)

---

## Backend (API + Indexer) Deployment

### 1. Pre-Deploy

```bash
cd backend
# Verify Docker Compose configuration
docker compose config
# Pull latest base images
docker compose pull
```

### 2. Database Migration

```bash
# Run migrations (always run before deploying new code)
docker compose run --rm api alembic upgrade head

# Verify migration
docker compose run --rm api alembic current
```

### 3. Deploy Services

```bash
# Build and start all services
docker compose up -d --build

# Verify all services healthy
docker compose ps
docker compose logs --tail=50 api indexer
```

### 4. Post-Deploy Verification

```bash
# Health check endpoints
curl -f http://localhost:8080/health
curl -f http://localhost:8080/health/db
curl -f http://localhost:8080/health/redis

# Verify indexer is processing events
curl http://localhost:8080/api/health/indexer | jq '.lag_seconds'
```

### 5. Indexer Replay (if needed)

To replay events from a specific ledger:

```bash
# Set cursor to target ledger in Redis
docker compose exec redis redis-cli SET orbitpay:indexer:cursor <LEDGER_SEQUENCE>

# Restart indexer — it will replay from cursor
docker compose restart indexer

# Monitor lag until caught up
watch -n 5 'curl -s http://localhost:8080/api/health/indexer | jq'
```

---

## Database Restore Procedure

### 1. Identify Restore Point

Determine the backup to restore from the backup catalog. Staging should be drilled with the most recent backup quarterly.

### 2. Stop Write Traffic

```bash
# Scale API to 0 (or pause indexer writes)
docker compose stop api indexer
```

### 3. Restore from Backup

```bash
# Drop existing database (staging only — prod uses separate restore target)
docker compose exec postgres dropdb -U orbitpay orbitpay --if-exists
docker compose exec postgres createdb -U orbitpay orbitpay

# Restore from pg_dump
docker compose exec -T postgres pg_restore -U orbitpay -d orbitpay < backup.dump
```

### 4. Run Pending Migrations

```bash
docker compose run --rm api alembic upgrade head
```

### 5. Replay Indexer (if gap exists)

Set the indexer cursor to the ledger corresponding to the backup's last event timestamp, then replay forward.

### 6. Resume Traffic

```bash
docker compose up -d api indexer
```

### 7. Verify

- [ ] API health checks pass
- [ ] Indexer lag returning to acceptable threshold
- [ ] Frontend dashboard loads data correctly
- [ ] Recent on-chain events visible in API

---

## Frontend Deployment

### 1. Build

```bash
cd frontend
npm ci
npm run build
```

### 2. Verify Build

```bash
# Confirm no build errors
npm run lint
npm run typecheck

# Smoke test static export (if applicable)
npx serve out/
```

### 3. Deploy

Frontend is deployed via Vercel (or Netlify). Environment variables are configured in the platform dashboard, not in repository files.

### 4. Post-Deploy Verification

- [ ] Load dashboard; verify wallet connects to correct network
- [ ] Treasury overview shows correct balance
- [ ] Stream list loads with correct data
- [ ] Vesting schedules render properly
- [ ] Governance proposals display correctly
- [ ] No console errors
- [ ] Mobile responsive breakpoints intact

---

## Rollback Decision Matrix

| Scenario | Rollback Action | Decision Maker |
|----------|----------------|----------------|
| Contract has bug, no funds at risk | Deploy fix as new contract instance; update references | Smart Contract Lead |
| Contract has bug, funds at risk | Pause interactions (admin emergency pause if implemented); deploy fix | Release Manager + Smart Contract Lead |
| Backend deploy fails health checks | `docker compose up -d` with previous image tag | Backend Lead |
| Database migration fails | `alembic downgrade -1`; restore pre-migration DB snapshot | Backend Lead |
| Frontend deploy has regression | Revert Vercel deployment to previous production build | Frontend Lead |
| Indexer corruption | Stop indexer; DB restore; indexer replay from safe cursor | Backend Lead |
| Critical vulnerability discovered | Full incident response (see `INCIDENT_RESPONSE.md`) | Release Manager |

### Contract Rollback

Since contracts are immutable, "rollback" means:
1. Identify the last known-good contract ID from `CONTRACT_REGISTRY.md`
2. Update all references (other contracts, frontend, backend, SDK) to point to previous contract ID
3. If funds were transferred to the buggy contract, develop and execute a recovery transaction
4. Mark the buggy contract as deprecated in the registry

---

## Recovery Objectives

| Metric | Target |
|--------|--------|
| **RTO** (Recovery Time Objective) | < 4 hours for critical incidents |
| **RPO** (Recovery Point Objective) | < 1 hour of indexer data loss (replayable from chain) |
| **DB backup frequency** | Daily automated; on-demand before each deploy |
| **DB backup retention** | 30 days |
| **Indexer replay verification** | Drilled in staging quarterly |

---

## Staging Drill Schedule

| Drill | Frequency | Owner |
|-------|-----------|-------|
| Full deployment from scratch (unfamiliar maintainer) | Monthly | Backend Lead |
| Database restore + indexer replay | Quarterly | Backend Lead |
| Contract deploy + initialize | Per release | Smart Contract Lead |
| Rollback procedure | Quarterly | Release Manager |
| Incident response simulation | Quarterly | Entire team |

The staging drill must be completable by an unfamiliar maintainer following only this runbook — if not, the runbook is incomplete and must be updated.

---

## Emergency Contacts

| Role | Contact Method | Escalation (if no response in 15 min) |
|------|---------------|---------------------------------------|
| Release Manager | Signal / Telegram primary | Phone call |
| Smart Contract Lead | Signal / Telegram primary | Phone call |
| Backend Lead | Signal / Telegram primary | Phone call |
| Security Reviewer | Signal / Telegram primary | Phone call |

*(Contact details maintained in team's password manager, never in this repository.)*
