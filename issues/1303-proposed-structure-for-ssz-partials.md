---
number: 1303
type: issue
title: Proposed structure for SSZ partials
state: closed
state_reason: completed
author: vbuterin
created_at: '2019-07-19T04:59:17Z'
updated_at: '2025-04-01T19:17:08Z'
closed_at: '2025-04-01T19:17:08Z'
labels:
  - ssz
url: https://github.com/ethereum/consensus-specs/issues/1303
comment_count: 10
---

# Proposed structure for SSZ partials

**@vbuterin** opened · Jul 19, 2019 · closed as completed · `ssz`

#### Problem statement

In an SSZ partial, we have:

* A list of generalized indices representing the `leaves` you want to make a partial of (often, this is just a single one)
* A list of generalized indices (`sisters`), that can be generated as a function `get_sister_indices(leaves: Sequence[int]) -> Sequence[int]`
* A set of `hashes` for each index in `leaves` and `sisters`

How do we serialize the whole structure? Specifically, how do we serialize which indices the partial is referring to and in what order?

### Proposed solution

First, we do **not** include `leaves` and `sisters` in the partial. We definitely don't need to include `sisters` because it can simply be calculated from `leaves`. But more importantly, we do not even need to include `leaves`, because much of the time some or all of the information from `leaves` will be known. For example, if we want to make a partial representing a Merkle path to a particular block in history from a history accumulator, it would be more reasonable to represent the block height (as that might already be included alongside the proof anyway), and then calculate the generalized index for the leaf from there. If we want to make a partial representing an entire object in a tree, we can easily calculate the set of generalized indices representing all leaves in that object. Also, this fits with the existing convention for Merkle proofs where the index is separate from the proof.

Second, we store `hashes` in the following order: first, the `leaves` in _bit-alphabetical left-to-right order_, then the `sisters` in _descending numerical order_.

Here's the definition of bit-alphabetical left-to-right order:

```python
def split_by_root(ints, depth):
    t, l, r = [], [], []
    for i in ints:
        if i.bit_length() < depth:
            t.append(i)
        elif (i >> (i.bit_length() - depth)) % 2 == 1:
            r.append(i)
        else:
            l.append(i)
    return t, l, r

def alphasort(ints, depth=2):
    if len(ints) <= 1:
        return ints
    t, l, r = split_by_root(ints, depth)
    return t + alphasort(l, depth+1) + alphasort(r, depth+1)
```

Example:

```
>>> alphasort([1,2,3,4,5,6,7])
[1, 2, 4, 5, 3, 6, 7]
```

The idea is that it is a recursive "top, then left subtree, then right subtree" order (though `leaves` should never contain both "top" and "left" (or "top" and "right")). To see more clearly, here's the tree structure:

```
   1
 2   3 
4 5 6 7
```

The goal of bitwise sorting is so that if a partial includes an entire SSZ object that has multiple levels, the different levels will be provided in order. The goal of the "leaves then sisters in descending order" structure is so that a single-item SSZ partial is always identical to a Merkle proof: first the object, then the leaves in the Merkle tree from bottom-to-top order.

For example, if we want a partial of just element 5 in the above tree, the order would be [5, 4, 3], just as in a regular Merkle proof. If we want a partial of elements 5 and 6, it would be [5, 6, 7, 4]. For elements 4 and 5, it would be [4, 5, 3].


---

**@cdetrio** · Jul 20, 2019

How does this relate to previous issues on SSZ partials? It is getting a bit difficult to [follow](https://github.com/ethereum/eth2.0-specs/pull/1261) [the](https://github.com/ethereum/eth2.0-specs/pull/1186) [various](https://github.com/ethereum/eth2.0-specs/pull/1184) [issues](https://github.com/ethereum/eth2.0-specs/pull/1180) [around](https://github.com/ethereum/eth2.0-specs/issues/1160) [SSZ](https://github.com/ethereum/eth2.0-specs/issues/1128) [merklization](https://github.com/ethereum/eth2.0-specs/issues/1115) [indexing](https://github.com/ethereum/eth2.0-specs/issues/1008) [light](https://github.com/ethereum/eth2.0-specs/issues/765) [client](https://github.com/ethereum/eth2.0-specs/issues/644) stuff.

If I understand correctly that the ultimate purpose of all this is to specify a [format for multiproofs](https://github.com/ethereum/eth2.0-specs/blob/d1c96c1e0d3b97ac6b436cbaa070e4a39f6b5876/specs/light_client/merkle_proofs.md#merkle-multiproofs), then all this documentation is quite cumbersome relative to the [docs on turbo-geth multiproofs](https://github.com/ledgerwatch/turbo-geth/blob/34afe5e8ad0892b2abb40aedf1f42ea9928deb91/docs/programmers_guide/guide.md#multiproofs). Perhaps it doesn't help that a trie structure, a multi-proof format, and a serialization encoding are all framed in terms of "SSZ" (imagine describing eth1 merkle-patricia-trie proofs in terms of "RLP merkle partials"), coming together to form a far from simple construct, part of which is ostensibly frozen with the phase0 spec and another part which is, apparently, not.



**Reactions**: 👍 2 · total_count 2

---

**@vbuterin** · Jul 21, 2019

It's almost equivalent, the only difference here is that the serialization is simpler, it's a list of hashes rather than being a container. It's also backwards compatible so eg. existing Merkle branches can fit into the format already.

I generally think of this scheme as "partials"; the connection to SSZ serialization and tree hashing is definitely not 100% right, though there are some connections, eg. the way that tree hashing is designed to insist on a binary tree structure without doing weird things to special-case in bytes, is in part there to make partials easier.

You're right a lot of those old issues need to be cleaned up; some have been accepted, others deprecated.

---

**@gitcoinbot** · Aug 5, 2019

Issue Status: 1. **Open** 2. Started 3. Submitted 4. Done 

<hr>

__This issue now has a funding of 400.0 DAI (400.0 USD @ $1.0/DAI)  attached to it.__

 * If you would like to work on this issue you can 'start work' [on the Gitcoin Issue Details page](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317).
* Want to chip in? Add your own contribution [here](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317).
* Questions? Checkout <a href='https://gitcoin.co/help'>Gitcoin Help</a> or the <a href='https://gitcoin.co/slack'>Gitcoin Slack</a>
* $145,505.05 more funded OSS Work available on the [Gitcoin Issue Explorer](https://gitcoin.co/explorer)


---

**@lightclient** · Aug 8, 2019

> We do not even need to include leaves, because much of the time some or all of the information from leaves will be known.

I'm not quite convinced on this. For example, if we think of EEs as pure state transition functions (for illustrative purposes, I'm breaking up the "data blob" parameter into `witness` and `transactions`):

```rust
fn state_transition(
    beacon_state_root: Bytes32,
    pre_state_root: Bytes32, 
    witness: Vec<Bytes32>, 
    transactions: Vec<Transaction>
) -> Bytes32;
```

And we imagine that the state root of the EE is the root of this tree:

```rust
// FixedVector[u256; 4]
//
//             +- 1 -+
//            /      \
//           2       3
//          / \     / \
//         4  5    6  7
```
We'll assume there is silly transaction that simply increases the sender's balance. The EE may receive a transaction package that looks something like this:

```jsonc
[
  {
    "amount": 0,
    "sender": 2,
    // "witness": ["ABC", "DEF", "GHI"]
  },
  {
    "amount": 0,
    "sender": 3,
    // "witness": ["JKL", "MNO", "PQR"]
  },
```
Where the witnesses would have been combined into the multiproof: `["ABC", "JKL", "MNO", "DEF"]` (e.g. chunks `[5, 6, 7, 4]`).

At this point, when the first transaction is executed and needs the path to the account at address `1`, I'm not sure how it is going to locate the correct chunks from the witness as it has no visibility into what leaves other transactions will be accessing.

I can think of two possible solutions off the top of my head:

1. There is an access list which declares the paths that will touched by the transaction package; in which case the EE will be able to calculate the accessed leaf indexes. Maybe this specifies a type of partial that is a subset of the structure described above?

2. The leaf node indexes are explicitly included in the partial structure.

---

**@gitcoinbot** · Aug 11, 2019

💰 A crowdfund contribution worth 0.10000  DAI (0.1 USD @ $1.0/DAI) has been attached to this funded issue  from @.💰 

Want to chip in also? Add your own contribution [here](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317).

---

**@gitcoinbot** · Aug 14, 2019

@uivlis Hello from Gitcoin Core - are you still working on this issue? Please submit a WIP PR or comment back within the next 3 days or you will be removed from this ticket and it will be returned to an ‘Open’ status. Please let us know if you have questions!
* [x] reminder (3 days)
* [ ] escalation to mods (6 days)

Funders only: Snooze warnings for [1 day](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=1) | [3 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=3) | [5 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=5) | [10 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=10) | [100 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=100)

---

**@uivlis** · Aug 14, 2019

Yeah, I'm still working, trying to figure out so much new jargon.

---

**@gitcoinbot** · Aug 20, 2019

@uivlis Hello from Gitcoin Core - are you still working on this issue? Please submit a WIP PR or comment back within the next 3 days or you will be removed from this ticket and it will be returned to an ‘Open’ status. Please let us know if you have questions!
* [x] reminder (3 days)
* [ ] escalation to mods (6 days)

Funders only: Snooze warnings for [1 day](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=1) | [3 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=3) | [5 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=5) | [10 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=10) | [100 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=100)

---

**@protolambda** · Aug 29, 2019 · _(edited)_

As discussed during EthBerlin (with Quilt, Vitalik, GBallet) and shared in the call, I created this repository to describe an idea to use helper-data to speed up tree lookups and verification, you could call the first approach "ssz tree offsets": https://github.com/protolambda/eth-merkle-trees **_\[experimental and untested\]_**
There is a wide effort to make multi-proof merkle trees better, also in Eth1 (@AlexeyAkhunov, @karalabe, @gballet).

Storing the bottom node data in a packed form is simplified when you can forget about what exactly is a witness, and what a leaf node. For batching it also makes a lot of sense to me to forget about the difference, as items in the batch may have one witness that is a leaf in another. Merging and verifying is just a lot more complicated (to do efficiently at least) if application differences like this influence lower level encoding.

Also, lookups for reading and in-place edits are important if we like to back a partial with a multi-proof, and have the partial behave as an interface. (less memory changes and copies in ewasm if the proof does not need to be deserialized at all).

Then, an instruction-set like approach (Alexey) is interesting for complicated structures like MPT, but I don't see how well it really speeds up lookups of leaves, and lookups should be cheap (do not construct the whole tree if possible). So I believe an approach with some kind of structural helper data, like the tree-offsets, would be better:
- No memory copies / parsing to make sense of the data
- Scale linearly with the amount of bottom nodes, regardless of structural shape
- Good for lookups: just `O(depth)` operations for lookup. (and possible to shorten during compile time for static parts)
- Good for verification: implemented a multi-proof verify function that scans the data with minimal memory, and constructs the root in a non-recursive manner (no stack-frames, yay). It is experimental and untested though, as I did not have the opportunity to focus on just this.
- Good replacement for those edge-cases where you need to encode dynamic shapes, and do not want to encode full generalized indices.

For Merkle Patricia Tree optimizations, representing MPTs as binary trees, and then changing the verification to use a bigger scratchpad-stack to hash hexary nodes, could be a solution to backport these optimizations (or at least run them efficiently in ewasm). And the encoding may also just be a start for a better encoding to back MPTs in Geth, if we add support for MPT extension nodes.

The thing where I still have doubts about is where to encode these offsets: it's simple and small, just doable in a single depth-first walk that encodes `2/32` extra data for most proofs. But at the same time there are places where you can know it during compile time and not have to read it from input.

After Eth2 interop week (= ends about 2 weeks from now) I like to pick this up with whoever is working on this. I am leaving it as-is now to focus on phase0 things.

**Reactions**: 🚀 2 · total_count 2

---

**@gitcoinbot** · Oct 17, 2019

Issue Status: 1. Open 2. **Cancelled** 

<hr>

__The funding of 400.0 DAI (plus a crowdfund of 0.1 DAI worth 0.1 USD) (400.0 USD @ $1.0/DAI) attached to this issue has been **cancelled** by the bounty submitter__

 
* Questions? Checkout <a href='https://gitcoin.co/help'>Gitcoin Help</a> or the <a href='https://gitcoin.co/slack'>Gitcoin Slack</a>
* $136,138.46 more funded OSS Work available on the [Gitcoin Issue Explorer](https://gitcoin.co/explorer)


---

## Timeline

- Jul 31, 2019 — **@hwwhww** added label `SSZ`
- Apr 1, 2025 — **@GaAstDev** closed

---

## Extracted relationships

**Inbound**
- [issue #1345 5 ETH phase 0 bounties](https://github.com/ethereum/consensus-specs/issues/1345) — cross-reference (timeline)
- [PR #1261 Partials v2](https://github.com/ethereum/consensus-specs/pull/1261) — cross-reference (timeline)
- [PR #3 Implement new proof format](https://github.com/quilt/sheth/pull/3) — cross-reference (timeline)
- [issue #474 SSZ Multiproofs](https://github.com/ChainSafe/lodestar/issues/474) — cross-reference (timeline)
- [issue #555 Light Client Task Force #2 - Multiproof Formats & Proof Negotiation](https://github.com/ChainSafe/lodestar/issues/555) — cross-reference (timeline)
- [PR #4231 Close outdated issues, part 1](prs/4231-close-outdated-issues-part-1.md) — cross-reference (timeline)

**Mentions**: @uivlis, @AlexeyAkhunov, @karalabe, @gballet

---

## References

- [format for multiproofs](https://github.com/ethereum/eth2.0-specs/blob/d1c96c1e0d3b97ac6b436cbaa070e4a39f6b5876/specs/light_client/merkle_proofs.md#merkle-multiproofs)
- [docs on turbo-geth multiproofs](https://github.com/ledgerwatch/turbo-geth/blob/34afe5e8ad0892b2abb40aedf1f42ea9928deb91/docs/programmers_guide/guide.md#multiproofs)
- [on the Gitcoin Issue Details page](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317)
- [Gitcoin Issue Explorer](https://gitcoin.co/explorer)
- [1 day](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=1)
- [3 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=3)
- [5 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=5)
- [10 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=10)
- [100 days](https://gitcoin.co/issue/ethereum/eth2.0-specs/1303/3317?snooze=100)
- https://gitcoin.co/help'
- https://gitcoin.co/slack'
- https://github.com/protolambda/eth-merkle-trees
