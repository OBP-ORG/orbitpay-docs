# Configuration & Secret Management — OrbitPay

This document defines every environment variable, its validation rules, sensitivity classification, and ownership for all OrbitPay components.

## Sensitivity Classification

| Class | Label | Storage Rule |
|-------|-------|-------------|
| **S0 — Critical Secret** | Private keys, API signing secrets, DB master passwords | **Never** in source control, `.env` files, or developer machines. Must live in a secrets manager (e.g., Vault, AWS Secrets Manager, Doppler). Access requires MFA + approval. |
| **S1 — Secret** | API keys, JWT secrets, RPC auth tokens | Must not appear in source control. Stored in environment-specific `.env` managed by deployment platform or secrets manager. |
| **S2 — Sensitive** | Internal endpoints, non-public contract IDs (pre-launch) | Not in public source control. OK in private repos or deployment platform env vars. |
| **S3 — Public** | Public RPC URLs, public contract IDs, feature flags | Safe in source control and documentation. |

---

## Smart Contract Configuration

### Deployer Key

| Variable | Sensitivity | Validation | Owner |
|----------|-------------|------------|-------|
| `DEPLOYER_SECRET_KEY` | **S0** | Must be a valid Stellar secret key (starts with `S`). 56 characters. | Smart Contract Lead |

**Storage:** Hardware wallet or secrets manager. Never entered on a shared terminal.
**Rotation:** Rotate after any suspected exposure. Deploy new contracts from new key.

### Network Configuration

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `SOROBAN_NETWORK` | S3 | `testnet` | One of: `testnet`, `mainnet`, `standalone` |
| `SOROBAN_RPC_URL` | S3 | `https://soroban-testnet.stellar.org` | Must return 200 on `GET /` |
| `SOROBAN_NETWORK_PASSPHRASE` | S3 | *(network-dependent)* | Must match the Soroban network |

### Contract Admin Key

| Variable | Sensitivity | Validation | Owner |
|----------|-------------|------------|-------|
| `ADMIN_SECRET_KEY` | **S0** | Valid Stellar secret key | Multi-sig threshold signers collectively |

**The admin key should be a multi-sig address** formed from at least 3 signers with a threshold of 2+. Individual signer keys are S0.

---

## Backend Configuration

### Database

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `DATABASE_URL` | **S1** | — | `postgresql://user:pass@host:port/dbname` format |
| `DATABASE_HOST` | S2 | `localhost` | Valid hostname |
| `DATABASE_PORT` | S3 | `5432` | Valid port (1024–65535) |
| `DATABASE_NAME` | S2 | `orbitpay` | Non-empty string |
| `DATABASE_USER` | **S1** | `orbitpay` | Non-empty string |
| `DATABASE_PASSWORD` | **S0** | — | Min 16 chars; enforced by secrets manager |
| `DATABASE_POOL_MIN` | S3 | `2` | Integer ≥ 1 |
| `DATABASE_POOL_MAX` | S3 | `10` | Integer ≥ `DATABASE_POOL_MIN` |
| `DATABASE_SSL` | S3 | `true` (prod) / `false` (dev) | Boolean |

**DO NOT** set `DATABASE_PASSWORD` in any `.env` file. Use the full `DATABASE_URL` (which embeds the password) only in the secrets manager.

### Redis

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `REDIS_URL` | S2 | `redis://localhost:6379` | Valid Redis URL |
| `REDIS_PASSWORD` | **S1** | — | Required in staging/prod |
| `REDIS_CACHE_TTL_DEFAULT` | S3 | `60` | Integer, seconds |
| `REDIS_CACHE_TTL_TREASURY` | S3 | `60` | Integer, seconds |
| `REDIS_CACHE_TTL_STREAMS` | S3 | `30` | Integer, seconds |
| `REDIS_CACHE_TTL_PROPOSALS` | S3 | `30` | Integer, seconds |

### Stellar RPC / Indexer

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `STELLAR_RPC_URL` | S3 | Network-dependent | Must be a valid HTTPS URL |
| `STELLAR_NETWORK_PASSPHRASE` | S3 | Network-dependent | Exact match required |
| `INDEXER_POLL_INTERVAL_SEC` | S3 | `5` | Integer ≥ 1 |
| `INDEXER_BATCH_SIZE` | S3 | `100` | Integer 1–200 |
| `INDEXER_START_LEDGER` | S3 | `0` (latest) | Non-negative integer |
| `INDEXER_CURSOR_REDIS_KEY` | S3 | `orbitpay:indexer:cursor` | Non-empty string |

### Contract Addresses (Indexer)

| Variable | Sensitivity | Validation |
|----------|-------------|------------|
| `TREASURY_CONTRACT_ID` | S2 | Valid Stellar contract ID (56-char string starting with `C`) |
| `PAYROLL_STREAM_CONTRACT_ID` | S2 | Valid Stellar contract ID |
| `VESTING_CONTRACT_ID` | S2 | Valid Stellar contract ID |
| `GOVERNANCE_CONTRACT_ID` | S2 | Valid Stellar contract ID |

Pre-launch these are S2 (not public). Post-launch they become S3 and match the contract registry.

### API Server

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `API_HOST` | S3 | `0.0.0.0` | Valid IP |
| `API_PORT` | S3 | `8080` | Valid port |
| `API_CORS_ORIGINS` | S3 | `http://localhost:3000` | Comma-separated valid origins |
| `API_RATE_LIMIT_PER_MIN` | S3 | `100` | Integer ≥ 1 |
| `API_RATE_LIMIT_BURST` | S3 | `20` | Integer ≥ 1 |

### API Key Authentication

| Variable | Sensitivity | Validation | Owner |
|----------|-------------|------------|-------|
| `API_KEY_SECRET` | **S0** | Min 32 chars, used to sign/verify API keys | Backend Lead |
| `API_READ_ONLY_KEYS` | **S1** | Comma-separated hashed keys | Backend Lead |
| `API_READ_WRITE_KEYS` | **S0** | Comma-separated hashed keys | Backend Lead |

### Logging & Monitoring

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `LOG_LEVEL` | S3 | `INFO` | One of: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `LOG_FORMAT` | S3 | `json` | One of: `json`, `text` |
| `PROMETHEUS_METRICS_PORT` | S3 | `9090` | Valid port |
| `SENTRY_DSN` | **S1** | — | Valid Sentry DSN URL |
| `GRAFANA_ADMIN_PASSWORD` | **S0** | — | Min 12 chars |

---

## Frontend Configuration

All frontend environment variables are prefixed with `NEXT_PUBLIC_` and are therefore exposed to the browser. **Never include secrets in frontend env vars.**

### Network

| Variable | Sensitivity | Validation |
|----------|-------------|------------|
| `NEXT_PUBLIC_STELLAR_NETWORK` | S3 | `TESTNET` or `PUBLIC` |
| `NEXT_PUBLIC_SOROBAN_RPC_URL` | S3 | Valid HTTPS URL |
| `NEXT_PUBLIC_NETWORK_PASSPHRASE` | S3 | Network-dependent exact string |

### Contract IDs

| Variable | Sensitivity | Validation |
|----------|-------------|------------|
| `NEXT_PUBLIC_TREASURY_CONTRACT_ID` | S3 | Valid Stellar contract ID |
| `NEXT_PUBLIC_PAYROLL_STREAM_CONTRACT_ID` | S3 | Valid Stellar contract ID |
| `NEXT_PUBLIC_VESTING_CONTRACT_ID` | S3 | Valid Stellar contract ID |
| `NEXT_PUBLIC_GOVERNANCE_CONTRACT_ID` | S3 | Valid Stellar contract ID |

### API Backend

| Variable | Sensitivity | Validation |
|----------|-------------|------------|
| `NEXT_PUBLIC_API_BASE_URL` | S3 | Valid HTTPS URL |
| `NEXT_PUBLIC_API_KEY` | **S1** | Read-only API key (browser-visible, minimal scope) |

`NEXT_PUBLIC_API_KEY` should be a **read-only** key with the narrowest possible scope. It will be visible in browser devtools.

### Feature Flags

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `NEXT_PUBLIC_ENABLE_GOVERNANCE` | S3 | `false` | Boolean |
| `NEXT_PUBLIC_ENABLE_BATCH_PAYROLL` | S3 | `false` | Boolean |
| `NEXT_PUBLIC_ENABLE_MAINNET` | S3 | `false` | Boolean |
| `NEXT_PUBLIC_MAINTENANCE_MODE` | S3 | `false` | Boolean |

### Analytics

| Variable | Sensitivity | Validation |
|----------|-------------|------------|
| `NEXT_PUBLIC_ANALYTICS_ID` | S3 | Valid analytics measurement ID (optional) |

---

## SDK Configuration

| Variable | Sensitivity | Default | Validation |
|----------|-------------|---------|------------|
| `ORBITPAY_RPC_URL` | S3 | Network-dependent | Valid HTTPS URL |
| `ORBITPAY_NETWORK_PASSPHRASE` | S3 | Network-dependent | Exact match |
| `ORBITPAY_CONTRACT_IDS` | S3 | *(from registry)* | JSON map of contract name → ID |

---

## Docker Compose Environment

### `.env.example` Template (checked into repo)

```bash
# === Network ===
SOROBAN_NETWORK=testnet
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
STELLAR_NETWORK_PASSPHRASE="Test SDF Network ; September 2015"

# === Database (dev defaults — DO NOT use in prod) ===
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_NAME=orbitpay
DATABASE_USER=orbitpay
# DATABASE_PASSWORD=<SET IN SECRETS MANAGER — NEVER HERE>

# === Redis (dev defaults — DO NOT use in prod) ===
REDIS_URL=redis://redis:6379
# REDIS_PASSWORD=<SET IN SECRETS MANAGER — NEVER HERE>

# === API ===
API_PORT=8080
API_CORS_ORIGINS=http://localhost:3000
LOG_LEVEL=INFO
LOG_FORMAT=json

# === Contract IDs (populated after deployment) ===
TREASURY_CONTRACT_ID=
PAYROLL_STREAM_CONTRACT_ID=
VESTING_CONTRACT_ID=
GOVERNANCE_CONTRACT_ID=
```

### Production Environment

Production variables are stored in the deployment platform's secrets manager. The `.env` file is **never** created in production. Variables are injected at container runtime.

---

## Secret Ownership

| Secret | Primary Owner | Backup Owner | Rotation Cadence |
|--------|--------------|-------------|------------------|
| Deployer secret key | Smart Contract Lead | Release Manager | Per deploy or quarterly |
| Admin secret key (contract) | Multi-sig signers | Release Manager | Annually or on signer change |
| Database master password | Backend Lead | Release Manager | Quarterly |
| Redis password | Backend Lead | Release Manager | Quarterly |
| API key signing secret | Backend Lead | Release Manager | Quarterly |
| API read-write keys | Backend Lead | Release Manager | On team change |
| Grafana admin password | Backend Lead | Release Manager | Quarterly |
| Sentry DSN | Backend Lead | Frontend Lead | On project change |

---

## Secret Validation Script

Run before any deployment to detect leaked secrets:

```bash
#!/bin/bash
# scripts/check-secrets.sh — scan for potential secrets in the repo
set -euo pipefail

echo "=== Scanning for Stellar secret keys (S*) ==="
if grep -r "S[A-Z0-9]\{55\}" . --exclude-dir=.git --exclude-dir=target --exclude-dir=node_modules; then
    echo "ERROR: Potential Stellar secret key found in source!"
    exit 1
fi

echo "=== Scanning for common secret patterns ==="
PATTERNS=(
    "DATABASE_PASSWORD="
    "REDIS_PASSWORD="
    "ADMIN_SECRET_KEY="
    "DEPLOYER_SECRET_KEY="
    "API_KEY_SECRET="
    "GRAFANA_ADMIN_PASSWORD="
    "SENTRY_DSN="
)
for pattern in "${PATTERNS[@]}"; do
    if grep -r "$pattern" . --exclude-dir=.git --exclude-dir=target --exclude-dir=node_modules \
        --exclude=CONFIGURATION.md --exclude=.env.example 2>/dev/null; then
        echo "WARNING: Pattern '$pattern' found in repo. Verify it is not a real secret."
    fi
done

echo "=== Secret scan complete ==="
```

Add this as a pre-commit hook and CI check to enforce the "no production secrets in source control" gate.
