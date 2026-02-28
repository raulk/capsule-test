---
number: 1135
type: issue
title: parametrize justified epoch stickiness parameter
state: closed
state_reason: completed
author: djrtwo
created_at: '2019-05-29T16:23:53Z'
updated_at: '2025-04-01T19:17:08Z'
closed_at: '2025-04-01T19:17:07Z'
labels:
  - forkchoice
milestone: 🔵 v1.0.0 candidates
url: https://github.com/ethereum/consensus-specs/issues/1135
comment_count: 7
reactions:
  '+1': 1
---

# parametrize justified epoch stickiness parameter

**@djrtwo** opened · May 29, 2019 · closed as completed · `forkchoice` · Milestone: 🔵 v1.0.0 candidates

In the fork choice document we specify a stickiness of 1 epoch for justified epochs.

> Let justified_head be the descendant of finalized_head with the highest epoch that has been justified for at least 1 epoch

This parameter of "1 epoch" should be a configurable constant -- `MIN_EPOCHS_TO_USE_IN_FORK_CHOICE` (_placeholder name_).

It could also be defined in terms of slots to get fractional epochs


**Reactions**: 👍 1

---

**@ralexstokes** · May 29, 2019

do you have use cases in mind that require the parametrization?

is this to allow clients to change it based on their subjective fault tolerance?
or simply to make it easy to change as we learn more about the "in the field" behavior of the fork choice (implying we may want higher stickiness when finding the justified head)

---

**@djrtwo** · May 30, 2019

This would not be something that a client would change locally, but rather change across deployments of the eth2 chain. This came up when I was working with some academics on a formalization of ffg+LMDGHOST. Essentially this is just a magic number in the spec. Would be clearer to name it explicitly.

If in practice, finality on a chain is sparse and it is super forkful (maybe very fast slot times and a large attacker), the chain might want to adjust this param to something other than our selected `1`

---

**@hwwhww** · Apr 20, 2020

@djrtwo It seems that this issue is gone after #1185?

---

**@djrtwo** · Apr 28, 2020

It seems that stickiness was dropped in #1185 without discussion.
It was originally addressing -- https://ethresear.ch/t/beacon-chain-casper-mini-spec/2760/20

We need to discuss if we still need the stickiness of 1 epoch in the context of the bounce attack mitigations in the spec

---

**@protolambda** · Jul 23, 2020

Going through old issues to clear stale/outdated things. What is the status here, is this critical for phase 0?

---

**@djrtwo** · Aug 19, 2020

definitely not critical. Might just drop it in there though... 


---

## Timeline

- Jun 9, 2019 — **@JustinDrake** added label `presentation`
- Aug 20, 2019 — **@JustinDrake** added label `fork-choice`
- Aug 21, 2020 — **@hwwhww** set milestone `🔵 v1.0.0 Nice-to-have`
- Apr 1, 2025 — **@GaAstDev** closed

---

## Extracted relationships

**Outbound**
- #1185 — mentioned

**Inbound**
- [PR #4231 Close outdated issues, part 1](prs/4231-close-outdated-issues-part-1.md) — cross-reference (timeline)

**Mentions**: @djrtwo

---

## References

- https://ethresear.ch/t/beacon-chain-casper-mini-spec/2760/20
