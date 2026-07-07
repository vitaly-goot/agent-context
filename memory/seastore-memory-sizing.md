---
name: seastore-memory-sizing
type: project
---
**Crimson `--memory` is a PER-SHARD cap, not a total-budget cap.** Seastar splits
it equally per reactor: real limit = `--memory / reactors` per shard. A shard
that exhausts its slice self-aborts the whole OSD with std::bad_alloc while the
box has 100s of GB free (Seastar per-shard allocator — NOT kernel oom-killer, NOT
systemd). Footprint ~5 GB/shard steady (~2.15 of it the SeaStore LRU = upstream
seastore_cache_lru_size 2G/reactor default). Hot shards ~2x avg, so budget the
worst case: **>= ~10-12 GB per reactor**. Old "3 GB/reactor / 27 GB OSD" is below
the floor and OOMs. Memory is a solved provisioning problem — NOT the rollout
blocker (see crimson-replicated-no-stable-layout for the real blocker).
