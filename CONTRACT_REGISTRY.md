# Contract Registry — OrbitPay

This document is the authoritative record of every OrbitPay smart contract deployed across all networks. It tracks contract IDs, WASM hashes, versions, deployment transactions, and signer configurations.

## How to Update

After every contract deployment (new or upgrade):

1. Add a new row to the appropriate network table below
2. Mark deprecated contracts with ~~strikethrough~~ and note the successor ID
3. Record the deploy transaction hash for on-chain verification
4. Commit this file — it is the single source of truth for contract addresses

---

## Stellar Testnet

| Contract | Version | Contract ID | WASM Hash (SHA-256) | Deploy Tx Hash | Deployer | Deployed At | Status |
|----------|---------|-------------|---------------------|----------------|----------|-------------|--------|
| Treasury | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |
| Payroll Stream | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |
| Vesting | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |
| Governance | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |

## Staging (Testnet — Separate Instances)

| Contract | Version | Contract ID | WASM Hash (SHA-256) | Deploy Tx Hash | Deployer | Deployed At | Status |
|----------|---------|-------------|---------------------|----------------|----------|-------------|--------|
| Treasury | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |
| Payroll Stream | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |
| Vesting | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |
| Governance | `0.1.0` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Active |

## Stellar Mainnet

| Contract | Version | Contract ID | WASM Hash (SHA-256) | Deploy Tx Hash | Deployer | Deployed At | Status |
|----------|---------|-------------|---------------------|----------------|----------|-------------|--------|
| Treasury | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Not deployed |
| Payroll Stream | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Not deployed |
| Vesting | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Not deployed |
| Governance | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | `TBD` | Not deployed |

---

## Contract Specifications

### Treasury

```
Purpose: Multi-signature treasury vault for secure fund management.

Storage:
  - Admin address
  - Signer set (Vec<Address>)
  - Approval threshold (u32)
  - Withdrawal proposals (persistent map)
  - Proposal counter

Public Functions:
  - initialize(admin, signers, threshold)
  - deposit(from, token, amount)
  - create_withdrawal(signer, token, recipient, amount, memo)
  - approve_withdrawal(signer, proposal_id)
  - execute_withdrawal(proposal_id)
  - cancel_withdrawal(caller, proposal_id)
  - get_config()
  - get_balance(token)
  - get_withdrawal(proposal_id)
  - add_signer(admin, new_signer)
  - remove_signer(admin, signer)
  - update_threshold(admin, new_threshold)

Events (9):
  - TreasuryInitialized, TreasuryDeposit, WithdrawalCreated,
    WithdrawalApproved, WithdrawalExecuted, WithdrawalCancelled,
    SignerAdded, SignerRemoved, ThresholdUpdated

Dependencies: Soroban token contract (any SEP-41 token)
```

### Payroll Stream

```
Purpose: Continuous payment streaming with per-second accrual.

Storage:
  - Stream records (persistent map, keyed by stream ID)
  - Stream counter
  - Token balances per stream

Public Functions:
  - create_stream(sender, recipient, token, amount, start, end)
  - create_batch_streams(sender, streams)
  - claim(recipient, stream_id)
  - cancel_stream(sender, stream_id)
  - pause_stream(sender, stream_id)
  - resume_stream(sender, stream_id)
  - get_stream(stream_id)
  - get_claimable(stream_id)
  - calculate_claimable(stream_id)

Events (5):
  - StreamCreated, StreamClaimed, StreamCancelled,
    StreamPaused, StreamResumed

Dependencies: Soroban token contract (any SEP-41 token)
```

### Vesting

```
Purpose: Cliff + linear vesting schedules for team, advisor, and investor tokens.

Storage:
  - Vesting schedules (persistent map, keyed by schedule ID)
  - Schedule counter
  - Claim history per schedule

Public Functions:
  - create_schedule(grantor, beneficiary, token, total_amount, cliff_amount,
                     start, cliff_duration, total_duration, revocable)
  - claim(beneficiary, schedule_id)
  - revoke(grantor, schedule_id)
  - get_schedule(schedule_id)
  - get_progress(schedule_id)
  - get_claim_history(schedule_id)

Events (4):
  - VestingCreated, VestingClaimed, VestingRevoked, VestingFullyClaimed

Dependencies: Soroban token contract (any SEP-41 token)
```

### Governance

```
Purpose: DAO-style proposal creation, voting, and execution.

Storage:
  - Member set (Vec<Address>)
  - Voting weights (Map<Address, u32>)
  - Proposals (persistent map)
  - Votes per proposal (persistent map)
  - Governance config (treasury address, grace period)

Public Functions:
  - create_proposal(proposer, title, token, recipient, amount, justification)
  - vote(voter, proposal_id, choice)
  - finalize(proposal_id)
  - execute(proposal_id)
  - cancel_proposal(proposer, proposal_id)
  - get_proposal(proposal_id)
  - get_proposal_status(proposal_id)
  - add_member(admin, member)
  - remove_member(admin, member)
  - set_voting_weight(admin, member, weight)
  - set_treasury(admin, treasury_address)

Events (5):
  - ProposalCreated, VoteCast, ProposalFinalized,
    ProposalExecuted, ProposalCancelled

Dependencies:
  - Treasury contract (for fund disbursement on execution)
  - Soroban token contract (any SEP-41 token)
```

---

## Version History

| Version | Date | Contracts | Changes |
|---------|------|-----------|---------|
| `0.1.0` | TBD | All | Initial protocol release — core treasury, payroll streams, vesting, governance |

---

## Inter-Contract References

```
Governance ──references──> Treasury (for fund disbursement)
                            │
                            │ (token contract)
                            ▼
                    SEP-41 Token Contract
                            ▲
                            │ (token contract)
                            │
Treasury ──────────────────┤
Payroll Stream ────────────┤
Vesting ───────────────────┘
```

Governance stores the Treasury contract ID in its config (`set_treasury()`). All three fund-holding contracts (Treasury, Payroll Stream, Vesting) interact with the same or different SEP-41 token contracts for transfers.

---

## Verification Instructions

To verify a contract on the Stellar block explorer:

1. Navigate to `https://stellar.expert/explorer/[testnet|public]/contract/<CONTRACT_ID>`
2. Confirm the WASM hash matches the value in this registry
3. Confirm the deploy transaction hash matches
4. Confirm the contract was initialized with correct parameters

For reproducible builds:

```bash
# Build the contract
cd contracts
soroban contract build --out-dir build

# Compute hash
sha256sum build/treasure.wasm
# Compare with the WASM hash in this registry
```

---

## Deprecated Contracts

| Contract | Version | Contract ID | Network | Deprecated At | Successor ID | Reason |
|----------|---------|-------------|---------|---------------|-------------|--------|
| *(none yet)* | | | | | | |

When a contract is deprecated:
1. Mark the row with ~~strikethrough~~ in the active table
2. Move the record to this deprecated table
3. Note the successor contract ID
4. If funds remain in the deprecated contract, document the recovery procedure
