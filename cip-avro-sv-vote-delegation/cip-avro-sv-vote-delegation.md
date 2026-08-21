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

Splice uses the SV operator party as both the node-automation identity and the
governance-voting identity. An SV organization that wants a policy owner or
executive to vote on its behalf must therefore give that person the SV's own
credentials, which carry full authority over the node.

This CIP lets an SV delegate its governance vote to a different party. An SV
signs a `VoteDelegation` contract naming one `voterParty`. That party exercises
two relay choices on the delegation, which in turn exercise the existing
`DsoRules_RequestVote` and `DsoRules_CastVote` on the SV's behalf. Both choices
gain one optional `voterParty` argument and take the delegated party as an
additional controller, so a delegated exercise is co-authorized and recorded as
such. An SV revokes a delegation by archiving the contract.

The vote remains the SV's own: it occupies the SV's existing slot in
`VoteRequest.votes`, carries no additional weight, and creates no second voting
unit. The change is additive and behavior-preserving when unused. The surface is
compatible with the external-party submission flow of [CIP-0103][cip-0103], so
the delegated party can hold its key in any conforming wallet.

[cip-0103]: https://github.com/canton-foundation/cips/blob/main/cip-0103/cip-0103.md
[cip-0105]: https://github.com/canton-foundation/cips/blob/main/cip-0105/cip-0105.md
[scu]: https://docs.daml.com/upgrade/smart-contract-upgrades.html

## Copyright

This CIP is licensed under CC0-1.0:
[Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

## Specification

This CIP standardizes one delegation primitive and the minimum change to
`DsoRules` needed to use it. All changes are additive and compatible with
[Smart Contract Upgrades (SCU)][scu]. The normative surface is a new
`VoteDelegation` template with two relay choices, one optional trailing
`voterParty` argument and one additional controller on each of
`DsoRules_RequestVote` and `DsoRules_CastVote`, and one read-only
`DsoRules_Fetch` choice.

Tallying, quorum, confirmation, execution, close, cooldown, automation, SV
membership, reward weights, and the SV locking framework of [CIP-0105][cip-0105]
all keep their current behavior. `Vote`, `VoteRequest`, `SvInfo`, and every
`DsoRules` choice other than the two named above are unchanged.

### The delegation template

```daml
template VoteDelegation
  with
    dso : Party
    sv : Party
    voterParty : Party
  where
    signatory sv
    observer voterParty
```

`sv` is the sole signatory: the delegation is a unilateral declaration by the
SV, not a bilateral agreement and not a DSO-governed record. `voterParty` is an
observer so the delegated party can see the delegation and exercise it. `dso` is
a data field used for the `ForDso` contract-group-id check on reads; the DSO
party is deliberately not a stakeholder.

`voterParty == sv` is permitted, and is equivalent to today's behavior. There is
no contract key and no on-ledger uniqueness guard (see
[Open questions](#open-questions)). Revocation is archival by the signatory SV;
there is no separate revoke or rotate choice.

### Relay choices

Both relay choices are `nonconsuming` and controlled by `voterParty`. Each takes
the corresponding `DsoRules` choice argument as a whole, checks that the relayed
argument names this delegation's SV and its `voterParty`, reads `DsoRules` as
public reference data, and relays:

```daml
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

`VoteDelegation_RequestVote` has the same shape, relaying a
`DsoRules_RequestVote` and checking `requestVote.requester == sv`.

The SV's authority reaches the `DsoRules` exercise through the delegation's
signatory; the delegated party's authority comes from the choice controller. A
delegation carries no other SV's authority, so it cannot be used to vote for a
different SV — that is the ledger authorization model, not the `require` guards,
which only produce a clearer error.

These two choices are the whole delegated surface. SV node automation drives the
close of a vote request, and confirmation and execution stay with the operator,
so none of those needs a delegation.

### `DsoRules` changes

`DsoRules_RequestVote` and `DsoRules_CastVote` each gain one optional argument,
appended last for SCU compatibility, and the delegated party as an additional
controller:

```daml
    nonconsuming choice DsoRules_RequestVote : DsoRules_RequestVoteResult
      with
        ... -- existing fields unchanged
        voterParty : Optional Party
      controller requester, voterParty

    nonconsuming choice DsoRules_CastVote : DsoRules_CastVoteResult
      with
        ... -- existing fields unchanged
        voterParty : Optional Party
      controller vote.sv, voterParty
```

A direct exercise by an SV passes `voterParty = None`, yielding the existing
controller set and existing behavior, because `Optional Party` converts to the
controller list and `None` contributes no controller. A delegated exercise
passes `Some voter`, making the transaction co-authorized by the SV and the
delegated party. The choice bodies are unchanged: there is no branch on
`voterParty`, no eligibility check, and no attribution written into `Vote`.

`DsoRules` also gains a read-only `DsoRules_Fetch` choice, controlled by its
argument party and returning `this`, mirroring the existing `AmuletRules_Fetch`.
It exists because `fetchReferenceData (ForDso dso) dsoRulesCid` cannot be
authorized from inside the delegation — `dso` is the sole `DsoRules` stakeholder,
and the delegation carries only `sv` and `voterParty`.

### Scope and limits of the delegated authority

A delegation authorizes exactly `VoteDelegation_RequestVote` and
`VoteDelegation_CastVote`, and through them `DsoRules_RequestVote` and
`DsoRules_CastVote`. Every other `DsoRules` choice stays with the SV party.

Within that, it covers **every** `ActionRequiringConfirmation` the delegating SV
may vote on. The ledger applies one authority path to all of them and draws no
distinction between governance and operational actions, so a client wanting a
narrower set enforces that itself. Note that `SRARC_SetConfig` and
`CRARC_SetConfig` between them reach every field of `DsoRulesConfig` and
`AmuletConfig`, and that several votable actions are driven by SV node
automation rather than by a person.

`VoteRequest.votes` remains keyed by SV display name and `Vote.sv` remains the
delegating SV, so both paths write the same slot and
`DsoRules_CloseVoteRequest` continues to count at most one vote per SV.

### Discovery and submission

A delegated party's client finds its delegation by querying the active contract
set for `VoteDelegation` contracts naming it as `voterParty`; the SV party and
contract id are read from that contract rather than configured. Nothing on the
ledger prevents an SV from signing several delegations, or several SVs from
delegating to the same party, so this is a client convention rather than a
ledger invariant.

Because the delegated party's participant need not host DSO contracts, a
delegated submission attaches the contracts it reads as explicitly disclosed
contracts: `DsoRules` for both choices, and the target `VoteRequest` for a cast.
Missing disclosure surfaces as `PERMISSION_DENIED`. `DsoRules_CastVote` archives
and recreates the `VoteRequest`, so a client must re-resolve the current contract
id immediately before each cast; a stale id fails with `CONTRACT_NOT_FOUND`.
This CIP does not standardize the read API that supplies those contracts.

## Motivation

Governance voting and node operation are different responsibilities, held in
many SV organizations by different people. The current model does not allow that
separation: `DsoRules_RequestVote` and `DsoRules_CastVote` are controlled by the
SV party, and the SV party is the node operator's party. An organization that
wants a policy owner or executive to cast its governance votes must either give
that person node-operator credentials, which grants far more authority than
voting needs, or route their intent through an operator who signs for them,
which leaves no ledger evidence that the vote reflects anyone's decision except
the operator's.

External signing makes the gap more visible. [CIP-0103][cip-0103] establishes
how an external party submits through a wallet with disclosed contracts, which
applies directly to governance voting — but it has nothing to attach to, because
the only party that can cast an SV's vote is the SV party itself, held by node
automation rather than as a wallet-held key.

This CIP adds the smallest on-ledger construct that closes that gap. It does not
redesign governance, change how much say each SV has, or move any existing
authority.

## Rationale

**Why a separate template rather than new `DsoRules` choices.** The delegation
needs a signatory that is the SV, and the SV's authority must reach the
`DsoRules` exercise; a guard on a contract the SV signed satisfies both at once.
Dedicated choices such as `DsoRules_CastGovernanceVote` were rejected because
they duplicate the cast logic, and each duplicate must then track the original
for cooldown, deadlines, and well-formedness.

**Why optional trailing arguments on the existing choices.** `DsoRules` still
has to know an exercise was co-authorized, so the delegated party appears in the
authorizing set. An optional trailing argument plus an additional controller
achieves that with no branch in the choice body, no change for existing callers,
and compliance with the SCU rule that new fields be optional and appended.

**Why the SV signs and not the DSO.** A delegation is an agreement between an SV
and the party it selects. Splice maintainers concluded during review of the
predecessor design that the network does not need to know about it. A DSO
signature would also require a default delegation at onboarding, a governance
action to change the delegated party, and duplicate cleanup — none of which adds
control, since the signing SV can already replace the delegated party at will.

**Why no action classification.** The predecessor design split
`ActionRequiringConfirmation` into voter-eligible and operator-only sets. An SV
that trusts a party to cast its votes has already expressed exactly that trust,
and a Daml-level allowlist implies an enforcement guarantee the model does not
give, because the operator can bypass it and vote directly. Which actions need
which authority is a real question, but Digital Asset is answering it upstream in
[splice#6247][pr6247]'s `voteType`, and a second classification here would
compete with it.

**CIP-0103 compatibility.** The relay choices are controlled by `voterParty`,
which may be an externally-hosted party, and every contract a delegated
submission reads can be supplied as a disclosed contract. `Requires: CIP-0103`
is deliberately not asserted: the on-ledger surface is independently useful, so
the relationship is compatibility rather than dependence.

### Alternatives considered

- **DSO-signed binding with confirmation-quorum rotation** — the withdrawn
  predecessor design ([cips#210][pr210]), rejected by Splice maintainers as too
  large for the problem.
- **Storing the delegated party on `SvInfo`** — rejected: it couples governance
  voting to the SV membership record and makes a change of delegated party an
  edit of SV membership.
- **Action allowlists, hardcoded or configurable** — rejected per *Why no action
  classification*; a configurable list would additionally let a delegated party
  vote to expand its own authority.
- **Attribution fields on `Vote`** (`castBy`, `castByRole`, `bindingCid`) —
  dropped to keep the change small, leaving attribution to the transaction
  history. See open question 3.
- **DSO as observer so Scan ingests the delegation** — done in an earlier
  revision; maintainers asked for its removal, as the delegated party is already
  visible as a controller on the `DsoRules` exercise.
- **Delegating vote closing, confirmation, or execution** — rejected: automation
  drives the close, and confirmation and execution are operator responsibilities.

### Relationship to in-flight upstream work

[splice#6247][pr6247] ("Dso Governance POC", Digital Asset) introduces
`SvRightOwner` with a `rightOwnerParty`, separate vote and reward weights, and
weight-based tallying. It decides **which party holds an SV's vote**; this CIP
decides **how the party holding the vote can let another party exercise it**, so
the two compose: applied to #6247, the delegating signatory becomes the rights
owner rather than the operator, and the template and relay choices are otherwise
unchanged.

One concrete conflict needs settling with the maintainers. #6247 appends
`rightOwnerCid : Optional (ContractId SvRightOwner)` to the same two choices, in
the same trailing position where this CIP appends `voterParty : Optional Party`.
Both are optional and additive so they can coexist, but whichever lands first
fixes the order, and if both land the interaction between #6247's
`getAndValidateVotingParty` and this CIP's co-controller should be specified
rather than left implicit. See open question 1.

[splice#5567][pr5567] (SV weight management on-ledger) touches reward weights
and `DsoRules.svs` membership, neither of which this CIP changes; no interaction
is expected beyond both editing `DsoRules.daml`.

## Trust model

An SV node operator controls the SV party, so it can already choose who casts
the SV's governance votes. A delegation records that choice on the ledger and
makes each delegated exercise co-authorized, where before the substitution was
invisible. The operator keeps the ability to archive a delegation or vote
directly at any time, so a delegation records an intention rather than binding
the party that can revoke it.

The ledger enforces that a delegation carries only its signatory SV's authority,
so it cannot be used to vote for another SV; that a delegated exercise is
co-authorized by both parties; that the delegated party gains authority over
exactly the two relay choices and cannot confirm actions, execute confirmed
actions, or exercise any other `DsoRules` choice; that a delegation cannot be
used against another DSO's `DsoRules`, via the `ForDso` check; and that
revocation takes effect immediately, a later exercise failing with
`CONTRACT_NOT_FOUND`. Because both paths execute the same choice bodies, the
vote window and per-SV cooldown apply identically to direct and delegated
exercises.

Three things the ledger does not enforce, which SV operators and SV dApp
implementers are therefore responsible for:

- **Restricting which actions a delegated party votes on.** The ledger applies
  one authority path to every action, so any narrower policy is a client or
  operational concern. A client should show the delegated party the full surface
  it can reach; `SRARC_OffboardSv` and both `SetConfig` actions deserve
  particular care.
- **Reading attribution from the transaction record.** `voterParty` appears as a
  choice argument and co-authorizer on the `DsoRules` exercise, and the update
  history is where an auditor finds it. Contract state names the delegating SV as
  the voter. Archival is likewise forward-looking: it stops later exercises and
  leaves votes already cast in place, so an SV wanting a cast vote changed casts
  again.
- **Selecting among delegations, and evidencing them.** A delegated party may
  hold delegations from two SVs, and an SV may have two live delegations; both
  are valid ledger states and the client decides how to present them. Since the
  DSO is not a stakeholder, a third party cannot enumerate delegations from the
  ledger and must obtain them from the SV or from the exercises made under them.

## Backwards compatibility

The change is additive and behavior-preserving when unused; an SV that creates
no delegation is unaffected in every respect. `Vote`, `VoteRequest`, and
`DsoRules_CloseVoteRequest` are unchanged, including result types, and no
existing call is rejected — there is no eligibility check that could break a
caller.

`voterParty = None` reproduces the previous controller set and behavior exactly.
Note this is ledger-compatible but not source-compatible for Daml callers:
record-syntax call sites must add the field, and the reference implementation
updates the in-repo callers (`SvApp.scala`, `CopyVotesTrigger.scala`, and the
`DsoTestUtils.daml` helpers) to pass `None`. The new `DsoRules_Fetch` choice is
read-only and confers no write authority; the content it exposes is already
served publicly by Scan's `/v0/dso` endpoint. `splice-dso-governance` moves from
0.1.28 to 0.1.29, all changes being additive under SCU.

**Deployment prerequisite.** Because both choices execute on a DSO-signatory
contract confirmed by every SV participant, the delegated path is not usable on
a multi-SV network until `splice-dso-governance` 0.1.29 is vetted on all SV
participants and `AmuletConfig.packageConfig.dsoGovernance` is raised to it.
That package-version change is itself a governance vote, so adopting this CIP
requires one non-delegated vote before the first delegated vote is possible.
Single-SV and development networks have no such constraint.

## Reference implementation

Two stacked pull requests on the
[canton-network/splice-sv-voting-dapp][fork] feature fork of Splice, both
unmerged as of 2026-08-20:

- [splice-sv-voting-dapp#12][pr12] — the Daml change: `VoteDelegation`,
  `DsoRules_Fetch`, the optional `voterParty` arguments, and Daml Script tests.
  Approved by a Splice maintainer on 2026-08-05, awaiting merge.
- [splice-sv-voting-dapp#19][pr19] — reference client: the SV frontend in a mode
  that logs in with a CIP-0103 wallet, reads from Scan, and submits through the
  delegation, plus a cross-participant integration test that drives a delegated
  cast through the UI and asserts the ballot is recorded against the SV.

Not yet covered: no conforming third-party wallet is exercised in CI, which
would require the wallet to offer a headless approval mode, and the delegated
*request* path and the negative cases are covered by Daml Script rather than
across participants.

[fork]: https://github.com/canton-network/splice-sv-voting-dapp
[pr12]: https://github.com/canton-network/splice-sv-voting-dapp/pull/12
[pr19]: https://github.com/canton-network/splice-sv-voting-dapp/pull/19
[pr210]: https://github.com/canton-foundation/cips/pull/210
[pr6247]: https://github.com/canton-network/splice/pull/6247
[pr5567]: https://github.com/canton-network/splice/pull/5567

## Open questions

1. **Argument ordering against [splice#6247][pr6247].** Should `voterParty` or
   `rightOwnerCid` land first on `DsoRules_RequestVote` / `DsoRules_CastVote`,
   and does `SvRightOwner` make the delegation redundant in Digital Asset's
   target state?
2. **Delegation uniqueness.** Should "at most one live delegation per SV" and
   "at most one delegating SV per delegated party" be ledger invariants — via a
   contract key on `sv`, a registry, or a duplicate-create guard — or remain
   client conventions? The landed design has no guard, and the reference client
   raises an error on zero or several matches.
3. **Attribution.** Is transaction-record-only attribution acceptable, or should
   `Vote` carry the casting party? Recording the caster is a narrower change
   than making the delegation network-visible, which is what review objected
   to, and it bears on any audit view over governance history.

## Changelog

- **2026-08-21:** Cut the draft to roughly a third of its length against the
  level of detail in comparable Daml-layer CIPs, per review. Removed the test
  matrix, the enumeration of votable action constructors, and the per-file
  artifact list; merged the overlapping Trust model and Security considerations
  sections; reduced the reference-implementation section to the two PRs and the
  open coverage gaps; and moved process questions to the pull request. Sections
  now follow the CIP-0000 order.
- **2026-08-20:** Corrected the approval state of the Daml PR. It was described
  as awaiting maintainer approval; it was in fact approved on 2026-08-05 and is
  mergeable, awaiting merge only. Both reference-implementation PRs remain
  unmerged.
- **2026-08-17:** Corrected the reference-implementation section: the
  implementation is two stacked PRs, not three, and the test inventory and
  coverage gaps were rewritten against what the branch contains.
- **2026-08-05:** Initial draft. Replaces the withdrawn
  `cip-XXXX-SV-Governance-Voter` draft ([cips#210][pr210], closed 2026-08-04),
  which specified a DSO-signed `SvGovernanceVoter` binding, a hardcoded action
  allowlist, confirmation-quorum rotation, `Vote` attribution fields, close-time
  staleness filtering, and binding garbage collection. That design was narrowed
  out during Splice review, and its reference implementation
  ([canton-network/splice#5533](https://github.com/canton-network/splice/pull/5533))
  was closed unmerged on 2026-06-10. This draft specifies only what the current
  reference implementation contains.
