# `contracts/token-vault` — Architecture & Hardening Design (`Design.md`)

> **Status:** Draft / Pre-implementation  
> **Repository Note:** `contracts/token-vault` does not currently exist in `Perpetua-Contracts` (existing crates under `contracts/` are `archival-probe`, `factory`, `governance`, `stream`, and `stream-nft`). This document defines the design for `contracts/token-vault` and specifies the technical architecture addressing issues **#451**, **#447**, **#450**, and **#449**.

---

## 1. Overview & Crate Conventions

`contracts/token-vault` is a standalone Soroban smart contract crate following the repository's existing contract conventions (e.g., `contracts/stream/Cargo.toml`):

- **SDK & Toolchain:** `soroban-sdk = "27.0.6"` (Stellar Protocol 27), Rust `1.97.1`, `edition = "2021"`, target `wasm32v1-none`.
- **Testing:** `soroban-sdk` with `testutils` feature + `proptest = "1"` for property-based / fuzz testing.
- **Release Profile:** Size-optimized (`opt-level = "z"`, `lto = "fat"`, `codegen-units = 1`, `panic = "abort"`, `strip = "symbols"`, `debug = 0`).

### Responsibilities
The Token Vault manages custody, accounting, and fractional stream withdrawals for deposited Soroban SAC/SEP-41 tokens, enforcing strict solvency invariants, deterministic fixed-point arithmetic, minimal Soroban host CPU/memory footprint, and resilience under adversarial / partitioned execution.

---

## 2. Storage & Ledger Footprint Design

To minimize ledger entry reads/writes and avoid unbounded deserialization costs:

1. **Instance Storage (`DataKey::Config`, `DataKey::ReentrancyGuard`, `DataKey::TotalReserves(Address)`):**
   - Global vault configuration (admin, paused state, fee config) and compact re-entrancy lock flag.
   - Per-token aggregate accounting (`VaultTotals { total_deposited, total_reserved, total_withdrawn }`) stored in a single `#[contracttype]` struct so balance invariants are checked with one read/write.
2. **Persistent Storage (`DataKey::StreamAccount(u64)` / `DataKey::VaultPosition(Address, Address)`):**
   - Individual stream/vault positions keyed by ID or `(owner, token)` rather than stored inside a monolithic `Vec` or `Map` in a single ledger entry.
   - Avoids `O(N)` deserialization of unrelated positions during single-user operations.
3. **TTL Management:**
   - Explicit `extend_ttl` thresholds on persistent entries and instance storage to prevent archival desync.

---

## 3. Issue-Specific Designs

### 3.1 `#451` — Strict Invariant Assertions, Re-Entrancy Protection & State Desync Prevention

**Threat Model:**
- Cross-contract calls to external token contracts (`token::Client::transfer`) or hook receivers could attempt re-entrant entry into vault mutating methods (`deposit`, `withdraw`, `claim_stream`, `cancel_stream`).
- Partial state updates before external calls or unchecked balance deltas could cause `total_reserved` or `total_deposited` to diverge from actual vault token holdings.

**Design:**
1. **Checks-Effects-Interactions (CEI) Pattern:**
   - All authorization (`require_auth()`), input validation, and state mutations (updating position `withdrawn` amounts and `VaultTotals`) MUST complete **before** any external `token::Client` transfer.
2. **Explicit Re-Entrancy Guard:**
   - A scoped RAII-style or explicit storage lock (`DataKey::Locked(bool)`) checked at the entry of every state-mutating entrypoint.
   - If locked, immediately return `Err(VaultError::ReentrancyDetected)` (avoiding raw `panic!` where structured `#[contracterror]` is expected, while still aborting the transaction atomically).
3. **Strict Solvency & Conservation Invariants (`assert_vault_invariants`):**
   - After state updates (and optionally verified against `token_client.balance(&env.current_contract_address())` in debug/invariant-checked paths):
     - `total_deposited >= total_withdrawn`
     - `total_deposited - total_withdrawn >= total_reserved`
     - `position.withdrawn <= position.allocated`
     - `actual_vault_balance >= total_deposited - total_withdrawn` (prevents state desync from unexpected balance reductions; handles fee-on-transfer or direct-transfer edge cases explicitly).
   - All arithmetic uses `checked_add`, `checked_sub`, `checked_mul`, `checked_div` returning `VaultError::ArithmeticOverflow` / `VaultError::InsufficientLiquidity`.

---

### 3.2 `#447` — Resolving Precision Loss in Fractional Stream Withdrawal Calculations

**Problem:**
- Naive integer division `(elapsed * rate)` or `(total * elapsed) / duration` in incremental withdrawals truncates fractional remainders on every claim. Over many small withdrawals, repeated truncation either permanently locks dust in the vault or allows cumulative rounding drift.

**Design:**
1. **Cumulative-First Accounting (No Incremental Rounding Drift):**
   - Never compute `delta = ((now - last_claim) * total) / duration`.
   - Instead, always compute **cumulative vested amount** from `start_time`:
     $$\text{vested}(t) = \left\lfloor \frac{\text{total\_amount} \times (t - \text{start\_time})}{\text{duration}} \right\rfloor$$
   - Withdrawable amount is:
     $$\text{withdrawable}(t) = \text{vested}(t) - \text{already\_withdrawn}$$
   - At `t >= end_time`, `vested(t)` MUST equal `total_amount` exactly, guaranteeing zero dust remains locked when the stream completes.
2. **Overflow-Safe Wide Arithmetic (`u128` / `I256`):**
   - Cast `i128` token amounts and `u64` timestamps to `i256` / `u128` (or use `soroban_sdk::I256` / checked `u128` mul-div) before multiplying `total_amount * elapsed`, then divide by `duration`, and safely convert back to `i128`.
   - Explicitly handle edge cases:
     - `t <= cliff_time` → `0`
     - `duration == 0` → `VaultError::InvalidDuration` (rejected at creation)
     - `t >= end_time` → `total_amount`
     - `total_amount == 0` → `0`
     - `vested < already_withdrawn` → `VaultError::InvariantViolation`

---

### 3.3 `#449` — Minimizing Soroban Host CPU Instructions via Loop Deserialization Optimization

**Problem:**
- Iterating over `soroban_sdk::Vec<T>` or `Map<K, V>` stored as a single `Val` triggers repeated host-to-guest XDR/Val deserialization and host function calls inside loops, spiking Soroban CPU instruction budgets.

**Design:**
1. **Eliminate Unbounded Loops Over Storage Collections:**
   - Replace monolithic `Vec<StreamPosition>` storage with per-ID persistent ledger entries (`DataKey::Position(u64)`), making single-stream operations `O(1)`.
2. **Batch Processing Optimizations (Where Loops Are Required, e.g., `batch_withdraw`):**
   - **Hoist Invariants Out of Loops:** Read `env.ledger().timestamp()`, `VaultConfig`, and `VaultTotals` **once** before the loop; accumulate `total_payout` in a local `i128` variable and write `VaultTotals` **once** after the loop.
   - **Pre-indexed / Compact Keys:** Pass only `Vec<u64>` (stream IDs) as calldata rather than full structs, minimizing argument deserialization.
   - **Single Token Transfer Aggregation:** When batch-claiming multiple streams of the same token for the same recipient, sum the net payouts in memory and execute a single `token_client.transfer(...)` call.
   - **Budget Measurement:** Add `env.cost_estimate().budget()` assertions in unit tests to lock in CPU instruction and memory byte ceilings.

---

### 3.4 `#450` — Byzantine Fault Tolerance (BFT) & Network Partition Test Scenarios

**Scope:**
- Simulate adversarial, out-of-order, stale-ledger, and partitioned execution conditions in the Soroban test environment (`soroban_sdk::testutils`).

**Test Scenarios:**
1. **Stale / Out-of-Order Timestamp & Ledger Sequence Advancement:**
   - Simulate delayed transactions arriving after cliff/end boundaries, concurrent claims across ledger jumps, and verify cumulative-first math remains monotonic and deterministic.
2. **Conflicting / Byzantine Actor Sequences:**
   - Duplicate/replay withdrawal attempts within the same ledger and across ledgers.
   - Malicious token contract mock attempting re-entrancy into `withdraw` / `cancel` during `transfer`.
   - Unauthorized caller / spoofed admin or recipient attempting state mutation under `mock_auths` / explicit auth failure (`#[should_panic]` / `try_*` error assertions).
3. **Partial State / Partition Recovery:**
   - Simulate TTL expiration / archival boundary conditions and verify recovery or safe failure without corrupting `VaultTotals`.
   - Simulate `proptest` randomized operation sequences (`deposit`, `partial_withdraw`, `full_withdraw`, `cancel`, `reentrant_call`) interleaved with arbitrary time jumps, asserting that `VaultTotals` conservation invariants hold after every step.

---

## 4. Proposed Module Layout (`contracts/token-vault/`)

```text
contracts/token-vault/
├── Cargo.toml
└── src/
    ├── lib.rs          # Contract entrypoint, public methods, re-entrancy guard
    ├── storage.rs      # DataKey, VaultConfig, VaultTotals, Position, TTL helpers
    ├── math.rs         # Cumulative-first fixed-point stream math (overflow-safe)
    ├── errors.rs       # #[contracterror] VaultError enum
    ├── invariants.rs   # Solvency & conservation invariant assertions
    └── test.rs         # Unit, BFT/partition, budget-regression, and proptest fuzz suites
```

---

## 5. Verification Plan

Once `contracts/token-vault` is scaffolded and implemented:
1. **Unit & Invariant Tests:** `cargo test` inside `contracts/token-vault` covering all `VaultError` paths, CEI/re-entrancy mocks, and boundary timestamps (`0`, `cliff - 1`, `cliff`, `end - 1`, `end`, `end + N`).
2. **Property / Fuzz Tests (`proptest`):** Verify zero dust lockup on stream completion and strict conservation (`sum(withdrawn) + remaining == total_deposited`) across randomized withdrawal schedules.
3. **CPU / Memory Budget Tests:** Assert `env.cost_estimate().budget().cpu_instruction_cost()` stays within target thresholds for batch operations.
4. **WASM Build:** `cargo build --target wasm32v1-none --release` to verify Soroban compatibility and binary size.
