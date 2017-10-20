# Bitswap Spec Sync Up - October 20

**Participants:**

- @diasdavid
- @dgrisham

David and David (:)) had a quick chat on October 20 with the purpose of identifying a roadmap for bitswap spec and research. This follows the thread on https://github.com/ipfs/js-ipfs-bitswap/issues/21#issuecomment-326826393

## Notes

We want to reach a point where multiple researchers can cooperate and/or compete to find better bitswap strategies that take into account different properties of the network such as: bandwidth available, disk space, location, load and more. We can safely assume that there won't be a one true bitswap strategy for every use-case and runtimes (browsers, IoT, datacenters, etc).

### Bitswap Contest

We want to **run a Bitswap Contest 2018** to find new and optimal strategies, similar to how [tit-for-tat](https://en.wikipedia.org/wiki/Tit_for_tat) was discovered. To make this contest happen, we want:
- Bitswap spec written and complete
- Have a testnet in place
- Have run through experiments ourselves

We see this contest as being and excellent driver to develop Bitswap more and things like the Interplanetary TestLab

### The Bitswap spec https://github.com/ipfs/specs/tree/master/bitswap

David Grisham is willing to take the lead on writting the bitswap spec given that input from David and Jeromy is provided on how it is implemented in both languages. With the current reduced availability and timeline, it looks like we should have a final version of the spec by the January.

Meanwhile, David will work on documenting his work with go-ipfs-bitswap and document the modifications he has done the codebase. The goal is so that we identify the primitives that Bitswap needs to expose in order to test different strategies.

We are considering rescheduling the Bitswap research call to make it more friendly for UTC+0~3 habitants

## Next steps

- [ ] Continue testing go-ipfs Bitswap strategy implementation
- [ ] Once Bitswap strategy impl is solid, add features to kubernetes-ipfs that are useful for Bitswap testing + metric gathering/aggregation
- [ ] Start delving into Bitswap modules for Bitswap spec background, ask @diasdavid and @whyrusleeping questions
