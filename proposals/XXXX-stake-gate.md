---
simd: 'XXXX'
title: Minimum participation threshold for gossip, turbine, and repair
authors:
  - Alex Pyattaev (Anza)
category: Standard
type: Core
status: Draft
created: 2026-05-19
feature: (to be assigned)
supersedes:
superseded-by:
extends:
---

## Summary

Under Alpenglow ([SIMD-0326]), the **active set** is strictly capped at
the top 2000 by stake and is the only set that participates in consensus
and leader rotation. Other nodes (RPC
providers, indexers, archivers, backup validators) are by construction
outside the active set, yet they still need to receive shreds,
maintain gossip records, and obtain repairs. This SIMD defines a
**stake-based admission path** for such nodes, parallel to the
active-set admission already defined by Alpenglow.

A node identity is a **participant** for an epoch if it is in the
**active set** ([SIMD-0326]) or its total activated stake meets
`PARTICIPATION_THRESHOLD` (proposed at 10 SOL); see Terminology for
the precise definitions.

A node MUST NOT admit non-participant identities into its CRDS, nor
exchange gossip protocol messages with them. Because CRDS feeds
turbine peer selection and repair routing, this single gate also
restricts turbine retransmission and repair to participants.

Non-participants may obtain shreds and gossip state via
out-of-protocol mechanisms (third-party shred relays, peering arrangements,
public mirrors, etc.); they are simply not visible to, nor served by, the
in-protocol logic.

[SIMD-0326]: https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0326-alpenglow.md
[SIMD-0357]: https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0357-alpenglow_validator_admission_ticket.md

## Motivation

Today the turbine tree, the repair subsystem, and gossip's active push set all
admit unstaked node identities. This creates several problems, the most acute being:

1. **Unstable ordering of the unstaked nodes.** Turbine tree construction
   in `ClusterNodes` performs a stake-weighted shuffle; the unstaked tail
   is ordered by pubkey, but the *set* of unstaked nodes each validator has
   learned about via gossip may differ across the cluster, so two parents
   may place the same unstaked child at different positions in their local
   trees. Affected nodes receive only a partial subset of the shreds they
   should — and the network compensates with elevated repair traffic.
   Further info in [Issue 6261](https://github.com/anza-xyz/agave/issues/6261)
2. **Inflated Repair load.** Nodes that consume shreds over turbine without stake create
   disproportional repair load on the staked nodes, which increases operational
   costs. Currently an unstaked node can decode ~90% of blocks without requesting repairs.
   This means for ~10% of slots it would be requesting at least some repairs. This
   gets amplified across all unstaked nodes, and disproportionally affects high-staked
   nodes (as they are preferred during repair peer selection).
3. **Free Sybil capacity.** Adding an unstaked identity to gossip is costless,
   but imposes costs of gossip message propagation and shred delivery on all the
   other nodes.

A stake threshold resolves all three by collapsing the zero-stake
shuffle tail (with `Pubkey` comparison as a globally consistent
tiebreak among equal-stake nodes), pricing Sybil entry at the threshold
per identity, and confining propagation to nodes with capital committed
to the cluster.

## Dependencies

- **[SIMD-0326] — Alpenglow.** This SIMD presupposes the Alpenglow
  active-set model and MUST NOT activate before Alpenglow is active on
  the target cluster (see Alternatives Considered #4 for the reasoning).
  The active set is treated as a primitive defined by SIMD-0326; this
  SIMD does not re-specify it.
- **[SIMD-0357] — Alpenglow Validator Admission Ticket.** Defines the
  per-epoch VAT mechanism by which the active set is materialized at the
  epoch boundary. This SIMD reads the active set as the post-VAT bank
  state.

The participant set is derived from the same epoch stake snapshot used
for leader schedule computation and introduces no further dependencies.

## New Terminology

- **`PARTICIPATION_THRESHOLD`**: a protocol constant for the stake-based
  admission path, proposed at `10 * LAMPORTS_PER_SOL = 10_000_000_000`
  lamports. This value is a starting point and is expected to be refined
  during discussion; see Alternatives Considered.
- **Active set** (of an epoch `E`): the set of validators admitted to
  Alpenglow consensus for `E` — materially, the top-2000-by-stake
  voting validators whose vote accounts had VAT deducted at the epoch
  boundary per [SIMD-0357]. Defined by [SIMD-0326]; this SIMD does not
  re-specify it.
- **Participant** (for an epoch `E`): a node identity (`Pubkey`) that
  belongs to the active set for `E`, or whose total activated stake —
  summed across every vote account whose `node_pubkey` equals that
  identity — at the epoch boundary is greater than or equal to
  `PARTICIPATION_THRESHOLD`.
- **Participant set** (of an epoch `E`): the set of all participants
  for `E`.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 and RFC 8174.

### Participant set computation

At every epoch boundary, each validator MUST compute the participant set
for the upcoming epoch as the union of:

1. **The active set**, computed by Alpenglow at the epoch boundary per
   [SIMD-0326] and [SIMD-0357]. All conforming clients observe it as
   part of the post-VAT bank state.

2. **Identities meeting the stake threshold**, derived from the
   activated-stake snapshot used to build the leader schedule of the
   upcoming epoch. For each `node_pubkey` `N`, `N` is included iff
   `stake(N) >= PARTICIPATION_THRESHOLD`, where

   ```
   stake(N) = sum(activated_delegation(v)) for every vote account v
              where v.node_pubkey == N
   ```

The participant set MUST remain constant within an epoch.

### Gossip

Gossip is the root gate through which all subsequent restrictions cascade.
A participant:

- MUST NOT accept into its CRDS any record whose publishing identity is not in
  the participant set for the current epoch. Existing records MUST be dropped
  from CRDS once publishing identity loses participant status.
- MUST drop any gossip protocol message whose source identity is not in
  the participant set.
- MUST upon feature activation prune all entries from non-participant nodes
  from its CRDS, and recalculate their push sets to target only participants.

A non-participant identity therefore cannot become known to participant
peers through gossip, and the gossip membership of the cluster collapses to
the participant set.

### Turbine

Because turbine tree construction reads peer TVU addresses from CRDS,
and CRDS no longer admits non-participant identities, the turbine tree is
automatically restricted to the participant set. Upon feature activation,
nodes MUST remove all non-participant peers from their turbine tree
calculations. Unlike typical turbine changes, this takes effect without
the extra epoch delay typical for turbine (since this only affects
the unstaked nodes, and thus can not break block propagation).

Out-of-protocol shred propagation channels are out of scope.

### Repair

Because in-cluster repair peer selection reads from CRDS, repair request
routing is automatically restricted to the participant set. In addition:

- A participant MUST drop repair requests whose source identity is not
  in the participant set.

Out-of-protocol repair channels are out of scope.


### Edge Cases

- **Boundary crossings.** A node whose stake crosses the threshold (in
  either direction) between two epoch snapshots changes status only at the
  next epoch boundary; mid-epoch status is fixed. The same applies to
  active-set membership — VAT payment and active-set selection within an
  epoch affect the *next* epoch's participant set, not the current one.
- **Active-set member with sub-threshold stake.** It is in principle
  possible (on small experimental clusters, devnets, or during stake
  migrations) for an active-set validator's delegated stake to fall below
  `PARTICIPATION_THRESHOLD`. Such an identity remains a participant by
  virtue of active-set membership, without needing to additionally clear
  `PARTICIPATION_THRESHOLD` — a paying voter cannot be expelled from
  gossip purely because of the stake amount.
- **Alpenglow eviction by zero rewards.** A node that earns 0 SOL in
  rewards in epoch `E` is removed from the active set for epoch `E+1`
  per [SIMD-0326]. If its delegated stake still satisfies
  `PARTICIPATION_THRESHOLD`, it remains a participant via the
  stake-based path; otherwise it drops out of the participant set at
  that boundary. This is the Alpenglow analogue of TowerBFT delinquency,
  and the stake-based path is what allows a former voter to remain
  visible to gossip as long as their stake holds.
- **Brand-new operators.** A new operator is not a participant until
  the first epoch boundary at which either their identity enters
  active set or meets `PARTICIPATION_THRESHOLD`.
  Node synchronization before that would require on out-of-protocol
  channels, and is not supported.

### Validator Components Affected

| Validator Component        | Impact                                       |
|----------------------------|----------------------------------------------|
| Transaction Execution      | None                                         |
| Virtual Machine            | None                                         |
| Block Packing              | None                                         |
| Consensus                  | None                                         |
| Gossip                     | MUST drop traffic from non-participants      |
| Turbine                    | Membership restricted to the participant set |
| Snapshots                  | None                                         |
| On-Chain Core BPF Programs | None                                         |
| Repair                     | Requests from non-participants dropped       |

## Alternatives Considered

1. **Status quo.** Retains the ordering instability and other problems described
   in Motivation.
2. **Fix the shuffle determinism for unstaked nodes.** Specifying
   a stable tiebreak for the unstaked tail addresses the coverage gap, but
   leaves Sybil costs at zero and does not reduce the turbine load attributable
   to passive unstaked receivers. Examined further in
   https://github.com/anza-xyz/agave/issues/6261, deemed too complex and unreliable.
3. **A different threshold.** 10 SOL is a round number selected for
   discussion. The "right" value depends on the trade-off between Sybil
   resistance (favoring higher) and openness to small operators (favoring
   lower). The mechanism in this SIMD is independent of the specific
   value.
4. **Activate before Alpenglow.** Would impose the threshold under
   TowerBFT, where there is no top-2000 cap on voters and no
   zero-reward eviction. Rejected because under TowerBFT a non-voting
   operator can self-delegate and remain in `staked_nodes` indefinitely,
   so the stake-based approach could create skips, and all staked nodes,
   no matter how little stake they hold, are expected to vote.
5. **Stake-only admission, without the active-set carve-out.** Would
   apply `PARTICIPATION_THRESHOLD` to every identity unconditionally,
   including active-set validators. Rejected because it would impose a
   second independent admission gate on a population already admitted
   by Alpenglow consensus, and could in pathological cases (small
   clusters, devnets) expel a paying voting validator from gossip
   solely based on the stake threshold — an outcome that would be
   surprising.

## Impact

- **Validators that already meet the threshold.** No operational change.
- **Non-voting node operators (RPC providers, indexers, archivers).**
  Must hold ≥ `PARTICIPATION_THRESHOLD` of delegated stake. The
  active-set path under Alpenglow is not available — non-voting
  active-set members are evicted per [SIMD-0326].
- **Frictionless ingress.** Lost. Operators previously running unstaked
  nodes must delegate stake or migrate to out-of-protocol shred sources
  during the transition.
- **Gossip spy tool.** Will stop working, unless a staked ID is used.
- **Cluster performance.** Expected to improve: the participant-set tree
  is smaller and globally consistent, equal-stake tiebreaks are
  well-defined, and repair load attributable to incompletely-covered
  unstaked receivers is removed.
- **Bandwidth usage.** The cluster currently hosts ~5000 nodes, of which
  only ~800 are staked. The participation requirement is expected to
  roughly halve the unstaked population; gossip and turbine bandwidth per
  validator scales with cluster size, so per-validator bandwidth should
  drop in proportion to the resulting cluster shrinkage.
- **End-users.** No direct impact.

## Security Considerations

- **Attack resistance.** Improved. Adding an identity to the in-protocol
  propagation fabric now costs at least `PARTICIPATION_THRESHOLD` of
  capital, not zero.
- **No new consensus surface.** The proposal does not introduce any new
  on-chain account, instruction, or runtime state. It modifies only the
  off-chain validator behavior that selects peers for shred and gossip
  propagation.

## Backwards Compatibility

MUST NOT activate before Alpenglow ([SIMD-0326]); see Dependencies and
Alternatives Considered #4.

This change affects gossip propagation and therefore requires a feature
gate.

Validators that have not upgraded to a release implementing this SIMD by
the activation epoch will continue to forward to and accept traffic from
below-threshold peers; this will result in excessive traffic in
gossip until everyone upgrades. This will not affect liveness.

## Future Work

- **Subscription accounts.** A subsequent SIMD MAY specify a per-epoch
  on-chain subscription mechanism that admits non-staking operators as
  participants in exchange for a fee distributed pro-rata to active-set
  validators, with a top-N cap (e.g., 4000 non-active-set slots, for
  ~6000 total participants alongside the 2000 active-set validators)
  determined by bid. Active-set identities would remain admitted
  automatically and would not compete with subscribers for slots.
- **Threshold revision.** The 10 SOL threshold value SHOULD be revisited once
  post-activation measurements of repair load, propagation latency, and
  operator population effects are available.
- **Snapshot fetch.** A subsequent SIMD MAY allow RPC nodes to refuse the
  service of snapshots to IP addresses which are not currently listed in
  gossip ContactInfo.
