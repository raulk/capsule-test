---
number: 433
type: issue
title: What CBCifying the beacon chain might look like
state: closed
state_reason: completed
author: vbuterin
created_at: '2019-01-11T22:34:52Z'
updated_at: '2025-01-24T07:02:14Z'
closed_at: '2019-02-22T19:21:26Z'
url: https://github.com/ethereum/consensus-specs/issues/433
comment_count: 12
reactions:
  '+1': 4
---

# What CBCifying the beacon chain might look like

**@vbuterin** opened · Jan 11, 2019 · closed as completed

This issue is intended as an illustration of the concrete spec changes that would be required to transition the beacon chain from its current FFG-based spec (see [the mini-spec](https://ethresear.ch/t/beacon-chain-casper-mini-spec/2760)) to CBC (reading material [here](https://vitalik.ca/general/2018/12/05/cbc_casper.html) [here](https://medium.com/@aditya.asgaonkar/casper-cbc-simplified-2370922f9aa6) [here](https://ethresear.ch/t/beacon-chain-friendly-cbc-casper/4710) and [here](https://ethresear.ch/t/bitwise-lmd-ghost/4749)).

I will start off by describing a version without any validator set changes, and then introduce validator set changes below.

### LMD GHOST enforcement

The fork choice rule changes from the current "favor the highest justified checkpoint then use LMD GHOST" to "just use LMD GHOST". But for CBC purposes, we need to have LMD GHOST not just be a suggestion, but rather an enforced rule; that is, _a block is not valid unless its parent is the result of executing the LMD GHOST fork choice rule given all the evidence the block knows about_. We need to keep track of some extra data to make it possible to verify this.

In the state, we make the following changes:

* Replace the `validator_balances` list with a `validator_volatile_data` list consisting of an object `ValidatorVolatileData = {balance: uint64, last_virtual_agreement_height: uint64, last_virtual_at_height: uint64}`.
* Extend `latest_block_roots` to last one year (ie. ~2^22 entries).
* Add a list `off_chain_block_hashes = List[Tuple[Hash32, uint64]]`.

We add a new method, `submit_offchain_block_header(BlockHeader)` where `BlockHeader` is equivalent to a block except replacing all non-top-level items with their hashes. It checks that the parent is already included in either `latest_block_roots` or `off_chain_block_hashes`, and that the hash of the header itself is not in either. It then adds to the list a tuple containing the hash of the header and the "virtual agreement height": if the parent is in `off_chain_block_hashes`, then the virtual agreement height is the same as for the parent, but if the parent is the value in `latest_block_roots` corresponding to slot N, then the virtual agreement height equals N * 256 + the number of most-significant-bits by which the hash of the off-chain block and the hash of the value in `latest_block_roots` corresponding to slot N+1 agree (eg. if the hashes start with 0x35 and 0x27 then they agree on three bits; if they start with 0x4c8 and 0x4cf then they agree on nine bits).

An attestation can only be included if the block it points to is in either `latest_block_roots` or `off_chain_block_hashes`. When this is done, we set the validator's `last_virtual_at_height` to equal 256 times the slot number of the message the attestation is signing, and `last_virtual_agreement_height` to equal the `last_virtual_at_height` if the attestation is of a block that is part of the chain, and the virtual agreement heigh of the off-chain block header if it is of an off-chain block.

`off_chain_block_hashes` are removed if their slot is < current slot - 2**22 (ie. after ~1 year).

For a block to be valid, we run the following check:

* Sort the `last_virtual_agreement_height` values of all validators and the `last_virtual_at_height` values of all validators
* Verify that there exists no height H such that the total deposit size of validators whose `last_virtual_agreement_height` is > H is less than half (the total deposit size of validators whose `last_virtual_agreement_height` is >= H minus the total deposit size of validators whose `last_virtual_at_height` == H) (see [here](https://ethresear.ch/t/bitwise-lmd-ghost/4749) for an efficient algorithm for doing this)

### LMD GHOST enforcement, option 2

We store in the state the same data structures as above, plus an additional three arrays: `balance_agreeing_upto: List[List[int]]` and `balance_at: List[List[int]]`, initialized as `[[] for _ in range(LMD_GHOST_LOOKBACK)]`. We define the helpers:

```python
def two_d_array_get(array, x, y):
    return array[x][y] if y < len(array[x]) else 0

def two_d_array_set(array, x, y, value):
    if value != 0 and y >= len(array[x]):
        array[x] += [0] * (y - len(array[x]) + 1)
    if y < len(array[x]):
        array[x] = y
    while len(array[x]) > 0 and array[x][-1] == 0:
        array[x].pop()
``` 

When we process a set of attestations, we generate a list `List[Tuple[int, int, int, int, int]]` where each tuple is a `(validator index, previous_virtual_agreement_height, new_virtual_agreement_height, previous_virtual_at_height, new_virtual_at_height)`, with the variables having the same meanings as before. We then transform that list into a set of lists: `[(agreement_height, delta_balance), ...]` and `[(at_height, delta_balance)]`. For each of these items, we let `slot = (agreement_height // 256) % LMD_GHOST_LOOKBACK` and `bits = agreement_height % 256`, and run `two_d_array_set(balance_agreeing_upto, slot, bits, two_d_array_get(balance_agreeing_upto, slot, bits) + delta_balance)`. We do the same with `at_height` values.

Note: everything so far is O(1) per attestation with no cryptography, through making `L` updates to the Merkle tree will require recomputing `L * log(LMD_GHOST_LOOKBACK / L)` hashes. To handle updating balances, we do a simple trick for optimization: we store in the validator record a whole-integer value that represents their balance for LMD GHOST purposes, and only adjust it downwards to the next lower integer when one's balance drops below `xx.0` and adjust it upwards to the next higher integer when one's balance rises above `xx.5`. Only when doing this adjustment do we also perform an adjustment to `agreement_height` and `at_height` to reflect that validator's new balance.

For a block to be valid, we run the same check as above, except we directly use right-facing partial sums of a flattened version of `agreement_height` rotated by `block.slot % LMD_GHOST_LOOKBACK`. For example, suppose `LMD_GHOST_LOOKBACK = 4` and `block.slot % LMD_GHOST_LOOKBACK = 1` and `agreement_height = [[10, 20], [30, 80, 50], [], [40]]`.

Then, `agreement_height` rotated would be `[[30, 80, 50], [], [40], [10, 20]`, the flattened version is `[30, 80, 50, 40, 10, 20]` and the partial sums are `[230, 200, 120, 70, 30, 20]` - which may be invalid unless at least there are >= 10 votes `at` the same position as the 40.

Note that as a matter of efficient implementation, the sums need not be recalculated each time; nodes can maintain a sum tree locally (see description of sum trees in https://ethresear.ch/t/bitwise-lmd-ghost/4749) and use the binary search algorithm to verify correctness. One can compute a rotated sumtree from an unrotated sumtree in real time as follows: for a rotation by `r` of a list of length `L`, `rotated_sum[i] = sum[i] - sum[r] if i <= r else sum[i] + sum[0] -sum[r]`.

### Slashing condition enforcement

We only need two slashing conditions, for these two cases:

* A validator signs two different blocks in the same epoch
* A validator signs a block with a slot number of C, in which their `last_virtual_at_height` is A, and they also sign a block with a slot number B, with A < B < C

For the first slashing condition, we can reuse code from the FFG beacon chain as is. For the second, we need to create a SlashingProof object which contains the parameters:

```python
{
    index: uint64,
    data1: SlashableVoteData,
    data2: SlashableVoteData,
    merkle_branch_bottom: ValidatorVolatileData,
    merkle_branch_in_volatile_data_tree: [hash32],
    block_header_1: BlockHeader,
    block_header_2: BlockHeader,
}
```

See [here](https://github.com/ethereum/eth2.0-specs/blob/master/specs/core/0_beacon-chain.md#slashablevotedata) for the definition of `SlashableVoteData`. Verifying this would entail verifying:

* Both `SlashableVoteData` objects pass a `verify_slashable_vote_data` check
* `index` is part of `intersection(union(data1.custody_bit_0_indices, data1.custody_bit_1_indices), union(data2.custody_bit_0_indices, data2.custody_bit_1_indices))`
* The root calculated by starting from `merkle_branch_bottom` and applying `merkle_branch_in_volatile_data_tree` using `index` as an index is the `validator_volatile_data` root in the `block_header_1`
* `hash(block_header_1) == data1.data` and `hash(block_header_2) == data2.data`
* `block_header_1.slot // EPOCH_LENGTH > block_header_2.slot // EPOCH_LENGTH > merkle_branch_bottom.last_virtual_at_height // (EPOCH_LENGTH * 256)`

Note that this is almost but not quite sufficient. The reason is that an attacker could make a signature on the main chain at height H1, then sign on a fake off-chain block _that only has a header_ at height H2, include that signature in the main chain and sign at height H3, then keep signing on the main chain, and then sign a message on another chain at a height between H2 and H3. The fact that the Merkle branch for height H2 is absent means that there is no way to catch and penalize the validator. We can solve this in two ways:

1. Add an additional challenge-response game where anyone can require any validator to publish the Merkle root for their own index for any signature that they participated in
2. Require clients to actually verify the off-chain blocks before they accept any chain that references them, and make this a validity rule for the chain

(1) could be extended into a general "proof of custody of beacon chain state" mechanism, which may also be useful for other reasons.

### Dynamic validator sets

We will follow the pattern of "change the validator set by a maximum of a few percent per day, so it keeps rotating regardless of finality"; this way we don't need to verify finality on-chain and we can avoid on-chain thresholds. We can do this by moving the existing "withdrawal queue" mechanism to rate-limit exits rather than withdrawals (so we get first-come-first-served prioritization for exits, rather than the current lowest-index-first-served, which is okay at present because we're allowing up to 1/64 of all validators to exit every epoch, but would be less okay if we make exiting _actually_ slow). We add an additional identical queue, `activation_queue`, that affects validator activations. We thus have both a `validator_registry_exit_count` and a `validator_registry_activation_count` in the state, and we simply reuse the `exit_count` field in the ValidatorRecord for both.

We change the fork choice rule to "execute the fork choice rule with the validator set that's active at slot N and verify that the result is the block in the chain at slot 2N-1, then execute the fork choice rule with the validator set that's active at 2N and verify that the result is the block in the chain at slot 3N-1, etc". We would change the validator set every N slots (think: 1 day) by a maximum of 1/64. As a validity condition, we can enforce the following rule.

Suppose the current block a slot in `[k*N .... (k+1)*N-1]`. Then, we check that:

* Conditional on the block in the chain at slot `k*N` being correct, the current block's parent is the winner of the LMD GHOST fork choice using the most recent validator set
* Conditional on the block in the chain at slot `(k-1)*N` being correct, the winner of the LMD GHOST fork choice either is the block in the chain at slot `k*N`, or could be modified to be by changing 1/64 of the validators' messages
* Conditional on the block in the chain at slot `(k-2)*N` being correct, the winner of the LMD GHOST fork choice either is the block in the chain at slot `(k-1)*N`, or could be modified to be by changing 2/64 of the validators' messages

And so on. This would be implemented in practice by relaxing `[vah > h] >= ([vah >= h] - [vath == h]) / 2` from above to `[vah > h] >= ([vah >= h] - [vath == h]) * (32 - i) / 64` (where `[vah > h]` means "the total deposit size of validators with virtual agreement height more than `h`", `vath` means "virtual at height", and `i` is the number of validator rotations back that we are checking.

The `i/64` slack is there to represent the possibility that even if the fork choice looks incorrect from the point of view of present validators, it could have been correct from the point of view of validators at the time; the intention is that if a block is valid under the LMD GHOST fork choice, then it passes this check, though there is the risk that a few blocks that pass this check are not valid under LMD GHOST fork choice. In any case, safety should only decrease by ~1/64 plus an additional 1/64 per validator rotation that the client has not yet seen a block finalized.


**Reactions**: 👍 4

---

**@naterush** · Jan 14, 2019 · _(edited)_

> The root calculated by starting from merkle_branch_bottom and applying merkle_branch_in_volatile_data_tree using index as an index is the validator_volatile_data root in the block_header

Should this be `block_header_1` rather just `block_header`?

---

I'm still working through the beacon chain spec (which is probably where this confusion comes from), but what exactly is `index` in the slashing condition (aka the index that the validator could fail to reveal)? Is this the index of the validator in the validator set? 

If so, the challenge response game requires the validator to reveal the `merkle_branch_in_volatile_data_tree` as well, right?

---
> Suppose the current block a slot in [k*N .... (k+1)*N-1]

Minor typo. "Suppose the current block slot is in `[k*N .... (k+1)*N-1]`"

---

> [vah > h] >= ([vah >= h] - [vath == h]) / 2

To clarify, is `vah` is the validators with "last_virtual_agreement_height" that are either `> h` or `>= h`, and `vath == h` is the validators where `last_virtual_at_height == h'.


---

**@adiasg** · Jan 14, 2019 · _(edited)_

What does `off_chain_block_hashes` correspond to? Uncle blocks?

---

**@vbuterin** · Jan 15, 2019

> Should this be block_header_1 rather just block_header?

Yep, you're right! Fixed.

> what exactly is index in the slashing condition (aka the index that the validator could fail to reveal)? Is this the index of the validator in the validator set?

Yep!

> To clarify, is vah is the validators with "last_virtual_agreement_height" that are either > h or >= h, and vath == h is the validators where `last_virtual_at_height == h'.

Yes.

> What does off_chain_block_hashes correspond to? Uncle blocks?

Yes. Though more precisely, uncles and distant cousins more generally, not just uncles with 1 degree of separation from the canonical chain like ethereum 1.0 uncles.

---

**@adiasg** · Jan 22, 2019 · _(edited)_

Listing the complexity for reference:
 - Validator set _**V**_ (order 10^6) and block depth _**h**_. 
 - [Option 1](https://ethresear.ch/t/bitwise-lmd-ghost/4749) takes **_O(V*log(h))_ per block**. A detailed explanation can be found [here](https://medium.com/@aditya.asgaonkar/bitwise-lmd-ghost-an-efficient-cbc-casper-fork-choice-rule-6db924e57d1f).

---

**@vbuterin** · Jan 23, 2019

There's two "sizes" of V that we need to worry about: there's the full validator set, which is of order 10^6, and there's the per-slot new attestation set, which is on average of order 10^4.

---

**@vbuterin** · Jan 23, 2019

Another important thing to note is that average case complexity is much much better than worst case complexity. To see why, consider that even though it's theoretically possible for every validator's previous and new latest message to be different, in practice, most of the time, the previous and new latest messages will be concentrated among the blocks produced in the last ~128 slots.

---

**@naterush** · Feb 5, 2019 · _(edited)_

I'm a bit confused by LMD GHOST enforcement, option 2 - and have a few questions. Thanks in advance for clarifications! :)

> plus an additional three arrays: 

Only two additional arrays listed. Is the third new array `agreement_height: List[List[int]]`?

> We then transform that list into a set of lists: [(agreement_height, delta_balance), ...] and [(at_height, delta_balance)]

How is the list of attestation tuples transformed into this set of lists? For example, what is `delta_balance` and `at_height`?  

> we directly use right-facing partial sums of a flattened version of `agreement_height`

I'm confused by the `agreement_height` array. What does it contain, and when is updated? I think most of my confusion comes from not understanding what `agreement_height` array is.




---

**@JustinDrake** · Feb 13, 2019

I think we agreed CBCifying the beacon chain won't happen in phase 0. It also seems unlikely to happen in phases 1 or 2. Tempted to nudge this discussion towards ethresear.ch :)

---

**@JustinDrake** · Feb 22, 2019

Closing for now :)

---

**@irajiesisavadlo** · Jan 24, 2025

# - [ ] _****_

---

**@irajiesisavadlo** · Jan 24, 2025

- [ ] []()

---

**@cheyenne-bot** · Jan 24, 2025

Ok


---

## Timeline

- Jan 27, 2019 — **@hwwhww** added label `enhancement`
- Jan 28, 2019 — **@hwwhww** added label `RFC`
- Jan 28, 2019 — **@hwwhww** removed label `enhancement`
- Feb 22, 2019 — **@JustinDrake** closed

---

## Extracted relationships

**Inbound**
- [issue #701 CBCification](https://github.com/ethereum/consensus-specs/issues/701) — cross-reference (timeline)

**Mentions**: @aditya

---

## References

- [the mini-spec](https://ethresear.ch/t/beacon-chain-casper-mini-spec/2760)
- [here](https://vitalik.ca/general/2018/12/05/cbc_casper.html)
- [here](https://medium.com/@aditya.asgaonkar/casper-cbc-simplified-2370922f9aa6)
- [here](https://ethresear.ch/t/beacon-chain-friendly-cbc-casper/4710)
- [here](https://ethresear.ch/t/bitwise-lmd-ghost/4749)
- [here](https://github.com/ethereum/eth2.0-specs/blob/master/specs/core/0_beacon-chain.md#slashablevotedata)
- [here](https://medium.com/@aditya.asgaonkar/bitwise-lmd-ghost-an-efficient-cbc-casper-fork-choice-rule-6db924e57d1f)
