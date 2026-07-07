# Memory index

One line per fact. Read this first, then open the relevant `memory/*.md`.

## Project findings (crimson/SeaStore evaluation)
- [crimson-replicated-no-stable-layout](memory/crimson-replicated-no-stable-layout.md) — is there a stable crimson/SeaStore layout for replicated (size=5) writes? NO — every layout crashes an OSD in 1-4 min via 2 upstream `ceph_abort` paths (unfixed in v21.3.0); memory solvable, stability not.
- [crimson-vs-bluestore-baseline](memory/crimson-vs-bluestore-baseline.md) — classic BlueStore on same HW: 105k IOPS sustained 25 min, 0 crashes vs crimson ~40k-then-crash → no performance case for crimson today.
- [seastore-single-host-scaling](memory/seastore-single-host-scaling.md) — single-host size=1: 4K IOPS scales with OSD count (single OSD ~9-10k cap) and reactors up to a ~16-32 knee; don't oversubscribe 48 cores.
- [seastore-memory-sizing](memory/seastore-memory-sizing.md) — crimson `--memory` is a PER-SHARD cap (`--memory/reactors`); size >= ~10-12 GB/reactor or it OOMs (self-abort with RAM free).

## Reference (infra, build, repo)
- [test-cluster-infra](memory/test-cluster-infra.md) — 8 OSD hosts + .68 client, excluded machines, SSH key, current live classic cluster.
- [crimson-deb-build-gotchas](memory/crimson-deb-build-gotchas.md) — build-with-container.py gotchas (WITH_CRIMSON truthiness, image network, classic vs crimson).
- [ceph-repo-and-docs](memory/ceph-repo-and-docs.md) — /a/ceph branch, the two benchmark docs, HEAD/author state, boot-fix PR #69972, force-push needed.
