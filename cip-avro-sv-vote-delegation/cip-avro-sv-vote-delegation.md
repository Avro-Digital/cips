## SV Governance Vote Delegation

<pre>
  CIP:  ?
  Layer: Daml
  Title: SV Governance Vote Delegation
  Author: Avro Digital
  License: CC0-1.0
  Status: Draft
  Type: Standards Track
  Created: 2026-08-05
  Post-History:
</pre>

## Abstract

The current Splice SV governance flow uses the SV operator party as both the
node-automation identity and the governance-voting identity. An SV organization
that wants a policy owner, committee representative, or executive to vote on its
behalf must therefore give that person the SV's own credentials, which carry
full authority over the node.

This CIP lets an SV delegate its governance vote to a different party, so that
party can review proposals, cast votes, and participate in governance without
holding SV credentials. An SV signs a `VoteDelegation` contract that names one
`voterParty`. That party exercises two relay choices on the delegation, which in
turn exercise the existing `DsoRules_RequestVote` and `DsoRules_CastVote` on the
SV's behalf. Both choices take one optional `voterParty` argument and add the
delegated party as a controller, so a delegated exercise is co-authorized and
recorded as such. The delegated party's authority extends to those two choices.

The vote remains the SV's own vote. It occupies the SV's existing slot in
`VoteRequest.votes`, carries no additional weight, and creates no second voting
unit. An SV revokes a delegation by archiving the contract.

The surface is compatible with the external-party submission flow of
[CIP-0103][cip-0103]: the delegated party can hold its key in any conforming
wallet and submit from a participant that does not host DSO contracts, using
disclosed contracts.

[cip-0103]: https://github.com/canton-foundation/cips/blob/main/cip-0103/cip-0103.md
[cip-0105]: https://github.com/canton-foundation/cips/blob/main/cip-0105/cip-0105.md
[scu]: https://docs.daml.com/upgrade/smart-contract-upgrades.html

## Copyright

This CIP is licensed under CC0-1.0:
[Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

## Specification

This CIP standardizes one delegation primitive and the minimum change to
`DsoRules` needed to use it. The change is additive and compatible with
[Smart Contract Upgrades (SCU)][scu]. It is a foundation: it establishes the
authority path and leaves enforcement, classification, and audit surfaces to
later work.

Throughout this document, **the delegated party** is the role, and `voterParty`
is the ledger field that names it. The two terms refer to the same party.

The normative surface is:

- A new `VoteDelegation` template in `splice-dso-governance`, signed by the
  delegating SV, that names exactly one delegated party.
- Two relay choices on that template, covering the two governance actions a
  delegated party performs: it opens a vote request, and it casts or updates a
  vote.
- One optional trailing `voterParty` argument and one additional controller on
  each of `DsoRules_RequestVote` and `DsoRules_CastVote`.
- One read-only `DsoRules_Fetch` choice, mirroring the existing
  `AmuletRules_Fetch`, so a delegated party can read `DsoRules` as public
  reference data.

Tallying, quorum, confirmation, execution,
close, cooldown, automation, SV membership, reward weights, and the SV locking
framework of [CIP-0105][cip-0105] all keep their current behavior. Client, wallet,
read-API, deployment, and UX choices are implementation decisions that sit
downstream of this CIP.

### Affected contract surface

In `daml/splice-dso-governance/daml/Splice/` in the
[Splice](https://github.com/canton-network/splice) repository:

- `Splice/DsoRules/VoteDelegation.daml` (new module) — the `VoteDelegation`
  template, its two relay choices, and their result types.
- `Splice/DsoRules.daml` — the `DsoRules_Fetch` choice, plus the optional
  `voterParty` argument and additional controller on `DsoRules_RequestVote` and
  `DsoRules_CastVote`.

Unchanged: `Vote`, `VoteRequest`, `SvInfo`, `DsoRules_CloseVoteRequest`,
`DsoRules_ConfirmAction`, `DsoRules_ExecuteConfirmedAction`, `Confirmation`,
and every other `DsoRules` choice.

### The delegation template

```daml
template VoteDelegation
  with
    dso : Party
    sv : Party
    voterParty : Party -- ^ The party that can vote on behalf of the SV through the SV dApp.
  where
    signatory sv
    observer voterParty
```

Rules:

- `sv` is the sole signatory. The delegation is a unilateral declaration by the
  SV, not a bilateral agreement and not a DSO-governed record.
- `voterParty` is an observer so the delegated party can see the delegation on
  its own participant and use it as the contract to exercise.
- `dso` is a data field used for the `ForDso` contract-group-id check on reads.
  The DSO party is deliberately **not** a stakeholder: the delegation is a
  bilateral matter between the SV and the party it selects, and making it
  network-visible was rejected by Splice maintainers during review.
- `voterParty == sv` is permitted, and is the special case equivalent to
  today's behavior.
- There is no contract key and no on-ledger uniqueness guard. See
  [Open questions](#open-questions).
- Revocation is archival by the signatory SV. There is no separate revoke or
  rotate choice; an SV moving from voter A to voter B archives the first
  delegation and creates a second.

### Relay choices

Both choices are `nonconsuming`, controlled by `voterParty`, and take the
corresponding `DsoRules` choice argument as a whole. Each verifies that the
relayed argument names this delegation's SV and its `voterParty`,
reads `DsoRules` as public reference data, and then relays.

```daml
    nonconsuming choice VoteDelegation_RequestVote : VoteDelegation_RequestVoteResult
      with
        dsoRulesCid : ContractId DsoRules
        requestVote : DsoRules_RequestVote
      controller voterParty
      do
        require "Delegated request is for the delegating SV" (requestVote.requester == sv)
        require "Delegated request carries this delegation's voter party" (requestVote.voterParty == Some voterParty)
        fetchPublicReferenceData (ForDso dso) dsoRulesCid (DsoRules_Fetch with p = voterParty)
        result <- exercise dsoRulesCid requestVote
        pure VoteDelegation_RequestVoteResult with voteRequest = result.voteRequest

    nonconsuming choice VoteDelegation_CastVote : VoteDelegation_CastVoteResult
      with
        dsoRulesCid : ContractId DsoRules
        castVote : DsoRules_CastVote
      controller voterParty
      do
        require "Delegated vote is cast on behalf of the delegating SV" (castVote.vote.sv == sv)
        require "Delegated vote carries this delegation's voter party" (castVote.voterParty == Some voterParty)
        fetchPublicReferenceData (ForDso dso) dsoRulesCid (DsoRules_Fetch with p = voterParty)
        result <- exercise dsoRulesCid castVote
        pure VoteDelegation_CastVoteResult with voteRequest = result.voteRequest
```

The SV's authority reaches the `DsoRules` exercise through the delegation's
signatory. The delegated party's authority comes from the choice controller. A
delegation carries no other SV's authority, so it cannot vote on behalf of any
SV other than its signatory. The two `require` guards check the relayed argument
for consistency; the ledger's authorization model is what prevents cross-SV use.

These two choices are the whole delegated surface. SV node automation drives the
close of a vote request, and confirmation and execution stay with the operator,
so none of those needs a delegation.

### Scope of the delegated authority

A delegation:

- **Covers** every action the delegating SV may vote on. The ledger applies one
  authority path to every `ActionRequiringConfirmation` and draws no distinction
  between governance and operational actions. See
  [The votable surface](#the-votable-surface). A client that wants a narrower set
  enforces that restriction itself.
- **Authorizes** `VoteDelegation_RequestVote` and `VoteDelegation_CastVote`, and
  through them `DsoRules_RequestVote` and `DsoRules_CastVote`. Every other
  `DsoRules` choice stays with the SV party.
- **Lasts** until the SV archives it. Archival stops future exercises and leaves
  votes already cast in place; an SV that wants a cast vote changed casts again.
- **Binds** the delegating SV alone. A delegation carries no other SV's
  authority.
- **Records** `voterParty` on the transaction as a choice argument and as a
  co-authorizer. `Vote` is unchanged, so contract state and any read API served
  from contract state show the delegating SV as the voter.

### The votable surface

A delegation covers every `ActionRequiringConfirmation` constructor. Three carry
actions today:

| Constructor | Actions |
| --- | --- |
| `ARC_DsoRules` | `SRARC_AddSv`, `SRARC_ConfirmSvOnboarding`, `SRARC_OffboardSv`, `SRARC_UpdateSvRewardWeight`, `SRARC_GrantFeaturedAppRight`, `SRARC_RevokeFeaturedAppRight`, `SRARC_UpdateFeaturedAppRight`, `SRARC_SetConfig`, `SRARC_CreateUnallocatedUnclaimedActivityRecord`, `SRARC_CreateExternalPartyAmuletRules`, `SRARC_CreateTransferCommandCounter`, `SRARC_CreateBootstrapExternalPartyConfigStateInstruction` |
| `ARC_AmuletRules` | `CRARC_SetConfig`, `CRARC_MiningRound_StartIssuing`, `CRARC_MiningRound_Archive`, `CRARC_StartProcessingRewardsV2`, and the three deprecated `CRARC_*FutureAmuletConfigSchedule` actions |
| `ARC_AnsEntryContext` | `SRARC_CollectInitialEntryPayment`, `SRARC_RejectEntryInitialPayment` |

`ExtActionRequiringConformation` is an extensibility placeholder and carries no
action today. A future action added under it is delegable from the moment it
exists, without a change to this CIP.

Two of these actions reach much further than the rest. `SRARC_SetConfig` sets
every field of `DsoRulesConfig`, including `voteRequestTimeout`,
`voteCooldownTime`, `decentralizedSynchronizer`, and the scheduled-upgrade
fields. `CRARC_SetConfig` sets every field of `AmuletConfig`, including the
nested `transferConfig`, `issuanceCurve`, `packageConfig`, and `rewardConfig`
records. A delegation therefore reaches every network parameter those two actions
govern, and the ledger applies no per-field distinction within either.

Several listed actions are driven by SV node automation rather than by a person:
the mining-round and reward-processing actions, and both ANS payment actions. The
ledger does not separate them from the rest, so a delegated party may vote on
them.

### `DsoRules` changes

`DsoRules_RequestVote` and `DsoRules_CastVote` each gain one optional argument,
appended last for SCU compatibility, and the delegated party as an
additional controller:

```daml
    nonconsuming choice DsoRules_RequestVote : DsoRules_RequestVoteResult
      with
        requester : Party
        action : ActionRequiringConfirmation
        reason : Reason
        voteRequestTimeout : Optional RelTime
        targetEffectiveAt : Optional Time
        voterParty : Optional Party
      controller requester, voterParty

    nonconsuming choice DsoRules_CastVote : DsoRules_CastVoteResult
      with
        requestCid : ContractId VoteRequest
        vote : Vote
        voterParty : Optional Party
      controller vote.sv, voterParty
```

A direct exercise by an SV passes `voterParty = None`, yielding the existing
controller set and existing behavior. A delegated exercise passes
`voterParty = Some voter`, making the transaction co-authorized by the SV and
the delegated party. `Optional Party` converts to the controller list, so `None`
contributes no controller.

The choice bodies are unchanged. There is no branch on `voterParty`, no
eligibility check, and no attribution written into `Vote`.

`DsoRules` also gains a read-only fetch choice:

```daml
    nonconsuming choice DsoRules_Fetch : DsoRules
      with
        p : Party
      controller p
      do
        pure this
```

This exists because `fetchReferenceData (ForDso dso) dsoRulesCid` cannot be
authorized from inside the delegation: `dso` is the sole `DsoRules`
stakeholder, and the delegation carries only `sv` and `voterParty`. The choice
mirrors the existing `AmuletRules_Fetch` and allows a read with `readAs` but
not `actAs` claims. Going through `fetchPublicReferenceData` rather than a bare
exercise also keeps the DSO-party check (the `ForDso dso` contract-group-id
check) in the read path.

### Delegation discovery

A delegated party's client discovers its delegation by a query against the active
contract set for `VoteDelegation` contracts where it is the `voterParty`. The
delegating SV party and the delegation contract id are read from that contract,
not configured.

This is a client convention, not a ledger invariant. Nothing on the ledger
prevents an SV from signing several delegations, or several SVs from delegating
to the same delegated party. See [Open questions](#open-questions).

### Submission from a non-SV participant

The delegated party's participant does not host DSO contracts. A delegated
submission therefore attaches the contracts it reads as explicitly disclosed
contracts: `DsoRules` for both choices, and the target `VoteRequest` for a
cast. Those can be sourced from Scan. Missing disclosure surfaces as
`PERMISSION_DENIED` rather than a domain error.

`DsoRules_CastVote` archives and recreates the `VoteRequest`, so a client must
re-resolve the current contract id immediately before each cast; a stale id
fails with `CONTRACT_NOT_FOUND`.

The delegated party can therefore be hosted anywhere and hold its key in any
conforming CIP-0103 wallet, provided a read API supplies the contract ids and
disclosed contracts.
This CIP does not standardize that read API, and does not require the
delegated party's participant to be independent of the SV node — it only avoids
precluding it.

### One vote per SV

Unchanged. `VoteRequest.votes` remains keyed by SV display name, `Vote.sv`
remains the delegating SV, and both paths write the same slot. A delegation
creates no additional voting unit and confers no additional weight.
`DsoRules_CloseVoteRequest` continues to count at most one vote per SV, with
its existing semantics.

## Relationship to in-flight upstream work

Two open Splice drafts restructure which party holds an SV's vote. This CIP is
scoped to compose with either outcome rather than to compete with them, and the
authors expect its argument shape to be reconciled with them before merge.

**[canton-network/splice#6247][pr6247] — Dso Governance POC (Digital Asset).**
Introduces `SvRightOwner` with a `rightOwnerParty` and separate `voteWeight` /
`rewardWeight`, a `VoteType = RightOwnerVote | OperatorVote` distinction with a
`voteType` classifier over `ActionRequiringConfirmation`, and weight-based
tallying (`totalVoteWeight`, `requiredVoteWeight`, `votersWithWeight`). Under
that model the voting party is the rights owner rather than the node operator,
and `Vote.sv` may hold either.

#6247 decides **which party holds an SV's vote**, and whether an action is voted
on by rights owners or operators. This CIP decides **how the party that holds the
vote can let another party exercise it**. The reference implementation draws the
same boundary: it
declares changes to the voting entity out of scope. Delegation remains useful
under #6247 because `rightOwnerParty` is still a single ledger party, and
external signing still requires that party to be either wallet-held or
delegated. Applied to #6247, the delegating signatory becomes the rights owner
rather than the operator; the template and both relay choices are otherwise
unchanged.

Two concrete points of contact need resolving with the maintainers:

1. **Argument slot.** #6247 appends `rightOwnerCid : Optional (ContractId SvRightOwner)`
   to `DsoRules_RequestVote` (after `targetEffectiveAt`) and to
   `DsoRules_CastVote` (after `vote`) — the same trailing position this CIP
   appends `voterParty : Optional Party`. Whichever lands first fixes the
   order; the second appends after it. Both are optional and additive, so they
   can coexist, but the ordering must be agreed rather than discovered at
   merge.
2. **Controller shape.** #6247 keeps `controller vote.sv` and validates the
   voting party through `getAndValidateVotingParty`. This CIP adds the
   delegated party as a co-controller. If both land, a delegated rights-owner
   vote carries `rightOwnerCid = Some _` and `voterParty = Some _`
   simultaneously, and the interaction between the two validations should be
   specified rather than left implicit.

**[canton-network/splice#5567][pr5567] — SV weight management fully on-ledger
(IntellectEU).** Moves reward weights off `DsoRules.svs` into `HostedSv`
contracts and separates SV node operators from reward-bearing hosted SVs. This
CIP does not touch reward weights, reward coupons, beneficiaries, or
`DsoRules.svs` membership, and has no expected interaction beyond both drafts
editing `DsoRules.daml`.

[pr6247]: https://github.com/canton-network/splice/pull/6247
[pr5567]: https://github.com/canton-network/splice/pull/5567

## Trust model

An SV node operator controls the SV party, so it can already choose who casts
the SV's governance votes. A delegation records that choice on the ledger and
makes each delegated exercise co-authorized, where before the substitution was
invisible. The operator keeps the ability to archive a delegation or vote
directly at any time.

What the ledger enforces:

- A delegation carries only its signatory SV's authority, so it cannot be used
  to vote for a different SV. This is the ledger authorization model, not an
  in-Daml check.
- A delegated exercise is co-authorized by the SV and the delegated party, and
  both appear in the transaction's authorizing set.
- The delegated party gains authority over exactly two choices. It cannot
  confirm actions, execute confirmed actions, onboard SVs, or exercise any
  other `DsoRules` choice.
- Revocation is immediate on archival; a subsequent delegated exercise fails
  with `CONTRACT_NOT_FOUND`.

What the ledger does not enforce, and what implementers should therefore not
rely on:

What implementers are responsible for:

- **Restricting which actions a delegated party votes on.** The ledger applies
  one authority path to every action, so a client or an operational process
  enforces any narrower policy an SV wants.
- **Reading attribution from the transaction record.** `voterParty` appears as a
  choice argument and co-authorizer on the `DsoRules` exercise, and the update
  history is where an auditor finds it. Contract state names the delegating SV
  as the voter.
- **Selecting among delegations.** A delegated party may hold delegations from
  two SVs, and an SV may have two live delegations. Both are valid ledger
  states, and the client decides how to present and choose between them.
- **Establishing that a delegation existed.** The DSO is not a stakeholder, so
  a third party cannot enumerate delegations from the ledger. Anyone relying on
  the delegation as evidence of intent obtains it from the SV or from the
  transaction record of the exercises made under it.

## Security considerations

- **Cross-SV use is impossible.** A delegation carries only its signatory SV's
  authority, so it cannot drive a `DsoRules` exercise on behalf of any other
  SV. This is the ledger authorization model rather than an in-Daml check; the
  `require` guards on the relay choices fail first and produce a clearer error.
  Covered by `testVoteDelegationRejectsMismatch`.
- **Wrong-DSO use is rejected.** Both relay choices read `DsoRules` through
  `fetchPublicReferenceData (ForDso dso)`, so a delegation cannot be used
  against another DSO's `DsoRules`. Covered by
  `testVoteDelegationRejectsWrongDso`.
- **Authority is limited to two choices.** A delegated party cannot exercise
  `DsoRules_ConfirmAction`, `DsoRules_ExecuteConfirmedAction`,
  `DsoRules_CloseVoteRequest`, amulet-price votes, onboarding, or any other
  `DsoRules` choice. It has no SV authority beyond opening and casting the SV's
  governance vote.
- **Deadline and cooldown are inherited, not re-implemented.** Both paths
  execute the same `DsoRules` choice bodies, so the vote window and the per-SV
  cooldown apply identically to direct and delegated exercises.
- **Revocation is immediate.** After the SV archives the delegation, any later
  delegated exercise fails with `CONTRACT_NOT_FOUND`. There is no grace period,
  and no separate revoke choice that must be authorized.
- **The delegated party votes on the SV's full action surface.** A delegation
  covers every action the SV can vote on, so a client should show that surface
  and its consequences to the delegated party rather than assume a narrower set.
  `SRARC_OffboardSv` and both `SetConfig` actions deserve particular care.
- **Archival is forward-looking.** It stops later exercises and leaves votes
  already cast in place, because `DsoRules_CloseVoteRequest` counts votes by
  represented SV and does not consult delegation state. An SV that wants a cast
  vote changed casts again.
- **The delegation does not constrain the operator.** An SV node operator can
  archive a delegation and create another, or vote directly, at any time. The
  delegation records an intention; it does not bind the party that can revoke
  it.

## Motivation

Governance voting and node operation are different responsibilities held, in
many SV organizations, by different people.

The current model does not allow that separation. `DsoRules_RequestVote` and
`DsoRules_CastVote` are controlled by the SV party, and the SV party is the node
operator's party. An organization that wants a policy owner, committee
representative, or executive to cast its governance votes directly must give
that person node-operator credentials, or route their intent through an operator
who signs on their behalf. Neither option is acceptable. The first grants far
more authority than voting needs. The second leaves no ledger evidence that the
vote reflects anyone's decision except the operator's.

External signing makes the gap more visible. [CIP-0103][cip-0103] establishes
how an external party submits through a wallet with disclosed contracts. That
mechanism applies directly to governance voting: a governance representative
holds a key in a wallet, reviews a proposal, and signs it. But the mechanism has
nothing to attach to. The only party that can cast an SV's vote is the SV party
itself, node automation holds that party, and it is not a wallet-held key.

This CIP adds the smallest on-ledger construct that fills that gap: a way for an
SV to name a party that may exercise its governance vote. It does not redesign
governance, change how much say each SV has, or move any existing authority.
Many SV organizations already separate these two roles internally. This CIP lets
the ledger express and authorize that separation.

## Rationale

This CIP is kept as small as possible. Every element is either necessary to
authorize a delegated vote, or is a consequence of SCU compatibility.
The predecessor design added templates and choices for enforcement,
auditability, and lifecycle management. This CIP leaves those out and records
each remaining concern as an open question below.

**Why a separate template rather than new `DsoRules` choices.** The delegation
needs a signatory that is the SV, and the SV's authority must reach the
`DsoRules` exercise. A guard on a contract that the SV signed satisfies both
conditions at once. The SV's authority comes from the signatory, and the
delegated party's authority comes from the choice controller, so the check that
the two agree lives on the same contract as the authority it limits. The authors
considered dedicated choices on `DsoRules`, such as
`DsoRules_CastGovernanceVote`, and rejected them: they duplicate the cast logic,
and each duplicate must then track the original for cooldown, deadlines, and
well-formedness.

**Why optional trailing arguments on the existing choices.** `DsoRules` still
has to know that an exercise was co-authorized, so that the delegated party
appears in the authorizing set and the transaction record. An optional trailing
argument plus an additional controller achieves this with no branch in the
choice body and no change for existing callers, and satisfies the SCU rule that
new fields be optional and appended.
`Optional Party` converts to a controller list, so `None` contributes no
controller and reproduces the previous controller set exactly.

**Why the SV signs and not the DSO.** A delegation is an agreement between an SV
and the party that it selects. Splice maintainers concluded, during review of the
predecessor design, that the network does not need to know about that agreement,
and that it needs no DSO signature, no DSO-side lifecycle choices, and no
network-wide visibility. A DSO signature also requires three additional functions
that this CIP does not need: a default delegation created at SV onboarding, a
governance action to change the delegated party, and cleanup for duplicate
delegations. The SV that signs a delegation can already replace the delegated
party at any time, so those functions add no control.

**Why no action classification.** The predecessor design divided
`ActionRequiringConfirmation` into governance-voter-eligible and operator-only
sets, and rejected the wrong path on each. This CIP removes that division. An SV
that trusts a party to cast its votes has already expressed exactly that trust. A
Daml-level allowlist also implies an enforcement guarantee that the model does
not give, because the operator can bypass the allowlist and vote directly. The
question of which actions need which authority is a real one. Digital Asset is
now answering it upstream in [#6247][pr6247]'s `voteType`, and a second
classification here would compete with that one. The same reasoning applies to
the finer-grained variant the predecessor scope also carried, which classified
individual `DsoRules` and Amulet config fields as operational, governance, or
fixed: a classification model belongs wherever the network decides authority
paths, not in a delegation primitive.

**Why the DSO-party check is a fetch choice.** A new `dso` field on the relayed
`DsoRules` choice argument cannot do this check: SCU rules do not allow a new
non-optional field on an existing type. `fetchReferenceData (ForDso dso) dsoRulesCid` is also unavailable, because
`dso` is the sole `DsoRules` stakeholder and the delegation carries only `sv`
and `voterParty`, so the fetch aborts unauthorized. A read-only
`DsoRules_Fetch` choice mirroring the existing `AmuletRules_Fetch`, read through
`fetchPublicReferenceData`, resolves this and reduces the DSO-party check to the
`ForDso` contract-group-id check already built into that helper.

### CIP-0103 compatibility

[CIP-0103][cip-0103] defines the dApp-to-wallet API. It does not prescribe
on-ledger contract patterns, but it does establish that external parties submit
via prepare/execute with disclosed contracts. The surface here is deliberately
compatible:

- The relay choices are controlled by `voterParty`, which may be an
  externally-hosted party whose key lives in any conforming wallet.
- The contracts a delegated submission must read — `DsoRules`, the target
  `VoteRequest`, and the delegation itself — can all be supplied as disclosed
  contracts, so the delegated party's participant need not host DSO contracts.
- Nothing in the delegated path requires visibility on contracts unique to the
  SV node.

`Requires: CIP-0103` is deliberately not asserted. The on-ledger surface is
independently useful — a delegated party hosted on the SV's own participant
works without any wallet — so the relationship is compatibility, not
dependence.

### Alternatives considered

- **DSO-signed binding with confirmation-quorum rotation.** The predecessor
  design ([cips#210][pr210], withdrawn): `SvGovernanceVoter` signed by the DSO,
  observed by the SV, created by default at `DsoRules_AddSv`, rotated through a
  new `SRARC_RotateGovernanceVoter` action and the standard confirmation-quorum
  flow. Splice maintainers rejected it as too large for the problem: the
  delegation is bilateral, and a DSO signature implies an enforcement guarantee
  that the trust model does not actually deliver.
- **Store the delegated party on `SvInfo`.** Rejected: it couples governance
  voting to the SV membership and operational-identity record, it widens the
  disclosure surface of operator records, and it makes a change of delegated
  party into an edit of SV membership.
- **Hardcoded action allowlist with a strict role split.** The predecessor's
  `isGovernanceVoterAction`, under which each action class could be voted on
  through exactly one path and the other path rejected. Rejected, per *Why no
  action classification* above.
- **Configurable action allowlist.** Rejected in the predecessor design and
  still rejected: a delegated party could vote to expand its own authority. If
  classification returns, it belongs in Daml or in the upstream `voteType`, not
  in mutable config.
- **Attribution fields on `Vote`.** The predecessor added `castBy`,
  `castByRole`, and `bindingCid`, because the vote record itself should say
  which authority path wrote it. This CIP drops those fields to keep the change
  small. The landed design therefore accepts attribution from the transaction
  history alone, which the predecessor explicitly rejected. See open question 3.
- **DSO as observer so Scan ingests the delegation.** An earlier revision did
  this, and Splice maintainers asked for its removal: a delegation is a
  bilateral agreement, and the delegated party is already visible as a
  controller on the `DsoRules` exercise, so a public delegation contract adds
  disclosure without adding information.
- **Contract key on the delegation.** Considered as a way to enforce one live
  delegation per SV. Deferred; see open question 2.
- **Propose-Accept on the delegation.** Adds a consent step with no Phase 1
  benefit. A later amendment can add it without invalidating the
  unilateral-declaration semantics.
- **Delegation of vote closing, confirmation, or execution.** Splice maintainers
  rejected this: SV node automation drives the close, and confirmation and
  execution are operator responsibilities. Only the two actions that a delegated
  party performs from a UI are
  delegated.

[pr210]: https://github.com/canton-foundation/cips/pull/210

## Backwards compatibility

The change is additive and behavior-preserving when unused. An SV that creates
no delegation is unaffected in every respect.

- **`Vote` and `VoteRequest` are unchanged.** No new fields, no key change, no
  change to how votes are stored or counted.
- **`DsoRules_RequestVote` and `DsoRules_CastVote` gain optional trailing
  arguments.** `voterParty = None` reproduces the previous controller set and
  the previous behavior exactly. Note that this is compatible on the ledger, not
  source-compatible for Daml callers: record-syntax call sites must add the
  field. The reference implementation updates the in-repo callers accordingly
  (`SvApp.scala`, `CopyVotesTrigger.scala`, and the `DsoTestUtils.daml`
  helpers), each passing `None`.
- **Every existing call still succeeds.** Unlike the predecessor design,
  neither choice rejects any action it previously accepted. There is no
  eligibility check, so no existing caller can be broken by one.
- **`DsoRules_CloseVoteRequest` is unchanged**, including its result type. The
  predecessor's optional `staleBindingVoters` field is not part of this CIP.
- **New `DsoRules_Fetch` choice** is read-only, returns `this`, and is
  controlled by its argument party. It adds an on-ledger read path that did not
  previously exist for `DsoRules`, mirroring `AmuletRules_Fetch`
  (`Splice/AmuletRules.daml`). It confers no write authority, and the contract
  content it exposes is already served publicly by Scan's `/v0/dso` endpoint.
- **Package version.** `splice-dso-governance` moves from 0.1.28 to 0.1.29. The
  new template, new choice, and appended optional arguments are all additive
  under SCU.

**Deployment prerequisite.** Because `DsoRules_RequestVote` and
`DsoRules_CastVote` execute on a DSO-signatory contract confirmed by every SV
participant, the delegated path is not usable on a multi-SV network until
`splice-dso-governance` 0.1.29 is vetted on all SV participants and
`AmuletConfig.packageConfig.dsoGovernance` is raised to it. On the current
model that package-version change is itself a governance vote, so adopting this
CIP requires one non-delegated vote before the first delegated vote is possible.
Single-SV and development networks have no such constraint.

## Reference implementation

The reference implementation is **two stacked pull requests** on the
[canton-network/splice-sv-voting-dapp][fork] feature fork of Splice. Both are
unmerged as of 2026-08-17, and the Daml PR is awaiting maintainer approval.

| PR | Contents | State |
| --- | --- | --- |
| [#12][pr12] | The Daml change: `VoteDelegation`, `DsoRules_Fetch`, the optional `voterParty` arguments, and Daml Script tests | Open, CI green, awaiting maintainer approval |
| [#19][pr19] | Reference client: the SV frontend in a mode that logs in with a CIP-0103 wallet, reads from Scan, and submits through the delegation; plus the cross-participant CI integration test | Open, stacked on #12 |

The integration-test work was raised as two further PRs, [#20][pr20] and
[#24][pr24], only so each could be shown green in CI independently. Both have
since been squashed into #19 and are not separate deliverables. #20's
`VoteDelegationIntegrationTest` was subsequently dropped in favor of #24's
`SvDappModeFrontendIntegrationTest`, which drives the same delegated ledger path
through the real UI and wallet gateway rather than through direct ledger
commands.

Artifacts:

- `daml/splice-dso-governance/daml/Splice/DsoRules/VoteDelegation.daml`
- `daml/splice-dso-governance/daml/Splice/DsoRules.daml`
- `daml/splice-dso-governance-test/daml/Splice/Scripts/TestGovernance.daml`
- `apps/sv/frontend/src/dapp/{discoverVoteDelegation,voteDelegationCommands,voteDelegationSubmission}.ts`
- `apps/app/src/test/scala/.../integration/tests/SvDappModeFrontendIntegrationTest.scala`
- `apps/app/src/test/scala/.../integration/tests/WalletGatewayTestFixture.scala`

### Test matrix

| Concern | Test |
| --- | --- |
| Delegated request creates a `VoteRequest` with the SV as `requester` | `testVoteDelegationRequestVote` |
| Delegated cast writes the SV's vote slot | `testVoteDelegationCastVote` |
| A delegation cannot be used on behalf of another SV | `testVoteDelegationRejectsMismatch` |
| A delegation cannot be used against another DSO's `DsoRules` | `testVoteDelegationRejectsWrongDso` |
| A request that names a party other than the delegation's `voterParty` is rejected | `testVoteDelegationRejectsWrongVoterParty` |
| A cast that names an SV other than the delegation's is rejected | `testVoteDelegationCastVoteRejectsWrongSv` |
| A cast that names a party other than the delegation's `voterParty` is rejected | `testVoteDelegationCastVoteRejectsWrongVoterParty` |
| A delegated party on a participant that is not the SV's, holding its key in a CIP-0103 wallet, casts through the UI with disclosed `DsoRules` and `VoteRequest`, and the ballot is recorded against the SV and not against the voter | `SvDappModeFrontendIntegrationTest`: "connect wallet, discover `VoteDelegation`, and cast a vote attributed to the SV" |
| Delegation discovery by ACS query; zero or several matches surface an error | `discover-vote-delegation.test.ts` |
| Command construction, disclosure, stale-contract-id re-resolution, wallet rejection | `vote-delegation-commands.test.ts`, `vote-delegation-submission.test.ts` |

The cross-participant test allocates the voter party through the reference
wallet gateway (`@canton-network/wallet-gateway-remote`) on a validator
participant that does not host DSO contracts, has the SV create the
`VoteDelegation` for it, seeds an open `VoteRequest` from a second SV, and drives
the connect and submit CIP-0103 approvals in the browser. It then asserts
on-ledger that the delegating SV's `Vote` is present with `accept = true` and
that no `Vote` exists for the voter party.

**Not yet covered.**

- **Third-party wallets.** The end-to-end path is automated against the
  reference wallet gateway only. No conforming partner wallet is exercised in
  CI, and doing so requires the wallet to offer a headless or scriptable
  approval mode.
- **The delegated request path across participants.** `VoteDelegation_RequestVote`
  is covered by Daml Script (`testVoteDelegationRequestVote`) and by the
  reference client's unit tests, but not by a cross-participant integration
  test. Only the cast path is.
- **Negative cases across participants.** Wrong-SV and wrong-`voterParty`
  rejections are asserted at the Daml Script level. The cross-participant
  negative case that `VoteDelegationIntegrationTest` carried went away when that
  test was dropped, and was not reproduced in the frontend test.

[fork]: https://github.com/canton-network/splice-sv-voting-dapp
[pr12]: https://github.com/canton-network/splice-sv-voting-dapp/pull/12
[pr19]: https://github.com/canton-network/splice-sv-voting-dapp/pull/19
[pr20]: https://github.com/canton-network/splice-sv-voting-dapp/pull/20
[pr24]: https://github.com/canton-network/splice-sv-voting-dapp/pull/24

## Open questions

These are unresolved and are the intended focus of review.

1. **Argument ordering against #6247.** Should `voterParty` or `rightOwnerCid`
   land first on `DsoRules_RequestVote` / `DsoRules_CastVote`, and does DA want
   both, or does `SvRightOwner` make the delegation redundant in their target
   state? The answer determines whether this CIP survives in its current shape.
   **Owner: ask Moritz on PR #12.**
2. **Delegation uniqueness.** Should "at most one live delegation per SV" and
   "at most one delegating SV per delegated party" be ledger invariants, or
   should they remain client conventions? If they must be ledger invariants, the
   candidate mechanisms are a contract key on `(sv)`, a DSO-owned registry, or a
   duplicate-create guard. The predecessor design deferred this question. The
   landed design has no guard at all, and the reference client raises an error
   when it finds zero or several matches.
3. **Attribution.** Is transaction-record-only attribution acceptable, or
   should `Vote` carry the casting party? Recording the caster is a narrower
   change than making the delegation contract network-visible, which is what
   review actually objected to. This bears directly on any audit view built
   over governance history.
4. **Delegation visibility for audit.** If attribution stays out of contract
   state, is a read API over the update history the right place to expose "this
   vote was cast by a delegate", and does that belong in this CIP or a
   separate one?
5. **Should this be a CIP at all in DA's view?** The change is additive,
   optional, and behavior-preserving when unused. Maintainers may prefer it as
   a plain Splice PR. Worth settling before investing in review cycles.
6. **Author of record.** Eric authored the withdrawn predecessor and the
   reference implementation; the byline above lists both authors and should be
   confirmed.
7. **Two Super Validator sponsors.** CIP-0000 requires support from at least
   two Super Validators to move from Draft to Proposed. Neither is identified.
8. **cip-discuss posting.** Required before an editor assigns a number, and
   outstanding since the Foundation asked on 2026-06-17. `Post-History` is
   blank pending that.

## Changelog

- **2026-08-17:** Corrected the Reference implementation section per Eric's
  review. The implementation is two stacked PRs, not three: #20 and #24 were
  split out only to show CI green independently and are squashed into #19. #20's
  `VoteDelegationIntegrationTest` has since been dropped in favor of #24's
  `SvDappModeFrontendIntegrationTest`, so the test matrix, artifact list, and
  coverage gaps were rewritten against what the branch actually contains — which
  means the end-to-end wallet path is no longer an open gap, and the
  cross-participant request and negative cases now are.
- **2026-08-05:** Initial draft. Replaces the withdrawn
  `cip-XXXX-SV-Governance-Voter` draft
  ([canton-foundation/cips#210](https://github.com/canton-foundation/cips/pull/210),
  closed 2026-08-04), which specified a DSO-signed `SvGovernanceVoter` binding,
  a hardcoded action allowlist, confirmation-quorum rotation, `Vote`
  attribution fields, close-time staleness filtering, and binding garbage
  collection. That design was narrowed out during Splice review; its reference
  implementation ([canton-network/splice#5533][pr5533]) was closed unmerged on
  2026-06-10. This draft specifies only what the current reference
  implementation contains.

[pr5533]: https://github.com/canton-network/splice/pull/5533
