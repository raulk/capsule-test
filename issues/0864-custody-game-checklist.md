---
number: 864
type: issue
title: Custody game checklist
state: closed
state_reason: completed
author: JustinDrake
created_at: '2019-03-31T05:28:05Z'
updated_at: '2025-04-01T19:17:07Z'
closed_at: '2025-04-01T19:17:07Z'
url: https://github.com/ethereum/consensus-specs/issues/864
comment_count: 4
---

# Custody game checklist

**@JustinDrake** opened · Mar 31, 2019 · closed as completed

**TODO**

- [x] 1. **Merkleisation-friendly challenge records**: Zero-out and over-write instead of `remove()` and `append()`.
- [ ] 2. **Worst-case key reveals**: Increase `MAX_CUSTODY_KEY_REVEALS` and `EPOCHS_PER_CUSTODY_PERIOD` to deal with worst-case key reveal bandwidth.
- [ ] 3. **Increase challenge bandwidth**: Increase `MAX_CUSTODY_CHUNK_CHALLENGES` and `MAX_CUSTODY_MIX_CHALLENGES`.
- [x] 4. **Fix withdrawability**: Validators need to regain withdrawability after `responder.withdrawable_epoch = FAR_FUTURE_EPOCH`.
- [x] 5. **MPC-friendly chunk bit**: Move to something like the Legendre symbol.
- [x] 6. **Force timely key reveals**: Similar to timely challenge responses.
- [x] 7. **Multiple challenges per validator**: Only the first challenge that passes the waiting period rewards the challenger.
- [x] 8. **Stagger reveal periods**: To avoid large key reveal bursts at period boundaries.
- [x] 9. **Remove `max_reveal_lateness`**: Find cleaner alternative.
- [ ] 10. **Redefine `BYTES_PER_SHARD_BLOCK`**: Replace by `BYTES_PER_SHARD_BLOCK = 2 * BYTES_PER_SHARD_BLOCK_BODY`.
- [ ] 11. **Economic review**: Particularly around sizes and structure of rewards and penalties.
- [ ] 12. **Review `BYTES_PER_CUSTODY_CHUNK`**: There are pros and cons to increasing/decreasing.
- [x] 13. **Executable spec**: Make custody game executable and add tests.
- [x] 14. **Chunk response delay**: Add `MIN_ACTIVATION_EXIT_DELAY` for chunk responses. (To avoid proposers responding to their own challenges.)

**Ideas to consider**

- [ ] 1. **G1 key reveals**: 2x smaller key reveals.
- [x] 2. **Merge RANDAO and key reveals**: Slightly reduced key reveal overhead.
- [x] 3. ~**Batched reveals**: Batch reveals in case of lateness.~
- [ ] 4. **Separate reveal per epoch**: Derive a new key per epoch from a single period reveal.
- [ ] 5. **Forced responses**: Force proposers to respond to their challenges.
- [x] 6. **Merkleise `chunk_bits`**: Consider storing a Merkle root of `chunk_bits` intead of `chunk_bits` directly.
- [ ] 7. **Multi-challenges**: All attesters assigned random chunks to a single data root challenge.
- [ ] 8. **Merge challenge records**: Merge custody chunk and custody bit challenges.
- [ ] 9. **Dynamic chunk size**: Adjust chunk size to size of crosslink data.


---

**@vbuterin** · Apr 1, 2019

>  10. Redefine BYTES_PER_SHARD_BLOCK: Replace by BYTES_PER_SHARD_BLOCK = 2 * BYTES_PER_SHARD_BLOCK_BODY.

I don't agree with this. I feel like the shard block data size is a more "fundamental" constant; it's closer to what users care about. The fact that there's a 2x overhead is more of a protocol-specific detail.

>  3. Batched reveals: Batch reveals in case of lateness.

Also don't think there's value to this. We still have to reveal each of the individual subkeys and verify them individually so there are no real savings.

Already started on some of the others (see open PRs).

---

**@hwwhww** · Jun 15, 2020

@dankrad do you mind checking the checklist again? It looks like many issues are solved in #1705 🎉 

---

**@dankrad** · Jun 17, 2020

10 and 12 in the TODO seem outdated?

---

**@JustinDrake** · Jun 17, 2020

> 10 and 12 in the TODO seem outdated?

May be worth starting a new checklist and closing this one :)


---

## Timeline

- Mar 31, 2019 — **@JustinDrake** added label `phase 1`
- Apr 22, 2019 — **@djrtwo** closed
- Apr 23, 2019 — **@JustinDrake** reopened
- May 17, 2019 — **@djrtwo** closed
- May 17, 2019 — **@hwwhww** reopened
- Apr 20, 2020 — **@hwwhww** added label `scope:custody`
- Apr 21, 2020 — **@hwwhww** added label `general:checklist`
- Apr 1, 2025 — **@GaAstDev** closed

---

## Extracted relationships

**Outbound**
- #1705 — mentioned

**Inbound**
- [issue #686 Phase 1 checklist](https://github.com/ethereum/consensus-specs/issues/686) — cross-reference (timeline)
- [PR #870 Replace with empty instead of popping finished challenges](https://github.com/ethereum/consensus-specs/pull/870) — cross-reference (timeline)
- [PR #872 Custody punishments for late reveals & deterministic staggering](https://github.com/ethereum/consensus-specs/pull/872) — cross-reference (timeline)
- [PR #934 Custody punishments & period staggering Take 2](https://github.com/ethereum/consensus-specs/pull/934) — cross-reference (timeline)
- [PR #1035 Allow multiple bit challenges, and recover withdrawability](https://github.com/ethereum/consensus-specs/pull/1035) — cross-reference (timeline)
- [PR #4231 Close outdated issues, part 1](prs/4231-close-outdated-issues-part-1.md) — cross-reference (timeline)

**Mentions**: @dankrad
