# Memory index

One line per fact. Read this first, then open the relevant `memory/*.md`.

## Project findings (crimson/SeaStore evaluation)
- [crimson-replicated-no-stable-layout](memory/crimson-replicated-no-stable-layout.md) — is there a stable crimson/SeaStore layout for replicated (size=5) writes? NO — every layout crashes an OSD in 1-4 min via 2 upstream `ceph_abort` paths (unfixed in v21.3.0); memory solvable, stability not.
- [crimson-vs-bluestore-baseline](memory/crimson-vs-bluestore-baseline.md) — classic BlueStore on same HW: 105k IOPS sustained 25 min, 0 crashes vs crimson ~40k-then-crash → no performance case for crimson today.
- [seastore-single-host-scaling](memory/seastore-single-host-scaling.md) — single-host size=1: 4K IOPS scales with OSD count (single OSD ~9-10k cap) and reactors up to a ~16-32 knee; don't oversubscribe 48 cores.
- [seastore-memory-sizing](memory/seastore-memory-sizing.md) — crimson `--memory` is a PER-SHARD cap (`--memory/reactors`); size >= ~10-12 GB/reactor or it OOMs (self-abort with RAM free).

## Reference (infra, build, repo)
- [test-cluster-infra](memory/test-cluster-infra.md) — LIVE cluster 8f7cb5d2 (2026-09-03, 9 hosts/45 OSDs, host->osd map, pools, build-id), hosts .69/.68 lost 2026-09-10, no ssh key on .70.
- [host-safety-core-pattern](memory/host-safety-core-pattern.md) — core_pattern=core + LimitCORE=infinity + cwd=/ + 7.9 GB root: one OSD core fills root and wedges the host; point cores at ZFS first.
- [crimson-deb-build-gotchas](memory/crimson-deb-build-gotchas.md) — build-with-container.py gotchas (WITH_CRIMSON truthiness, image network, classic vs crimson).
- [ceph-repo-and-docs](memory/ceph-repo-and-docs.md) — /a/ceph branch, the two benchmark docs, HEAD/author state, boot-fix PR #69972, force-push needed.

## EC benchmark + OSD profiling
- [ec43-load-setup](memory/ec43-load-setup.md) — the EC 4+3 A/B load setup: h0..h8 images (b43/o43/i43), p3_client_fio.sh recipe, last mix4k_w70 numbers, images still mapped.
- [unwindpmp-profiling](memory/unwindpmp-profiling.md) — unwindpmp usage, one-thread-at-a-time attach, debuglink symbol recipe, build-id matrix, waitpid/die() bugs, safe-attach rules after losing .69/.68.
