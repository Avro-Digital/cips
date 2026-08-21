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

Splice uses the SV operator party for two different purposes. It is the identity
that runs node automation, and it is also the identity that votes on governance.
An SV organization can want a policy owner or an executive to vote for it. Today
it must give that person the credentials of the SV, and those credentials hold
full authority over the node.

This CIP lets an SV delegate its governance vote to a different party. An SV
signs a `VoteDelegation` contract that names one `voterParty`. That party
exercises two relay choices on the delegation. Each relay choice then exercises
the existing `DsoRules_RequestVote` or `DsoRules_CastVote` for the SV. Both of
those choices get one optional `voterParty` argument, and both take the
delegated party as an additional controller. Two parties therefore authorize a
delegated exercise, and the transaction records it as such. An SV revokes a
delegation when it archives the contract.

The vote stays the SV's own vote. It writes the SV's existing entry in
`VoteRequest.votes`. It adds no weight, and it creates no second voting unit.
The change is additive, and behavior does not change if an SV creates no
delegation. The delegated party can hold its key in any wallet that conforms to
[CIP-0103][cip-0103].

[cip-0103]: https://github.com/canton-foundation/cips/blob/main/cip-0103/cip-0103.md
[cip-0105]: https://github.com/canton-foundation/cips/blob/main/cip-0105/cip-0105.md
[scu]: https://docs.daml.com/upgrade/smart-contract-upgrades.html

## Copyright

This CIP is licensed under CC0-1.0:
[Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

## Specification

This CIP adds one delegation template and the smallest change to `DsoRules` that
can use it. All changes are additive and comply with
[Smart Contract Upgrades (SCU)][scu]. This CIP adds these items:

- A new `VoteDelegation` template with two relay choices.
- One optional trailing `voterParty` argument on `DsoRules_RequestVote` and on
  `DsoRules_CastVote`.
- One additional controller on each of those two choices.
- One read-only `DsoRules_Fetch` choice.

These parts of governance do not change: tallying, quorum, confirmation,
execution, close, cooldown, automation, SV membership, reward weights, and the SV
locking framework of [CIP-0105][cip-0105]. `Vote`, `VoteRequest`, and `SvInfo` do
not change. Every `DsoRules` choice other than the two named above also does not
change.

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

The SV is the only signatory. A delegation is a unilateral declaration by the
SV. It is not a bilateral agreement, and the DSO does not govern it.
`voterParty` is an observer, so the delegated party can see the delegation and
exercise it. The `dso` field supports the `ForDso` contract-group-id check on
reads. The DSO party is deliberately not a stakeholder.

A delegation can set `voterParty` to the SV itself. This case is equivalent to
the behavior today. The template has no contract key and no on-ledger uniqueness
guard. See [Open questions](#open-questions). The signatory SV revokes a
delegation when it archives the contract. There is no separate revoke choice and
no rotate choice.

### Relay choices

Both relay choices are `nonconsuming`, and `voterParty` controls both. Each
choice takes the whole argument of the corresponding `DsoRules` choice, and then
does three steps. First it checks that the argument names this delegation's SV
and its `voterParty`. Then it reads `DsoRules` as public reference data. Then it
relays the argument.

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

`VoteDelegation_RequestVote` uses the same structure. It relays a
`DsoRules_RequestVote`, and it checks that `requestVote.requester == sv`.

The signatory of the delegation supplies the SV's authority to the `DsoRules`
exercise. The choice controller supplies the authority of the delegated party. A
delegation holds no authority from any other SV, so it cannot vote for a
different SV. The ledger authorization model gives this guarantee. The two
`require` guards only produce a clearer error.

A delegation authorizes these two choices only. SV node automation closes a vote
request. Confirmation and execution stay with the operator. None of those
actions needs a delegation.

### `DsoRules` changes

`DsoRules_RequestVote` and `DsoRules_CastVote` each get one optional argument.
SCU compatibility requires that the new argument comes last. Each choice also
takes the delegated party as an additional controller.

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

An SV that exercises a choice directly passes `voterParty = None`. This gives
the controller set and the behavior that exist today, because `Optional Party`
converts to the controller list and `None` adds no controller. A delegated
exercise passes `Some voter`. The SV and the delegated party then both authorize
the transaction.

The choice bodies do not change. There is no branch on `voterParty`, and there
is no eligibility check. The choices write no attribution into `Vote`.

`DsoRules` also gets a read-only `DsoRules_Fetch` choice. Its argument party
controls it, and it returns `this`. It mirrors the existing `AmuletRules_Fetch`.
A delegation cannot authorize `fetchReferenceData (ForDso dso) dsoRulesCid`
directly, because `dso` is the only `DsoRules` stakeholder and a delegation
holds only `sv` and `voterParty`.

### Scope and limits of the delegated authority

A delegation authorizes `VoteDelegation_RequestVote` and
`VoteDelegation_CastVote`. Through them it authorizes `DsoRules_RequestVote` and
`DsoRules_CastVote`. Every other `DsoRules` choice stays with the SV party.

Within those limits, a delegation covers every `ActionRequiringConfirmation`
that the signatory SV can vote on. The ledger applies one authority path to all
of them, and it makes no distinction between governance actions and operational
actions. A client that wants a smaller set must enforce that limit itself. Note
that `SRARC_SetConfig` and `CRARC_SetConfig` together reach every field of
`DsoRulesConfig` and `AmuletConfig`. Note also that SV node automation drives
several votable actions, not a person.

`VoteRequest.votes` keeps its key of the SV display name, and `Vote.sv` remains
the signatory SV. Both paths therefore write the same entry, and
`DsoRules_CloseVoteRequest` still counts at most one vote for each SV.

### Discovery and submission

The client of a delegated party queries the active contract set to find its
delegation. It looks for `VoteDelegation` contracts that name it as
`voterParty`. It reads the SV party and the contract id from that contract
instead of from configuration. The ledger does not prevent an SV from signing
several delegations, and it does not prevent several SVs from delegating to the
same party. This query is therefore a client convention, not a ledger invariant.

The participant of the delegated party does not need to host DSO contracts. A
delegated submission therefore attaches the contracts that it reads as
explicitly disclosed contracts. Both choices need `DsoRules`, and a cast also
needs the target `VoteRequest`. A submission that omits a disclosed contract
fails with `PERMISSION_DENIED`.

`DsoRules_CastVote` archives the `VoteRequest` and then creates it again. A
client must therefore read the current contract id immediately before each cast.
A stale id fails with `CONTRACT_NOT_FOUND`. This CIP does not standardize the
read API that supplies these contracts.

## Motivation

Governance voting and node operation are different responsibilities. In many SV
organizations, different people hold them. The current model does not permit that
separation. The SV party controls `DsoRules_RequestVote` and
`DsoRules_CastVote`, and the SV party belongs to the node operator.

An organization that wants a policy owner or an executive to cast its governance
votes has two options today. It can give that person node-operator credentials,
which grant much more authority than voting needs. As an alternative, an
operator can sign for that person. The ledger then holds no evidence that the
vote reflects the decision of anyone except the operator.

External signing makes the gap more visible. [CIP-0103][cip-0103] defines how an
external party submits through a wallet with disclosed contracts. That mechanism
applies directly to governance voting. However, it has no contract to use,
because only the SV party can cast the SV's vote. Node automation holds the SV
party, and that party is not a wallet-held key.

This CIP adds the smallest on-ledger construct that closes this gap. It does not
redesign governance. It does not change how much say each SV has, and it moves
no existing authority.

## Rationale

**Why a separate template and not new `DsoRules` choices.** The delegation needs
the SV as its signatory, and the authority of the SV must reach the `DsoRules`
exercise. A guard on a contract that the SV signed satisfies both conditions
together. The authors considered dedicated choices such as
`DsoRules_CastGovernanceVote` and rejected them. Such choices duplicate the cast
logic. Each duplicate must then track the original for cooldown, deadlines, and
well-formedness.

**Why optional trailing arguments on the existing choices.** `DsoRules` must
know that two parties authorized an exercise, so that the delegated party
appears in the authorizing set. An optional trailing argument and an additional
controller achieve this. They add no branch to the choice body, and they require
no change from existing callers. They also comply with the SCU rule that new
fields are optional and come last.

**Why the SV signs and not the DSO.** A delegation is an agreement between an SV
and the party that it selects. During review of the predecessor design, Splice
maintainers concluded that the network does not need to know about that
agreement. A DSO signature would also require three more functions: a default
delegation at onboarding, a governance action to change the delegated party, and
cleanup of duplicates. None of these adds control, because the signatory SV can
already replace the delegated party at any time.

**Why no action classification.** The predecessor design divided
`ActionRequiringConfirmation` into voter-eligible actions and operator-only
actions. An SV that trusts a party to cast its votes has already expressed that
trust. A Daml-level allowlist also implies an enforcement guarantee that the
model does not give, because the operator can ignore the allowlist and vote
directly. Which actions need which authority is a real question. Digital Asset
answers it upstream in the `voteType` of [splice#6247][pr6247], and a second
classification here would compete with that one.

**CIP-0103 compatibility.** `voterParty` controls the relay choices, and
`voterParty` can be an externally-hosted party. A delegated submission can
receive every contract that it reads as a disclosed contract. This CIP
deliberately does not assert `Requires: CIP-0103`. The on-ledger change is
useful on its own, so the relationship is compatibility and not dependence.

### Alternatives considered

- **DSO-signed binding with confirmation-quorum rotation.** This was the
  withdrawn predecessor design ([cips#210][pr210]). Splice maintainers rejected
  it as too large for the problem.
- **Store the delegated party on `SvInfo`.** Rejected. It couples governance
  voting to the SV membership record, and it makes a change of delegated party
  into an edit of SV membership.
- **Action allowlists, either hardcoded or configurable.** Rejected, as *Why no
  action classification* explains. A configurable list would also let a
  delegated party vote to expand its own authority.
- **Attribution fields on `Vote`**, such as `castBy`, `castByRole`, and
  `bindingCid`. Dropped to keep the change small. Attribution now comes from the
  transaction history. See open question 3.
- **DSO as observer, so that Scan can read the delegation.** An earlier revision
  did this, and maintainers asked for its removal. The delegated party is
  already visible as a controller on the `DsoRules` exercise.
- **Delegate the close, the confirmation, or the execution of a vote.**
  Rejected. Automation closes a vote request, and confirmation and execution are
  operator responsibilities.

### Relationship to in-flight upstream work

[splice#6247][pr6247] ("Dso Governance POC", Digital Asset) adds `SvRightOwner`.
That template holds a `rightOwnerParty`, separate vote and reward weights, and
weight-based tallying. It decides which party holds the vote of an SV. This CIP
decides how the party that holds the vote can let another party exercise it. The
two changes therefore work together. Under [splice#6247][pr6247], the rights
owner becomes the signatory of the delegation instead of the operator. The
template and both relay choices do not otherwise change.

One conflict needs a decision from the maintainers. [splice#6247][pr6247]
appends `rightOwnerCid : Optional (ContractId SvRightOwner)` to the same two
choices, in the same trailing position that this CIP uses for
`voterParty : Optional Party`. Both arguments are optional and additive, so they
can coexist. However, the change that merges first sets the order. If both
changes merge, the specification must define how `getAndValidateVotingParty` in
[splice#6247][pr6247] interacts with the co-controller in this CIP. See open
question 1.

[splice#5567][pr5567] moves SV weight management on-ledger. It changes reward
weights and `DsoRules.svs` membership. This CIP changes neither. The authors
expect no interaction, except that both changes edit `DsoRules.daml`.

## Trust model

An SV node operator controls the SV party. It can therefore already choose who
casts the governance votes of the SV. A delegation records that choice on the
ledger, and it makes both parties authorize each delegated exercise. Before this
change, such a substitution was invisible. The operator keeps the ability to
archive a delegation or to vote directly at any time. A delegation records an
intention. It does not constrain the party that can revoke it.

The ledger enforces these properties:

- A delegation holds only the authority of the SV that signed it. It cannot vote
  for a different SV.
- Both the SV and the delegated party authorize a delegated exercise.
- The delegated party gets authority over the two relay choices only. It cannot
  confirm actions, execute confirmed actions, or exercise any other `DsoRules`
  choice.
- The `ForDso` check stops a delegation that acts against the `DsoRules` of a
  different DSO.
- Revocation takes effect immediately. A later delegated exercise fails with
  `CONTRACT_NOT_FOUND`.
- Both paths execute the same choice bodies. The vote window and the cooldown
  for each SV therefore apply equally to direct and delegated exercises.

The ledger does not enforce the three items below. SV operators and the
implementers of SV dApps are responsible for them.

- **Which actions a delegated party votes on.** The ledger applies one authority
  path to every action, so any smaller policy is a client concern or an
  operational concern. A client should show the delegated party every action
  that it can reach. `SRARC_OffboardSv` and both `SetConfig` actions need
  particular care.
- **Attribution.** `voterParty` appears as a choice argument and as a
  co-authorizer on the `DsoRules` exercise. An auditor finds it in the update
  history. Contract state names the signatory SV as the voter. Archival also
  applies only to later exercises. It leaves votes already cast in place, so an
  SV that wants to change a cast vote must cast again.
- **Selection between delegations, and evidence of them.** A delegated party can
  hold delegations from two SVs, and an SV can have two live delegations. Both
  are valid ledger states, and the client decides how to present them. The DSO
  is not a stakeholder, so a third party cannot list delegations from the
  ledger. It must obtain them from the SV, or from the exercises made under
  them.

## Backwards compatibility

The change is additive. Behavior does not change if an SV creates no delegation.
`Vote`, `VoteRequest`, and `DsoRules_CloseVoteRequest` do not change, and this
includes their result types. Neither choice rejects any call that it accepts
today, because this CIP adds no eligibility check.

`voterParty = None` reproduces the controller set and the behavior that exist
today. Note that this compatibility applies to the ledger and not to Daml
source. Call sites that use record syntax must add the field. The reference
implementation updates the callers in the repository to pass `None`. These are
`SvApp.scala`, `CopyVotesTrigger.scala`, and the helpers in `DsoTestUtils.daml`.

The new `DsoRules_Fetch` choice is read-only and grants no write authority. Scan
already serves the content that it exposes, at the `/v0/dso` endpoint.
`splice-dso-governance` moves from version 0.1.28 to version 0.1.29. All changes
are additive under SCU.

**Deployment prerequisite.** Both choices execute on a contract that the DSO
signs and that every SV participant confirms. The delegated path is therefore
not usable on a multi-SV network until two conditions hold. All SV participants
must vet `splice-dso-governance` 0.1.29. `AmuletConfig.packageConfig.dsoGovernance`
must then rise to that version. That change of package version is itself a
governance vote. Adoption of this CIP therefore requires one non-delegated vote
before the first delegated vote is possible. Single-SV networks and development
networks have no such constraint.

## Reference implementation

The reference implementation is two stacked pull requests on the
[canton-network/splice-sv-voting-dapp][fork] feature fork of Splice. Both are
unmerged as of 2026-08-20.

- [splice-sv-voting-dapp#12][pr12] contains the Daml change: `VoteDelegation`,
  `DsoRules_Fetch`, the optional `voterParty` arguments, and Daml Script tests.
  A Splice maintainer approved it on 2026-08-05. It awaits merge.
- [splice-sv-voting-dapp#19][pr19] contains the reference client. The SV
  frontend gets a mode that logs in with a CIP-0103 wallet, reads from Scan, and
  submits through the delegation. This pull request also adds a
  cross-participant integration test. That test drives a delegated cast through
  the UI, and then asserts that the ledger records the ballot against the SV.

Two gaps remain. CI exercises no third-party wallet that conforms to CIP-0103,
because such a test needs a wallet with a headless approval mode. Daml Script
covers the delegated request path and the negative cases, but no test covers
them across participants.

[fork]: https://github.com/canton-network/splice-sv-voting-dapp
[pr12]: https://github.com/canton-network/splice-sv-voting-dapp/pull/12
[pr19]: https://github.com/canton-network/splice-sv-voting-dapp/pull/19
[pr210]: https://github.com/canton-foundation/cips/pull/210
[pr6247]: https://github.com/canton-network/splice/pull/6247
[pr5567]: https://github.com/canton-network/splice/pull/5567

## Open questions

1. **Argument order against [splice#6247][pr6247].** Which argument comes first
   on `DsoRules_RequestVote` and `DsoRules_CastVote`, `voterParty` or
   `rightOwnerCid`? Does `SvRightOwner` make the delegation redundant in the
   target state of Digital Asset?
2. **Delegation uniqueness.** Should the ledger enforce at most one live
   delegation for each SV, and at most one source SV for each delegated party? A contract key on `sv`, a registry, or a duplicate-create guard could
   do this. The alternative is to keep both rules as client conventions. The
   current design has no guard, and the reference client raises an error when it
   finds zero matches or several matches.
3. **Attribution.** Is attribution from the transaction record alone acceptable?
   The alternative is to record the casting party on `Vote`. That change is
   narrower than making the delegation visible across the network, which is what
   review objected to. The answer affects any audit view over governance
   history.

## Changelog

- **2026-08-21:** Rewrote the prose against the ASD-STE100 rules for descriptive
  writing. Split long sentences, changed passive constructions to active ones,
  removed metaphorical uses of "surface", "slot", and "shape", and reduced the
  number of parenthetical asides.
- **2026-08-21:** Cut the draft to about half its length, to match the level of
  detail in comparable Daml-layer CIPs. Removed the test matrix, the list of
  votable action constructors, and the per-file artifact list. Merged the Trust
  model and Security considerations sections, which asserted the same properties
  twice. Moved the process questions to the pull request. The sections now
  follow the order in CIP-0000.
- **2026-08-20:** Corrected the approval state of the Daml pull request. The
  draft described it as awaiting maintainer approval. A maintainer in fact
  approved it on 2026-08-05, and it awaits merge only.
- **2026-08-17:** Corrected the reference-implementation section. The
  implementation is two stacked pull requests, not three. Rewrote the test
  inventory and the coverage gaps against the content of the branch.
- **2026-08-05:** Initial draft. Replaces the withdrawn
  `cip-XXXX-SV-Governance-Voter` draft ([cips#210][pr210], closed 2026-08-04).
  That draft specified a DSO-signed `SvGovernanceVoter` binding, a hardcoded
  action allowlist, confirmation-quorum rotation, `Vote` attribution fields,
  close-time staleness filtering, and binding garbage collection. Splice review
  narrowed all of it out, and its reference implementation
  ([canton-network/splice#5533](https://github.com/canton-network/splice/pull/5533))
  closed unmerged on 2026-06-10. This draft specifies only what the current
  reference implementation contains.
