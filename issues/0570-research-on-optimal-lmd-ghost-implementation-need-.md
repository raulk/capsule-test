---
number: 570
type: issue
title: 'Research on optimal LMD-GHOST implementation: need help with network topology/expectations'
state: closed
state_reason: completed
author: protolambda
created_at: '2019-02-04T13:53:13Z'
updated_at: '2026-01-18T06:41:45Z'
closed_at: '2019-02-13T18:11:01Z'
url: https://github.com/ethereum/consensus-specs/issues/570
comment_count: 7
reactions:
  rocket: 1
---

# Research on optimal LMD-GHOST implementation: need help with network topology/expectations

**@protolambda** opened · Feb 4, 2019 · closed as completed

Since the last Eth 2.0 implementers call (nr 11), I started work on a comparison of the different LMD-GHOST implementations. This issue is NOT to discuss the implementations themselves (please submit an issue/PR to my research repo), but rather what kind of simulation(s) we need to pick the best option.

Network topology is also something other teams are struggling with, e.g. see prysmaticlabs/prysm#1479

The LMD-GHOST implementations are written in Go, with comments to guide you through.
A simulated chain with some parameters is also included, but it may need to be changed to fit the results of the discussion below better.
Repo here: https://github.com/protolambda/lmd-ghost
I'll do a write-up in the Readme of the different implementation features later.

Now, to get a good simulation going, I'd like to have some help with answering the following questions:

1. How many active validators are participating? (i.e. actively attesting)
2. How many attestations does a beacon-chain node process per second?
3. Is there any maximum number of attestations per epoch? (Other than the number of active validators)
4. What is the expected latency? And what is the worst case? Latency is defined here by the number of blocks back up the tree from the local beacon-chain node head, where simulated proposers build on top of.
5. How many slots per block do we expect? (I.e. avg. skip count)
6. What's your view on pruning?
7. Related to 6: When do we delete/store old parts of the chain? (I.e. how big would the list of block-entries be in memory?)
8. Did I miss any properties?




**Reactions**: 🚀 1

---

**@protolambda** · Feb 4, 2019

Pre-PR discussion from the sharding gitter:

> My current simulation parameters:
100,000 new blocks, 640 different active validators, 100 attestation changes per block, determining the head each block. Latency factor of 0.8, max-skip slot of 4, epoch length 64.
My implementation: 11.4 seconds
Vitalik's: 34.2 seconds
But latency affects the branching factor here, and number of validators is also important. Vitalik's may perform better in other situations
E.g. same settings, but with only 64 validators, Vitalik's is ~10.8 seconds, mine is still ~11.5 seconds
Each of the two implementations has their own forte. Mine doesn't care much about total number of attestations, but much more about the number of changes

> Changing the number of attestation changes here: mine is 47s with same settings, but 64 validators, and 1000 attestation changes per block. (yes, that's 100,000,000 attestation changes in total, minus those that end up attesting the same thing twice). While Vitalik's is 42s. Not as much of a difference, but mine has a slightly harder time handling changes.
Knowing the topology + expected throughput of attestations + expected pruning is everything for a good decision here.
Although, there's also one big difference in usage: as a side-effect, the data-structure used in my computation can be used to not just look-up the relative head for the justified block in O(1), and the best child in O(1), but for every other block as well, which could make pruning rather efficient.

> I.e. you could prune a list of N blocks in O(N). All you have to do is just check their best-target and/or slot number.


---

**@vbuterin** · Feb 4, 2019

> How many active validators are participating? (i.e. actively attesting)

In the worst case up to 4m.

> How many attestations does a beacon-chain node process per second?

In the worst case ~10k.

> Is there any maximum number of attestations per epoch? (Other than the number of active validators)

We can expect every validator to attest every epoch.

> What is the expected latency? And what is the worst case? Latency is defined here by the number of blocks back up the tree from the local beacon-chain node head, where simulated proposers build on top of.

Realistically 1-5. But we need to be able to handle larger reorgs due to attackers as well, so probably have a distribution, and include some blocks going much further back. Also include attesters that just go offline some of the time.

> How many slots per block do we expect? (I.e. avg. skip count)

1-1.5?

> What's your view on pruning?

I think pruning past ~1-3 months is a great idea.

> Related to 6: When do we delete/store old parts of the chain? (I.e. how big would the list of block-entries be in memory?)

See above.

> Did I miss any properties?

In general, I'd say keep worst-case performance in mind; we don't want something that works amazingly well for the average case but breaks quickly for the worst case. Also don't want something that gets efficiency from complex heuristics that are attackable.

Also, there are some efficiency considerations here that the above did not touch upon. It's not that difficult to make LMD GHOST work in _optimistic_ cases for any number of validators, because you just run a pre-processing step that groups together the impacts from all validators whose latest attestations are any specific block B. So if everyone is participating every epoch, you can compute the fork choice rule in time O(64 * log(t)). The challenge is if there is a long tail of most recent blocks, in the worst case one validator per each slot going back the last month or more. It's important to test the fork choice rule in both cases.

**Reactions**: 👍 1 · total_count 1

---

**@protolambda** · Feb 7, 2019

The above answers help a lot. But the huge amount of validators, in addition to writing a more elaborate simulation, sparked a lot of new questions regarding `BeaconState`. See #582 

Update on my LMD-GHOST implementations: I changed every algorithm to work with arbitrary attestation weighting, and batching (aggregation, without signature things, but that can be added later).

The current state of the master-branch of my repo is not working currently, because of some uncertainties in how to deal with state. I can either simplify and continue testing, if people need their lmd-ghost implementation testing quick, or wait and implement basic storage, based on insights from my new issue (#582), for my simulation. This would enable me to simulate proper epoch transitions with shuffling.


---

**@JustinDrake** · Feb 13, 2019

@protolambda Any outstanding questions?

---

**@protolambda** · Feb 13, 2019 · _(edited)_

No, I think I'm done for now with my LMD-GHOST implementation work. Implementer teams can figure out their own preferred algorithm, using my simulator + writeup of the implementations. Parameters are easily configurable now, and combined with above answers they can change these parameters based on their speed/usage requirements. I'll close this issue now.

**Reactions**: 👍 1 · total_count 1


---

## Timeline

- Feb 12, 2019 — **@hwwhww** added label `discussion`
- Feb 13, 2019 — **@protolambda** closed

---

## Extracted relationships

**Outbound**
- #582 — mentioned

**Inbound**
- [issue #1538 Block spam accounts](https://github.com/ethereum/devops/issues/1538) — cross-reference (timeline)

**Mentions**: @protolambda

---

## References

- https://github.com/protolambda/lmd-ghost
