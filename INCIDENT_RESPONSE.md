# Incident Response Runbook — OrbitPay

This runbook defines the process for detecting, triaging, responding to, and learning from production incidents affecting the OrbitPay protocol.

## Severity Definitions

### SEV-0 — Critical

**Definition:** Protocol funds at risk, contract vulnerability exploitable, or complete service outage affecting all users.

**Examples:**
- Unauthorized withdrawal possible from treasury contract
- Governance bypass allowing single-signer fund movement
- RPC endpoint serving incorrect/stale data causing financial loss
- Indexer halted with > 30 min lag, dashboards showing wrong balances
- Database breach exposing PII or API keys

**Response time:** Immediate (any hour, any day)
**Notification:** War room (all-hands), all stakeholders within 15 minutes

### SEV-1 — High

**Definition:** Major feature broken, partial outage, or degraded service affecting a significant percentage of users.

**Examples:**
- Frontend unable to connect to wallet for > 50% of users
- API returning 5xx for any payment-critical endpoint
- Indexer lag > 10 minutes
- Contract interaction failing for all users on a specific function

**Response time:** < 1 hour
**Notification:** On-call engineer + component lead within 15 minutes

### SEV-2 — Medium

**Definition:** Non-critical feature degraded, minor data inconsistency, or issue affecting a small subset of users.

**Examples:**
- Vesting timeline visualization rendering incorrectly (data accurate, display wrong)
- API rate limiting more aggressive than configured
- Specific proposal detail page failing to load
- Redis cache stale beyond TTL

**Response time:** < 4 hours (business hours)
**Notification:** Engineering team Slack channel

### SEV-3 — Low

**Definition:** Cosmetic issues, non-blocking bugs, or preemptive concerns.

**Examples:**
- UI misalignment on specific viewport
- Warning logs spiking without user impact
- Documentation out of date

**Response time:** Next business day
**Notification:** File GitHub issue

---

## Incident Commander Role

For SEV-0 and SEV-1 incidents, an **Incident Commander (IC)** is designated. The IC:

1. Declares the incident and severity
2. Opens the incident channel / war room
3. Assigns responders to investigation and mitigation tracks
4. Maintains the incident timeline
5. Communicates status to stakeholders every 30 minutes
6. Decides when to escalate severity
7. Declares incident resolved

The first responder on scene acts as IC until relieved.

**Default IC order:** Backend Lead → Smart Contract Lead → Frontend Lead

---

## Notification Templates

### Initial Notification (SEV-0 / SEV-1)

```
INCIDENT: OrbitPay [SEV-0/SEV-1]

Severity: [SEV-0 | SEV-1]
Status: [Investigating | Mitigating | Monitoring]
Time Detected: [YYYY-MM-DD HH:MM UTC]
Incident Commander: [Name]
Channel: [War room link / Signal group]

Summary: [1-2 sentence description of what is observed]

Impact: [Which components/users are affected]

Next update: [Time, typically 30 min from now]
```

### Status Update (every 30 min)

```
UPDATE: OrbitPay Incident #[ID]

Time: [HH:MM UTC]
Status: [Investigating | Mitigating | Monitoring]
Elapsed: [Xh Ym]

Progress since last update:
- [Bullet points of actions taken]
- [Bullet points of findings]

Current hypothesis: [What we think is happening]

Next steps:
- [Immediate actions]

Next update: [HH:MM UTC]
```

### Resolution Notification

```
RESOLVED: OrbitPay Incident #[ID]

Severity: [SEV-X]
Duration: [Xh Ym] (detected → resolved)
Root cause: [1-2 sentence summary]

Impact summary:
- Users affected: [count or percentage]
- Funds at risk: [amount or "none"]
- Data loss: [description or "none"]

Mitigation: [What was done to resolve]

Follow-up:
- Postmortem scheduled: [date/time]
- Tracking issue: [link]
```

---

## Incident Response Procedures

### 1. Detect

Incidents may be detected via:
- **Automated alerts:** Prometheus alert rules, Sentry error spikes, health check failures
- **User reports:** GitHub issues, community channels, direct reports
- **Proactive monitoring:** Dashboard anomaly review, indexer lag monitoring

### 2. Declare

The first responder:
1. Acknowledges the alert or report
2. Assesses severity against the definitions above
3. If SEV-0 or SEV-1: declares incident and assumes IC role
4. If unsure between severities: **declare the higher severity** — downgrade later if warranted
5. Opens incident channel and sends initial notification

### 3. Investigate & Contain

**SEV-0 / Smart Contract Vulnerability:**
1. Immediately assess: can funds be drained? Is the exploit actively occurring?
2. If exploit is active and contract has pause mechanism: execute emergency pause
3. If no pause mechanism exists: notify all signers; coordinate rapid governance action
4. Monitor on-chain for any suspicious transactions involving OrbitPay contracts
5. If user funds at risk on frontend: enable `NEXT_PUBLIC_MAINTENANCE_MODE=true`

**SEV-0 / SEV-1 — Backend:**
1. Check monitoring dashboards: Prometheus metrics, Grafana, logs
2. Identify failing component: API, Indexer, DB, Redis
3. If DB issue: check connection pool, disk space, replication lag
4. If Indexer issue: check cursor position, RPC connectivity, event parsing errors
5. If Redis issue: check memory, eviction policy, connectivity
6. Restart affected services if safe (check for in-flight DB migrations first)

**SEV-1 — Frontend:**
1. Check Vercel/Netlify deployment status
2. Verify environment variables in deployment platform
3. Check API backend is reachable from frontend
4. Verify contract IDs match the registry
5. Check browser console for client-side errors
6. Rollback to last known-good deployment if needed

### 4. Mitigate

The goal is to stop user impact, not necessarily to fix the root cause:
- Rollback to previous deployment
- Scale down/stop affected service
- Enable maintenance mode
- Rotate compromised credentials
- Deploy hotfix if low-risk
- Block specific IPs or actors if under attack

### 5. Verify

Before resolving:
- [ ] Affected endpoints/services confirmed working
- [ ] Monitoring dashboards show normal metrics
- [ ] Indexer lag back within acceptable threshold (< 5 min)
- [ ] Key user flows smoke-tested (wallet connect, deposit, stream claim)
- [ ] No active exploit transactions in recent blocks
- [ ] If credentials were rotated, old credentials confirmed revoked

### 6. Resolve

- Send resolution notification
- Close war room / incident channel (keep read-only for postmortem reference)
- Create postmortem tracking issue
- Schedule postmortem meeting (within 5 business days for SEV-0, 10 for SEV-1)

---

## Evidence Collection

During an incident, preserve the following evidence:

| Evidence Type | How to Collect | Retention |
|--------------|----------------|-----------|
| Application logs | `docker compose logs --tail=2000 <service> > incident-<id>-logs.txt` | 90 days |
| Database snapshot | `pg_dump` of relevant tables at time of incident | 90 days |
| Redis state | `redis-cli --rdb incident-<id>-dump.rdb` | 30 days |
| On-chain transactions | Links to block explorer for all related tx hashes | Permanent (chain) |
| Prometheus metrics | Screenshot / Grafana snapshot of relevant dashboards | 90 days |
| Communication log | Export incident channel messages | 90 days |
| Timeline | IC maintains a chronological log of all actions/times | Permanent (in postmortem) |

---

## Postmortem Process

### Timeline

| Event | Deadline |
|-------|----------|
| Incident resolved | — |
| Postmortem issue created | Within 24 hours of resolution |
| Draft postmortem circulated | Within 3 business days (SEV-0) / 5 business days (SEV-1) |
| Postmortem meeting held | Within 5 business days (SEV-0) / 10 business days (SEV-1) |
| Action items assigned | During postmortem meeting |
| Action items completed | Per agreed deadlines in postmortem |

### Postmortem Template

```markdown
# Postmortem: [Incident Title]

**Date:** [YYYY-MM-DD]
**Severity:** [SEV-0 | SEV-1 | SEV-2]
**Duration:** [Xh Ym] (detected → resolved)
**Incident Commander:** [Name]
**Postmortem Owner:** [Name]

## Summary
[2-3 sentence description of what happened and impact]

## Timeline (UTC)
| Time | Event |
|------|-------|
| HH:MM | Incident detected via [alert/user report] |
| HH:MM | IC declared; war room opened |
| HH:MM | Root cause identified as [X] |
| HH:MM | Mitigation deployed |
| HH:MM | Verified resolution |
| HH:MM | Incident resolved |

## Root Cause Analysis
[Detailed technical explanation of the root cause]

## What Went Well
- [Things the team did effectively]

## What Went Wrong
- [Gaps in detection, response, or process]

## Impact
- **Users affected:** [count or %]
- **Financial impact:** [amount or "none"]
- **Data loss:** [description or "none"]
- **Reputational:** [assessment]

## Action Items
| # | Action | Owner | Deadline | Status |
|---|--------|-------|----------|--------|
| 1 | [Specific, actionable item] | @name | YYYY-MM-DD | ⬜ |
| 2 | [Specific, actionable item] | @name | YYYY-MM-DD | ⬜ |

## Lessons Learned
[Key takeaways to share with the broader team]
```

### Postmortem Meeting Agenda

1. Review timeline — ensure everyone agrees on the sequence of events
2. Walk through root cause analysis — is the diagnosis complete and correct?
3. Discuss what went well — recognize effective responses
4. Discuss what went wrong — blameless, focused on process and systems
5. Review action items — assign owners, set realistic deadlines
6. Identify whether runbooks need updating based on the incident

---

## Recovery Objectives

### Measured Metrics

| Metric | Target | How Measured |
|--------|--------|-------------|
| **Mean Time to Detect (MTTD)** | < 15 min (SEV-0), < 30 min (SEV-1) | Time from incident start to first alert/declaration |
| **Mean Time to Respond (MTTR)** | < 4 hours (SEV-0), < 8 hours (SEV-1) | Time from declaration to resolution |
| **Recovery Time Objective (RTO)** | < 4 hours | Maximum acceptable downtime |
| **Recovery Point Objective (RPO)** | < 1 hour | Maximum acceptable data loss (indexer replayable) |

### Measurement Method

- Incident timeline recorded by IC during incident
- Reviewed and validated during postmortem
- Tracked in incident metrics dashboard (Grafana)
- Quarterly review of MTTD/MTTR trends

---

## Alerting Rules

### Prometheus Alert Rules

```yaml
groups:
  - name: orbitpay_critical
    rules:
      - alert: IndexerLagHigh
        expr: orbitpay_indexer_lag_seconds > 600
        for: 5m
        labels:
          severity: sev1
        annotations:
          summary: "Indexer lag exceeds 10 minutes"
          description: "Indexer is {{ $value }} seconds behind. Contract events may not appear in API."

      - alert: IndexerDown
        expr: up{job="orbitpay-indexer"} == 0
        for: 2m
        labels:
          severity: sev0
        annotations:
          summary: "Indexer service is down"
          description: "Indexer has been down for more than 2 minutes. No events are being processed."

      - alert: ApiHighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: sev1
        annotations:
          summary: "API 5xx error rate > 5%"
          description: "API error rate is {{ $value | humanizePercentage }}."

      - alert: DatabaseConnectionPoolExhausted
        expr: orbitpay_db_pool_available < 1
        for: 1m
        labels:
          severity: sev0
        annotations:
          summary: "Database connection pool exhausted"
          description: "No available DB connections. API cannot serve requests."

      - alert: RedisDown
        expr: up{job="orbitpay-redis"} == 0
        for: 2m
        labels:
          severity: sev1
        annotations:
          summary: "Redis is down"
          description: "Cache unavailable. API may experience degraded performance."

      - alert: ContractEventAnomaly
        expr: rate(orbitpay_contract_events_total[15m]) < 1
        for: 30m
        labels:
          severity: sev2
        annotations:
          summary: "Low contract event volume"
          description: "No contract events in 30 minutes. Possible RPC or contract issue."
```

### Health Check Endpoints

Monitor these endpoints from an external probe (e.g., UptimeRobot, Better Uptime):

| Endpoint | Expected | Alert if |
|----------|----------|---------|
| `GET /health` | 200, `{"status": "ok"}` | Non-200 or timeout |
| `GET /health/db` | 200, `{"database": "connected"}` | Non-200 |
| `GET /health/redis` | 200, `{"redis": "connected"}` | Non-200 |
| `GET /api/health/indexer` | 200, `{"lag_seconds": < 300}` | Non-200 or `lag_seconds > 600` |

---

## Emergency Contact Matrix

*(Actual contact details stored in team password manager. This documents the escalation path.)*

| Level | Channel | Expected Response |
|-------|---------|-------------------|
| Primary | On-call rotation in PagerDuty / Opsgenie | < 5 min |
| Secondary | Team Signal group / Telegram | < 15 min |
| Escalation | Phone call to Release Manager | < 30 min |
| Last resort | Phone call to all maintainers | < 60 min |

### On-Call Rotation

On-call rotates weekly between Backend Lead and Smart Contract Lead. The on-call engineer:
- Carries the pager/notification device
- Has production access
- Knows how to follow this runbook
- Can escalate to IC if needed

---

## Runbook Maintenance

This runbook must be:
- Reviewed and updated after every SEV-0 or SEV-1 incident
- Reviewed quarterly as part of the incident response drill
- Versioned in the `orbitpay-docs` repository
- Accessible to all maintainers (not behind production auth)

Any procedure found incomplete or incorrect during a real incident or drill must be updated within 48 hours of the postmortem.
