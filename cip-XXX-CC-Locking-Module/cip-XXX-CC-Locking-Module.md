<pre>
Number: CIP-XXX
Layer: Daml
Title: CC Locking Module — Phase 2 Primitive
Author(s): Luke Farrell (Cashen), Ian Hensel (Avro) w/ Community Working Group
Type: Standards Track
Status: Draft
Created: 2026-05-22
Approved:
License: CC0-1.0
Requires: CIP-0105, CIP-0116
</pre>

## 1. Abstract

This CIP specifies the on-chain locking primitive that the Global Synchronizer must provide in order to satisfy the Phase 2 enforcement requirements of [CIP-0105](../cip-0105/cip-0105.md) (Super Validator Locking and Long-Term Commitment Framework) and [CIP-0116](../cip-0116/cip-0116.md) (Featured App Locking), together with the Lock Substitution provision defined in the Super Validator Locking Operational Guidelines.

The primitive defines the network's responsibilities to **compute, enforce, and expose** the state required for continuous, on-chain evaluation of Super Validator (SV) and Featured App (FA) locking obligations. It covers the two-state token model (Liquid / Locked), the lifecycle of `ActiveLockContext` and `VestingLockContext`, beneficiary attribution and aggregation, authorization sets for unlock / withdraw / substitution, tier-evaluation inputs, atomic Lock Substitution, and the event surface required for indexers, dashboards, and the underlock / restoration enforcement flow.

This CIP defines **what** the network must do. It does not mandate **how** Daml-level templates must be structured; two illustrative implementation paths from the working-group requirements document — Path A (Amulet root state) and Path D (sibling template) — are summarized in §6 as examples of designs that satisfy the requirements. The final implementation will be selected by Splice contributor engineers and may differ from these examples; further engineering input is ongoing.

## 2. Motivation

[CIP-0105 §8](../cip-0105/cip-0105.md) specifies a two-phase rollout:

- **Phase 1 — Transitional Enforcement.** SVs disclose segregated PartyIds to the Foundation. Locking is evaluated weekly via off-chain dashboards. Lock Substitution is handled by a 24-hour add-then-remove window. This is the mechanism in use today.
- **Phase 2 — On-Chain Enforcement.** "Activates upon deployment of CC locking contracts to MainNet. Only on-chain locked CC counts. Vesting-based unlocks apply automatically. SV Weight updates continuously."

Phase 1 is operationally adequate for bootstrapping but has structural limits: (i) weekly cadence cannot enforce continuous tier evaluation; (ii) dashboard reconciliation depends on off-chain processes and does not produce cryptographic evidence of compliance; (iii) the 24-hour Lock Substitution window relies on a threshold-bridging mechanism that cannot be expressed atomically off-chain; (iv) [CIP-0116 §3](../cip-0116/cip-0116.md) requires immediate revocation of Featured App status on under-lock, which is not achievable at Phase 1 cadence.

Phase 2 requires a network primitive that:

- Represents lock state as on-chain, machine-verifiable facts rather than reported balances.
- Enforces the vesting-unlock model directly, with no path to bypass vesting other than atomic Lock Substitution.
- Supports per-round (continuous) tier evaluation against arbitrary thresholds set by policy.
- Emits the events required for indexers to compute aggregate locked balances per beneficiary, fire underlock and restoration events, and reconstruct historical state for at least the 35-day enforcement window in the SV Locking Operational Guidelines.

The Tokenomics-level commitments in CIP-0105 and CIP-0116 already exist; this CIP defines the Standards Track primitive they assume.

## 3. Scope

### In scope

The requirements in §4 must be sufficient to support, on-chain and continuously:

- CIP-0105 Phase 2: SV locking, vesting unlock, beneficiary aggregation, tier-input computation, and underlock enforcement.
- CIP-0116: per-PartyId Featured App locking with the minimums and 60-day linear vest period defined in CIP-0116.
- The Lock Substitution mechanism defined in the SV Locking Operational Guidelines, in its Phase 2 atomic form.

### Out of scope

- CIP-0105 Phase 1 transitional enforcement (off-chain dashboards).
- CIP-0116 transitional dashboards.
- Specific tier-threshold values, reward formulas, and penalty timelines — these are policy parameters set externally and consumed by the primitive (see §4.5).
- Application-layer workflows (commercial deal terms, marketplace mechanics, off-chain coordination), except insofar as they constrain primitive behavior (see Appendix A).
- Off-platform implementation details (custody integrations, dashboard providers, wallet UX).

See §11 for source documents and references.

## 4. Specification

### 4.1 Terminology

| Term | Definition |
| --- | --- |
| **Lock state** | The state of a CC token: `Liquid` or `Locked`. |
| **LockContext** | A contract that defines the semantics of a particular lock. Of type `ActiveLockContext` or `VestingLockContext`. Referenced by Locked tokens via `lockContextId`. |
| **ActiveLockContext** | LockContext representing a position locked toward a beneficiary; contributes to beneficiary aggregation. |
| **VestingLockContext** | LockContext representing a position in vesting after unlock initiation; does **not** contribute to beneficiary aggregation. |
| **Beneficiary** | The entity (SV operator or Featured App) to which an Active lock is attributed for compliance calculation. |
| **Aggregate locked balance** | Sum of `amount` across all `ActiveLockContext`s attributed to a given beneficiary, across all PartyIds. |
| **Lock Substitution** | An atomic operation: archive of an existing LockContext and its associated Locked token, paired with creation of a new LockContext for the inheriting party and its associated Locked token. On Vesting substitution, currently-available balance is atomically withdrawn to the outgoing holder. |
| **Tier evaluation** | Computation comparing a beneficiary's aggregate locked balance to a threshold. Thresholds themselves are policy parameters set by CIP-0105 / CIP-0116 and are not encoded in the primitive. |
| **Underlock event** | Transition of a beneficiary's aggregate locked balance from at-or-above threshold to below threshold. |
| **Restoration event** | The reverse transition. |

Each requirement below is tagged by category:

- **Compute** — the network must derive a value.
- **Enforce** — the network must reject operations that violate the rule.
- **Expose** — the network must emit events or support queries.

### 4.2 State model

#### 4.2.1 Two lock states must exist *(Compute, Expose)*

CC tokens MUST be representable as being in exactly one of two mutually exclusive states: `Liquid` or `Locked`. A Locked token references a LockContext that defines the semantics of the lock; "Active" or "Vesting" are properties of the LockContext, not of the token itself.

*Source:* CIP-0105 §2 ("Only actively locked CC counts toward the SV weighting algorithm"); CIP-0116 §2.1 ("Only actively locked CC counts toward FA eligibility").

#### 4.2.2 State transitions must be restricted *(Enforce)*

Token state transitions MUST occur only as paired operations with LockContext lifecycle events:

- **Lock creation:** `Liquid → Locked`, paired with creation of an `ActiveLockContext`.
- **Unlock initiation:** `Locked → Locked` (the token's `lockContextId` reference changes), paired with archive of an `ActiveLockContext` and creation of a `VestingLockContext`.
- **Withdraw:** `Locked → Liquid` for the withdrawn amount, paired with archive of the parent `VestingLockContext` and creation of a successor `VestingLockContext` if locked balance remains.
- **Lock Substitution:** paired archive of the outgoing {LockContext + Locked token}, with creation of the incoming {LockContext + Locked token}. See §4.6.

Authorization required to trigger each operation is specified in §4.4.

*Source:* CIP-0105 §2 ("All locked CC, including any balance above the applicable tier threshold, must be withdrawn through the vesting process. Any withdrawal of locked CC that circumvents vesting constitutes a lock violation").

#### 4.2.3 Vesting progression must be derivable from VestingLockContext fields *(Compute)*

For any `VestingLockContext` at time `t`, the following quantities MUST be derivable deterministically from the contract's immutable fields and the current time. No mutation is required:

- *Cumulative vested:* `min(scheduleSize × (t − vestStartedAt) / (vestEndsAt − vestStartedAt), scheduleSize)`
- *Already withdrawn:* `scheduleSize − currentLockedAmount`
- *Currently available:* `cumulativeVested − alreadyWithdrawn`
- *Currently locked:* `scheduleSize − cumulativeVested`

*Source:* CIP-0105 §2 (SV linear vest rate); CIP-0116 §2.1 (60-day FA unlock period).

#### 4.2.4 Multiple vest schedules must be supported *(Compute, Enforce)*

The primitive MUST support at least two vest schedules, specified on the `ActiveLockContext` at lock creation:

- **365.25 days** for SV beneficiaries (per CIP-0105 §2).
- **60 days** for FA beneficiaries (per CIP-0116 §2.1).

The vest schedule is immutable on the `ActiveLockContext`. On unlock initiation, the `VestingLockContext`'s `vestEndsAt` MUST be set to current time plus the parent `ActiveLockContext`'s vest schedule.

Additional vest schedules MAY be added via governance choices on the network-level `LockingRules` singleton (see §4.7.2).

### 4.3 Beneficiary and attribution

#### 4.3.1 Beneficiary must be attributable to each Lock *(Compute, Expose)*

Each Locked position MUST be associated with a beneficiary identifier. The beneficiary identifier is the PartyId to which the lock is attributed for tier-calculation purposes.

*Source:* CIP-0105 §4; CIP-0116 §1.

#### 4.3.2 Beneficiary types must be distinguishable *(Compute, Expose)*

The primitive MUST distinguish between SV-operator beneficiaries and Featured-App beneficiaries, because their tier-evaluation rules differ. (CIP-0105 §4 vs CIP-0116 §1.)

Whether the type is stored explicitly on the `ActiveLockContext` or derived from external registries is left to the implementation.

#### 4.3.3 Only ActiveLockContext contributes to beneficiary aggregation *(Compute)*

Beneficiary aggregation MUST include only Locked tokens whose `lockContextId` references an `ActiveLockContext`. Locked tokens referencing a `VestingLockContext` MUST be excluded.

Unlock initiation (§4.2.2) immediately removes the unlocked amount from beneficiary aggregation by replacing the `ActiveLockContext` reference with a `VestingLockContext` reference.

*Source:* CIP-0105 §2 ("Only the fully locked (non-vesting) portion counts towards maintaining SV Weight"); CIP-0116 §2.1.

#### 4.3.4 Aggregate locked balance per beneficiary must be queryable *(Compute, Expose)*

The primitive MUST support queries of the form "what is the total CC currently attributed to beneficiary `X`?". The query MUST operate over Locked tokens whose `lockContextId` references an `ActiveLockContext` with `beneficiary = X`, summing the contract `amount` across all PartyIds.

*Source:* CIP-0105 §7 ("It must be possible to lock from multiple wallets, custodians, and PartyIds and meet the locking threshold requirements in aggregate"); CIP-0116 §1.

### 4.4 Holder and authorization

#### 4.4.1 Holder identity must be tracked per Lock *(Compute, Expose)*

Each `LockContext` MUST record the holder identity. The holder is the party that controls the locked CC (operationally, the wallet or custody account where the CC resides).

*Source:* CIP-0105 §7 (locking from self-custody wallets, institutional custodians, and qualified third-party custody providers); SV Locking Operational Guidelines ("PartyIDs must not commingle funds with other holders").

#### 4.4.2 Lock attributes must be inheritable across paired lock operations *(Compute, Enforce)*

A new `LockContext` created atomically with the archive of an existing `LockContext` MUST inherit the attributes required for continuity of the lock. Inheritance rules are specified per attribute in §4.6.2.

No CC movement between wallets is required; each party operates on their own CC throughout.

#### 4.4.3 Unlock authorization must be specifiable per Lock *(Compute, Enforce)*

The parties authorized to exercise `InitiateUnlock` on an `ActiveLockContext` MUST be specifiable at lock creation, as a list of one or more **alternative authorization sets**.

An `InitiateUnlock` attempt is authorized if any one alternative set is fully satisfied (every party in that set signs). Within an alternative set, all parties must sign; across alternative sets, any one suffices.

Alternative sets MAY include the holder, third parties (custodians, counterparties), and the DSO. Unlock authorization is stored on the `ActiveLockContext` only; `VestingLockContext` has its own withdraw authorization per §4.4.7.

#### 4.4.4 Unlock and Withdraw initiation must be enforceable to authorized parties only *(Enforce)*

`InitiateUnlock` on an `ActiveLockContext` is authorized only if one of the alternative sets in `unlockAuthorizationSets` is fully satisfied (per §4.4.3). `Withdraw` on a `VestingLockContext` is authorized only if one of the alternative sets in `withdrawAuthorizationSets` is fully satisfied (per §4.4.7). The network MUST reject operations that do not satisfy at least one alternative set.

#### 4.4.5 Lock creation is unilateral authorization by holder *(Enforce)*

Creation of an `ActiveLockContext` (and the paired `Liquid → Locked` token transition) MUST be authorized solely by the holder. No additional party authorization is required at the moment of lock creation. The network MUST reject creation attempts not authorized by the holder.

#### 4.4.6 Substitution authorization must be specifiable *(Enforce)*

The parties required to authorize a substitution MUST be recorded on each `LockContext` as `substitutionAuthorizationSets`, expressed as a list of one or more alternative authorization sets. A substitution attempt is authorized if any one alternative set is fully satisfied. Alternative sets MAY include the outgoing holder, the incoming holder, the beneficiary (for Active substitution), and DSO parties. The network MUST reject substitution attempts that do not satisfy at least one alternative set.

On the `Active → Vesting` transition (`InitiateUnlock`), `substitutionAuthorizationSets` is inherited from the parent `ActiveLockContext` to the new `VestingLockContext`. On substitution, `substitutionAuthorizationSets` is set **fresh** by the incoming holder on the new `LockContext`, not inherited from the outgoing context.

#### 4.4.7 Withdraw authorization must be specifiable *(Compute, Enforce)*

The parties required to authorize `Withdraw` on a `VestingLockContext` MUST be recorded as `withdrawAuthorizationSets`, expressed as a list of one or more alternative authorization sets. A `Withdraw` attempt is authorized if any one alternative set is fully satisfied. Alternative sets MAY include the holder, third parties (custodians, smart contracts acting as agents, platform parties), the beneficiary, and the DSO. The set need **not** include the holder.

`withdrawAuthorizationSets` is set fresh at `ActiveLockContext` creation by the holder, inherited on the `Active → Vesting` transition (`InitiateUnlock`) to the new `VestingLockContext`, and set fresh by the incoming holder on substitution. Withdraw chain continuation (`Withdraw` → successor `VestingLockContext`) preserves `withdrawAuthorizationSets` unchanged.

### 4.5 Tier evaluation

#### 4.5.1 Tier evaluation inputs must be queryable *(Compute, Expose)*

For any beneficiary at any moment, the following inputs MUST be derivable from network state:

- Aggregate locked balance — sum of `amount` across `ActiveLockContext`s attributed to this beneficiary (per §4.3.4).
- Beneficiary type (SV operator or Featured App; per §4.3.2).
- For SV beneficiaries: aggregate lifetime earned SV rewards (the denominator of the CIP-0105 percentage-based tier calculation).

The aggregate lifetime SV rewards value MUST reference a **single canonical metric formula** agreed by the network. The primitive does not define this formula — multiple dashboards already produce this calculation in Phase 1, and the network must converge on one shared definition for Phase 2 enforcement. The primitive consumes the resulting value as an input to tier evaluation.

*Source:* CIP-0105 §4.

#### 4.5.2 Tier evaluation must be performable on a per-round cadence *(Compute)*

Tier evaluation is performed continuously, per CIP-0105 Phase 2 ("SV Weight updates continuously"), on a per-minting-round basis. The primitive MUST support computation of tier inputs at any round boundary, without requiring batch updates or manual triggers.

#### 4.5.3 Underlock events must be observable *(Expose)*

When a beneficiary's aggregate locked balance drops below a threshold (defined externally), an **underlock** event MUST be emitted with the timestamp of the threshold crossing. Threshold values are not encoded in the primitive — the primitive supports queries against arbitrary thresholds determined by tier-evaluation logic.

*Source:* CIP-0105 §6 (penalty mechanics start at the underlock event); CIP-0116 §3 ("If locking falls below required thresholds, Featured App status is immediately removed").

#### 4.5.4 Restoration events must be observable *(Expose)*

When a beneficiary's aggregate locked balance returns from below threshold to at-or-above threshold, a **restoration** event MUST be emitted with timestamp.

*Source:* CIP-0105 §6 ("The SV has 30 days from the initial under-lock event to restore its lock and return to the higher tier"); SV Locking Operational Guidelines ("Restoring an SV Weight").

### 4.6 Lock Substitution

Lock Substitution is referenced in CIP-0105 §2 as "the sole exception" to the vesting unlock model. The Phase 1 implementation uses a 24-hour add-then-swap window; the Phase 2 primitive defined here replaces that with an atomic substitution operation.

Substitution is an atomic operation: the outgoing party's `LockContext` and Locked token are archived, and a new `LockContext` and Locked token are created for the incoming party with inherited attributes per §4.6.2. Substitution applies to both `ActiveLockContext` (Active substitution) and `VestingLockContext` (Vesting substitution). Each party operates on their own CC; no CC moves between wallets.

#### 4.6.1 Substitution must be supported as a single primitive operation *(Compute, Enforce)*

Substitution atomically commits the following:

- Archive of the outgoing party's `LockContext` and Locked token.
- Creation of the incoming party's `LockContext` and Locked token, with attributes inherited per §4.6.2.

For Vesting substitution, currently-available balance on the outgoing `VestingLockContext` MUST also be atomically withdrawn to the outgoing holder (see §4.6.4).

#### 4.6.2 Inherited attributes *(Enforce)*

Attributes inherited on substitution depend on the LockContext type.

**ActiveLockContext substitution** — the new `ActiveLockContext` inherits:

- `beneficiary`
- `vestSchedule`
- `originatedAt`

`unlockAuthorizationSets`, `substitutionAuthorizationSets`, and `withdrawAuthorizationSets` are specified **fresh** on the new `ActiveLockContext` at creation per §4.4.3, not inherited.

**VestingLockContext substitution** — the new `VestingLockContext` inherits:

- `vestEndsAt` (preserves the original maturity date)

The following are set fresh on the new contract:

- `scheduleSize` = the amount being substituted to the incoming party.
- `currentLockedAmount` = `scheduleSize`.
- `vestStartedAt` = current time (transaction commit time).
- `holder` = incoming party.
- `substitutionAuthorizationSets` and `withdrawAuthorizationSets` — set fresh by the incoming holder.

*Rationale:* Vested progress is not reset; preservation of `vestEndsAt` is what makes substitution "the sole exception" to the vesting requirement in CIP-0105 §2.

#### 4.6.3 Substitution must support partial substitution *(Compute, Enforce)*

Both Active and Vesting substitution MUST support partial substitution. Given an outgoing `LockContext` with total amount `N` and a partial substitution of amount `M` where `M < N`:

- *Active:* the outgoing party's successor `ActiveLockContext` receives `amount = N − M`; the incoming party's new `ActiveLockContext` receives `amount = M`. Both inherit attributes per §4.6.2.
- *Vesting:* after the atomic withdraw step (§4.6.4) drains available balance, the remaining locked amount `L` is split. The outgoing party's successor receives `scheduleSize = L − M` and a fresh `vestStartedAt`; the incoming party's new `VestingLockContext` receives `scheduleSize = M` and a fresh `vestStartedAt`. Both inherit `vestEndsAt` per §4.6.2. The aggregate CC that vests per day and the final maturity date are preserved (split across two holders).

All resulting `LockContext`s and their paired Locked tokens MUST be created atomically with the archive of the outgoing `LockContext` and Locked token.

#### 4.6.4 Vesting substitution must atomically withdraw available balance *(Compute, Enforce)*

When substituting a `VestingLockContext`, currently-available balance (per §4.2.3) MUST be withdrawn to the outgoing holder as part of the same atomic operation. The substitution proceeds on the remaining locked balance after withdrawal. This eliminates any "available but unwithdrawn" balance from being inherited by the incoming holder.

#### 4.6.5 Substitution must be atomic *(Enforce)*

All archive and create operations across the token and `LockContext` layers MUST commit in a single transaction. No interim state is observable.

- For *Active* substitution, the beneficiary aggregate is invariant at the commit boundary.
- For *Vesting* substitution, the beneficiary aggregate is already zero, and the atomic withdraw produces a Liquid token of `currentlyAvailable` amount addressed to the outgoing holder.

This replaces the Phase 1 transitional 24-hour add-then-swap mechanism.

#### 4.6.6 Substitution must not generate underlock or restoration events *(Enforce, Expose)*

Active substitution preserves beneficiary aggregate exactly; Vesting substitution operates on positions that do not contribute to beneficiary aggregate. In neither case does substitution cross a threshold. No underlock or restoration event SHALL be emitted by the substitution itself. Any such event observable around the time of a substitution is attributable to a separate cause.

### 4.7 Observability and module shape

#### 4.7.1 State-transition events must be observable *(Expose)*

Every `Liquid ↔ Locked` token transition MUST emit a token event including: token identifier, previous state, new state, timestamp, holder, and `lockContextId` (when transitioning to or from `Locked`).

Every `LockContext` creation and archive MUST emit a corresponding lifecycle event including: context identifier, context type (`ActiveLockContext` or `VestingLockContext`), and relevant fields per §4.7.2.

#### 4.7.2 LockContext lifecycle and substitution events must be observable *(Expose)*

- `ActiveLockContext` create events expose: context identifier, holder, beneficiary, amount, vest schedule, `originatedAt`, `unlockAuthorizationSets`, `substitutionAuthorizationSets`, `withdrawAuthorizationSets`, timestamp.
- `ActiveLockContext` archive events expose: context identifier, timestamp.
- `VestingLockContext` create events expose: context identifier, holder, `scheduleSize`, `currentLockedAmount`, `vestStartedAt`, `vestEndsAt`, `substitutionAuthorizationSets`, `withdrawAuthorizationSets`, timestamp.
- `VestingLockContext` archive events expose: context identifier, timestamp.

Substitution MUST be observable as a single atomic transaction containing paired archive and create events. Indexers correlate the outgoing and incoming contexts by transaction identifier.

#### 4.7.3 Aggregate balance changes must be queryable historically *(Expose)*

For audit and dashboard purposes, the network MUST support queries of aggregate locked balance per beneficiary at historical timestamps, not just current state. Historical data MUST be queryable to cover at least the enforcement window of **35 days** specified in the SV Locking Operational Guidelines ("Number of rounds within the past 35 days when the locked amount dropped below the defined thresholds"). Beyond the enforcement window, public explorers MAY provide a longer history.

#### 4.7.4 Module shape

Section §6 below describes two illustrative implementation paths. In both, the primitive is realized as:

- A singleton **`LockingRules`** template holding network-level parameters (supported vest schedules, DSO authorization hooks, governance choices) — held by DSO parties, following the existing `AmuletRules` pattern.
- Per-instance **`ActiveLockContext`** and **`VestingLockContext`** templates holding the parameters of an individual lock or vest schedule.
- Token-layer state that records lock status and references the active `LockContext` via `lockContextId`.

The shape of the token-layer state is the point of divergence between the candidate paths (see §6).

### 4.8 Reward attribution (work in progress)

The SV or FA beneficiary of a lock currently accrues all coupon rewards generated by that lock. The working group has flagged the following questions for further specification, which are **not** resolved by this CIP:

1. **Coupon attribution and distribution** — whether coupons are split automatically into locked vs liquid, the rule that drives the split, and whether the action is automatic or post-delivery.
2. **Milestone (escrowed) rewards** — when these are counted in the lifetime-rewards denominator, and how the locked / liquid split is decided.
3. **Auto-routing for third-party delegations** — whether rewards auto-split for third-party holders is part of the primitive or settled at the application layer.

These items are expected to be addressed in a follow-up CIP and are listed here for traceability.

## 5. Application workflows (informative)

The following representative workflows are supported by the primitive specified in §4. They are informative; the primitive's behavior is defined by §4.

1. **SV lock creation from a custodian.** Holder (the custodian's wallet) exercises `CreateActiveLockContext` (§4.4.5) with the SV operator as beneficiary, vest schedule 365.25 days, and configured authorization sets. The `Liquid → Locked` token transition occurs atomically. The locked amount contributes to the SV beneficiary's aggregate per §4.3.4.
2. **Partial unlock on an SV lock.** A party satisfying one of the `unlockAuthorizationSets` exercises `InitiateUnlock` for a specified amount. The parent `ActiveLockContext` is archived; a successor `ActiveLockContext` is created for the remaining amount; a new `VestingLockContext` is created for the unlocking amount, with `vestStartedAt = now`, `vestEndsAt = now + 365.25 days`, and inherited authorization sets. Tier evaluation may transition the SV to a lower tier.
3. **Substitution of an Active position.** Outgoing party's `ActiveLockContext` and Locked token are atomically archived; the incoming party's previously-Liquid CC is locked atomically as a new `ActiveLockContext`, inheriting beneficiary, vest schedule, and `originatedAt`. Authorization sets are fresh.
4. **FA lock creation.** Holder exercises `CreateActiveLockContext` with the FA PartyId as beneficiary and vest schedule 60 days.
5. **Substitution of a Vesting position.** Outgoing party's `VestingLockContext` and Locked token are atomically archived. Currently-available balance is withdrawn to the outgoing holder. Incoming party's previously-Liquid CC is locked atomically with `vestEndsAt` inherited from the outgoing context.
6. **Withdraw of vested balance.** A party satisfying one of the `withdrawAuthorizationSets` exercises `Withdraw` for an amount no greater than currently-available. The parent `VestingLockContext` is archived; a Liquid token is created in the recipient wallet; if locked balance remains, a successor `VestingLockContext` is created with `currentLockedAmount` decremented and authorization sets inherited.
7. **Beneficiary aggregate change on unlock initiation.** When unlock is initiated, the unlocking amount immediately leaves the beneficiary's aggregate per §4.3.3. If the resulting aggregate crosses a threshold, an underlock event is emitted per §4.5.3.

## 6. Rationale

### 6.1 Why a primitive, and not policy

This CIP intentionally specifies what the network must compute, enforce, and expose — not the precise policy thresholds, reward schedules, or restoration timelines. Those values live in CIP-0105 and CIP-0116 (and in the SV Locking Operational Guidelines) and are subject to amendment under the existing governance process. Locating policy outside the primitive lets governance adjust thresholds without protocol changes, while still relying on the same network-level guarantees of correctness and atomicity.

### 6.2 Why the two-layer (token + LockContext) model

Beneficiary, authorization sets, vest schedule, and originated-at are properties of a lock, not of an individual token. Carrying them in a separate `LockContext` keeps token-layer changes minimal, allows multiple Locked tokens to share semantics, and gives indexers a single contract to subscribe to for lifecycle changes. The token layer carries the lock state and `lockContextId` reference, which is sufficient for transfer enforcement and for joining at query time.

### 6.3 Why atomic substitution

Phase 1 substitution uses a 24-hour window during which the SV's balance is observed above threshold by adding the incoming loan provider's contribution and then removing the outgoing provider's. This relies on two off-chain operations and a tolerance window in the dashboard. Phase 2 requires continuous evaluation, which the bridging mechanism cannot satisfy without permitting an observable temporary imbalance. Defining substitution as a single atomic transaction at the protocol level eliminates the bridging window and removes the need for a tolerance band in the indexer.

### 6.4 Why alternative authorization sets

Custodial and counterparty arrangements vary widely (self-custody, single-custodian, multi-custodian, qualified third-party). A single fixed set of signatories cannot express all of them. Lists of alternative sets generalize this: any one alternative set may authorize the operation, and each set requires all of its members to sign. This makes it possible to encode arrangements such as "holder alone may withdraw, or holder + custodian, or DSO alone in emergency" without protocol changes.

### 6.5 Candidate implementation paths (illustrative)

The working-group requirements document (linked in §11) evaluates two implementation paths that the working group considers functionally equivalent at the requirements level. **Both are listed here as examples of how the requirements in §4 can be satisfied. The choice of path is not made by this CIP and remains open to further engineering input from Splice contributors.** A future revision of this CIP — or a subsequent Standards Track CIP — will record the selected approach.

#### Path A — Amulet root state

The `Amulet` contract is extended with typed fields encoding lock state directly on the token. The transfer function becomes state-aware natively.

- Token-level fields added to `Amulet`:
  - `lockState : enum { Liquid, Locked }` (default `Liquid`).
  - `lockContextId : LockId?` (non-null when `lockState == Locked`, null otherwise).
- Transfer enforcement: the existing `Amulet` transfer choice checks `lockState`. Transfer is permitted only when `lockState == Liquid`. Lock state changes are privileged operations invoked by the Locking Module.
- `LockContext` (beneficiary, vest schedule, holder, authorization sets, vest progression fields) lives in the Locking Module. The token references the active context via `lockContextId`.

**Strengths:** type-safe; transfer enforcement is native; no external query at transfer time.
**Trade-offs:** requires changes to the `Amulet` contract; existing consumers must become aware of the new state field; schema migration is required if tokens exist at deployment.

#### Path D — Sibling template (no Amulet change)

A new template parallel to the existing `LockedAmulet` represents the locked position. The `Amulet` contract is unchanged. Locking moves CC from an `Amulet` to the new template (archive `Amulet` + create new template); unlocking reverses the move.

- Token-level changes to `Amulet`: **none.**
- New template (suggested name: `GovernanceLockedAmulet`):
  - `amulet : Amulet` — the embedded token data (matching the existing `LockedAmulet` pattern).
  - `lockContextId : ContractId LockContext` — reference to the active `LockContext`.
- Transfer enforcement: `GovernanceLockedAmulet` has no transfer choice. The CC is non-transferable while held in this template by construction.

**Strengths:** no `Amulet` changes; preserves existing template invariants; mirrors Splice's existing `LockedAmulet` architectural pattern; type-safe.
**Trade-offs:** two distinct templates for "this CC is locked" (existing `LockedAmulet` for TimeLocks + new `GovernanceLockedAmulet` for governance locks) — boundary must be clearly communicated to consumers; lock state is implied by which template the CC is in, not by a queryable field on a single contract.

The requirements document also considers, and rules out, two further paths:

- *Path B (state registry / governor module)* — removed from consideration on query-workload grounds; the mirror of token holder identity introduces sync risk and per-transfer hook latency.
- *Path C (vault / wrapper)* — removed because the wrapper holds CC in custody, which changes the trust and tax profile for institutional holders and conflicts with the design preference that CC remain in user wallets.

Path A is the working group's currently preferred path; Path D is the preferred alternative if modifying `Amulet` proves infeasible. Final selection is a decision for Splice contributor engineers.

## 7. Backwards compatibility

This CIP introduces a Standards Track change to the on-chain locking primitive. Backwards-compatibility considerations vary by implementation path (§6.5) and are summarized here.

### 7.1 Phase 1 to Phase 2 cutover

Phase 1 (off-chain dashboard reporting per CIP-0105 §8) coexists with the Phase 2 primitive only during the deployment window. Once the Phase 2 contracts are live on MainNet, CIP-0105 §8 specifies that "only on-chain locked CC counts" — Phase 1 dashboards become observability tools, not the source of truth. Foundation tooling and SV/FA reporting infrastructure that today consumes weekly dashboard data will need to consume the continuous event stream defined in §4.7. The operational impact is flagged in §10.

### 7.2 Existing Amulet and LockedAmulet contracts

- *Path A* requires a schema migration for any `Amulet` tokens existing at deployment time to populate the new `lockState` and `lockContextId` fields. Consumers of `Amulet` (wallets, scan APIs, third-party indexers) must be updated to read the new fields.
- *Path D* leaves the `Amulet` contract untouched. The existing `LockedAmulet` template (used for TimeLock-based escrow) continues to operate. A new `GovernanceLockedAmulet` template is added; consumers that need to identify locked CC must query both templates and recognize the distinct semantics.

In either path, the existing `LockedAmulet` template's TimeLock semantics are preserved unchanged — this CIP does not subsume the short-term escrow use cases that `LockedAmulet` serves today.

### 7.3 Substitution mechanism

The Phase 1 24-hour add-then-swap window for Lock Substitution (per the SV Locking Operational Guidelines) is replaced by the atomic substitution operation defined in §4.6. SVs that have substitution arrangements with loan providers will need to update operational runbooks accordingly. No data migration is required.

### 7.4 Token Standard compatibility

This CIP defines locking governance state that lives alongside, but is distinct from, the [CIP-0056](../cip-0056/cip-0056.md) Canton Network Token Standard. Token-standard operations on `Amulet` are expected to continue to work for unlocked balances. Behavior of token-standard operations attempted against Locked CC, and any interaction with [CIP-0112](../cip-0112/cip-0112.md) (Token Standard V2), is implementation-defined and should be specified by the selected reference implementation.

## 8. Reference implementation

A reference implementation is **not yet available**. As noted in §6.5, two candidate paths (A and D) have been evaluated by the working group at the requirements level and are considered functionally equivalent for the purposes of §4. Selection of the implementation path is in the hands of Splice contributor engineers and is the subject of ongoing engineering input.

When a reference implementation is available, it MUST include:

- Daml templates for `LockingRules`, `ActiveLockContext`, and `VestingLockContext` (or the equivalents in the selected path).
- Token-layer changes necessary to satisfy §4.2 and §4.3.3.
- Tests covering each requirement in §4, including atomic-substitution edge cases (§4.6) and authorization-set evaluation (§4.4).
- Indexer guidance for satisfying §4.7.3 (historical reconstruction for the 35-day enforcement window).
- Migration tooling for any state required by the selected path (e.g., Path A's `Amulet` field additions).

This section will be updated when an implementation candidate is proposed.

## 9. Security considerations

- **Atomicity of substitution (§4.6.5).** Any non-atomic implementation re-opens the threshold-bridging window that Phase 2 is intended to eliminate. Implementations MUST commit all four (or more) archive/create events in a single ledger transaction.
- **Vesting cannot be reset by substitution (§4.6.2).** A `VestingLockContext` substitution that reset `vestEndsAt` to a later time would constitute a lock violation in the sense of CIP-0105 §2. The requirement that `vestEndsAt` be inherited unchanged is the protocol guarantee against this.
- **Withdraw authorization need not include the holder (§4.4.7).** This permits arrangements (e.g., custodian-only or smart-contract-agent) that intentionally exclude the holder. Implementations and indexers SHOULD make this explicit in event metadata so that holders are aware of who can withdraw their vested balance.
- **Lock creation is unilateral (§4.4.5).** A holder can create an `ActiveLockContext` against any beneficiary they choose, contributing to that beneficiary's aggregate. Beneficiaries that do not wish to receive locks from arbitrary holders should consider this when defining substitution and withdraw authorization arrangements with counterparties (the operational, not protocol, layer).
- **Underlock event timing (§4.5.3).** The event fires at the round in which the threshold is crossed. Foundation tooling that triggers penalties at the round level needs to accept this cadence; tooling that batches events on a slower cadence may delay enforcement actions defined in CIP-0105 §6.

## 10. Open discussion points

The following items are flagged for working-group discussion and are not resolved by this CIP. Each represents a design decision or operational concern that benefits from broader ecosystem input.

1. **Atomic withdraw inside `Substitute` on `VestingLockContext`** (§4.6.4). The alternative is to require the holder to call `Withdraw` before `Substitute` as a two-step process and reject `Substitute` if any available balance remains. Atomic withdraw is operationally simpler; two-step is more conservative. The choice affects authorization semantics and the indexer event surface.
2. **Per-round indexing performance.** Continuous evaluation cadence (§4.5.2) combined with high-frequency state changes (substitutions, partial unlocks) may produce significant indexer load. Performance implications and any required indexer infrastructure standards should be discussed.
3. **Foundation operational tooling at Phase 2 cadence.** Underlock and restoration events fire at the round in which the threshold is crossed (§4.5.3 / §4.5.4). Whether existing Foundation tooling and SV/FA reporting infrastructure can consume these events at continuous cadence — and what changes to current Phase 1 dashboards and notification systems may be required — is a separate operational question.
4. **Beneficiary type on `ActiveLockContext` vs derived from external registries** (§4.3.2). Storage on the contract is the default in this draft; an alternative is to derive the type at query time from an external registry of SVs and FAs. The choice affects the contract schema and the query surface.
5. **Rewards split mechanics and rules.** For active-round rewards and escrowed milestone rewards, the working group has flagged the need to specify in what state rewards are delivered to beneficiaries, and how disputes about escrow timelines are handled. See §4.8.
6. **Metadata and traffic.** Onchain metadata and traffic utilization implications of the candidate paths, particularly under continuous evaluation, should be assessed for impact on network performance.

## 11. References

- [CIP-0105: Super Validator (SV) Locking & Long-Term Commitment Framework](../cip-0105/cip-0105.md)
- [CIP-0116: Featured App Locking](../cip-0116/cip-0116.md)
- [CIP-0056: Canton Network Token Standard](../cip-0056/cip-0056.md)
- [CIP-0112: Canton Network Token Standard V2](../cip-0112/cip-0112.md)
- [SV Locking Operational Guidelines (Canton Foundation configs)](https://github.com/canton-foundation/configs/blob/main/Super%20Validator%20Operational%20Processes/SV-Locking-Process.md)
- [CC Locking Module — Requirements and Implementation Approaches (working group document)](https://docs.google.com/document/d/13zl8ILEWq6CvSALk2LA79I2rE1PHfni1EFf5ywr-Aik/edit?tab=t.0#heading=h.dtx31zjb1okc) — the requirements in §4 are drawn from Section A of this document; the candidate paths summarized in §6.5 are drawn from Section B.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

## Changelog

- 2026-05-22: Initial draft published. Section A requirements specification with Path A and Path D summarized as illustrative implementations pending further engineering input.
- 2026-05-26: Review feedback. Moved Source documents subsection out of §3 (consolidated under §11 References). Reworded §4.3.2 to frame the beneficiary-type storage question as implementation discretion rather than an open working-group item. Updated cross-reference in §6.5 from §3 to §11.
