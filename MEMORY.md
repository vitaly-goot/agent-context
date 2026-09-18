> **NEW AGENT: START HERE.**
>
> **Your assignment:** prototype a durable write-coalescing journal
> ("log-structured machinery") in Ceph's EC backend, to remove the partial-stripe
> read-modify-write penalty on small writes.
> Read **[memory/ec-journal-prototype-handoff.md](memory/ec-journal-prototype-handoff.md)**
> before anything else — it opens with a *gate* that may tell you not to build this
> at all, and lists measured dead ends so you do not repeat them.
>
> **Bootstrap facts you will need in the first five minutes:**
> - Work from **198.19.34.70**. Ceph fork `/a/ceph` (= /usr/local/akamai/ceph,
>   branch build/toolchain-selection-20.2.2); scripts `/a/agent-scratchpad/profiling/`;
>   this repo `/agent-context`.
> - Fleet ssh: `ssh -i /root/.ssh/fleet.key root@<ip>`.
> - **Every ceph command must bypass the wedged mgr**, or it hangs:
>   `ceph -m 198.19.32.228,198.19.34.70 ...`
> - **3 of 9 hosts are wedged** (.68 .71 .232) with no console. Their OSDs still
>   serve; you have only 4 usable load clients. See [[test-cluster-infra]].
> - **Two rules that cost us three hosts and a ruined dataset:** never let a
>   profiler read debug symbols on a target host ([[offline-symbolization-method]]),
>   and always quiesce recovery+scrub and verify a 0-reads idle baseline before
>   measuring ([[measurement-harness]]).

# Memory index

One line per fact. Read this first, then open the relevant `memory/*.md`.

## ACTIVE HANDOFF — EC journal / log-structured prototype
- [ec-journal-prototype-handoff](memory/ec-journal-prototype-handoff.md) — **START HERE.** Goal, the gate that decides whether to build at all, design sketch, 2-4 engineer-quarter estimate, and the negative results not to re-litigate.
- [ec-write-path-code-map](memory/ec-write-path-code-map.md) — where the EC write path lives: ECSwitch (legacy vs optimized), ECTransaction RMW/PDW decision, ECExtentCache (the thing to make durable), plugin capability flags, and what does NOT exist.
- [ec-capacity-efficiency-constraints](memory/ec-capacity-efficiency-constraints.md) — 90% efficiency + 22 hosts => 18+2, which makes the full stripe 72 KiB and the journal far more valuable; 90% *sellable* is arithmetically impossible; the IOPS-per-usable-TB conflict.
- [measurement-harness](memory/measurement-harness.md) — the working benchmark/profiling scripts in /a/agent-scratchpad/profiling/ and the method rules (quiesce first, aqu-sz not %util, one-OSD-vs-same-host controls).

## Project findings (crimson/SeaStore evaluation)
- [crimson-replicated-no-stable-layout](memory/crimson-replicated-no-stable-layout.md) — is there a stable crimson/SeaStore layout for replicated (size=5) writes? NO — every layout crashes an OSD in 1-4 min via 2 upstream `ceph_abort` paths (unfixed in v21.3.0); memory solvable, stability not.
- [crimson-vs-bluestore-baseline](memory/crimson-vs-bluestore-baseline.md) — classic BlueStore on same HW: 105k IOPS sustained 25 min, 0 crashes vs crimson ~40k-then-crash → no performance case for crimson today.
- [seastore-single-host-scaling](memory/seastore-single-host-scaling.md) — single-host size=1: 4K IOPS scales with OSD count (single OSD ~9-10k cap) and reactors up to a ~16-32 knee; don't oversubscribe 48 cores.
- [seastore-memory-sizing](memory/seastore-memory-sizing.md) — crimson `--memory` is a PER-SHARD cap (`--memory/reactors`); size >= ~10-12 GB/reactor or it OOMs (self-abort with RAM free).

## Reference (infra, build, repo)
- [test-cluster-infra](memory/test-cluster-infra.md) — LIVE cluster 8f7cb5d2 (9 hosts/45 OSDs, host->osd map, pools, build-id); **3 hosts wedged (.68/.71/.232) with no console, only 4 usable load clients, single mgr, 2-of-3 quorum**; ssh via /root/.ssh/fleet.key.
- [host-safety-core-pattern](memory/host-safety-core-pattern.md) — core_pattern=core + LimitCORE=infinity + cwd=/ + 7.9 GB root: one OSD core fills root and wedges the host; point cores at ZFS first.
- [classic-deb-build-on-70](memory/classic-deb-build-on-70.md) — rebuilding symbol-matched classic ceph-osd debs on .70: docker data-root on ZFS, --network=host, boost slow-mirror fix, make-dist skip via stashed tarball, ccache at /a/ccache (redis off; never CCACHE_REMOTE_ONLY=false).
- [crimson-deb-build-gotchas](memory/crimson-deb-build-gotchas.md) — build-with-container.py gotchas (WITH_CRIMSON truthiness, image network, classic vs crimson).
- [ceph-repo-and-docs](memory/ceph-repo-and-docs.md) — /a/ceph branch, the two benchmark docs, HEAD/author state, boot-fix PR #69972, force-push needed.

## EC benchmark + OSD profiling
- [ec43-load-setup](memory/ec43-load-setup.md) — the EC 4+3 A/B load setup: h0..h8 images (b43/o43/i43), p3_client_fio.sh recipe, last mix4k_w70 numbers, images still mapped.
- [ec-optimizations-ab-result](memory/ec-optimizations-ab-result.md) — fast-EC A/B: ec_optimizations = +29% IOPS on 4k partial-stripe writes, via 33% less write amplification (not fewer RMW reads).
- [ec-extent-cache-not-the-lever](memory/ec-extent-cache-not-the-lever.md) — 18x bigger EC extent cache = -1.7% reads (noise); one-OSD-vs-same-host-controls method; lowers the case for a durable EC journal.
- [bluestore-deferred-writes-negative](memory/bluestore-deferred-writes-negative.md) — deferred writes tested on EC 4+3 small writes: -13.1% IOPS, amplification 15.9x->20.2x. Negative result; don't re-try.
- [ec43-large-write-knee](memory/ec43-large-write-knee.md) — 1 MiB writes ARE device-bound: aqu-sz 1.09->68.93 for only 2.1x tput; ceiling ~18 GiB/s / ~754 MB/s per NVMe; amplification becomes the one lever that matters.
- [ec43-large-write-profile](memory/ec43-large-write-profile.md) — 1 MiB writes: 12 GiB/s, zero RMW reads, 2.17x amplification, OSD 97.2% parked; EC math visible (gf_* routines) but its WALL share FALLS to 0.33% — refutes "ISA-L matters once aligned".
- [ec43-alignment-curve](memory/ec43-alignment-curve.md) — 4k/8k/16k/32k/64k aligned-write curve: device reads hit exactly 0 at the 16 KB full stripe; 8 KB already gives 1.8x; app-batching break-even is only ~1.1-1.2 updates/block.
- [ec43-knee-and-bottleneck](memory/ec43-knee-and-bottleneck.md) — **use the July 11-host/10-client 8+3 sweep for knees (ours is 4-client-limited)**; EC's throughput knee and latency knee differ — p99 is already 1.8s at the throughput plateau, so EC has no safe operating point; NOT cpu/device/net bound; ~9.5ms of 10.3ms op latency is EC partial-stripe RMW; 16k full-stripe = 75x fewer reads, 3.4x bandwidth.
- [ec43-osd-profile-2026-09-11](memory/ec43-osd-profile-2026-09-11.md) — the actual EC 4+3 OSD profile: EC encode only 3.3%; RocksDB memtable + bufferlist copies + allocator dominate; ~85% of threads parked.
- [offline-symbolization-method](memory/offline-symbolization-method.md) — never read big .debug in-process on a fleet host (wedges sshd); capture raw PCs + maps, symbolize offline in a container.
- [profiling-incident-2026-09-11](memory/profiling-incident-2026-09-11.md) — profiling wedged .71; the "22 OSDs down" was a STALE OSDMAP from a mon write-stall caused by 5.47s clock skew; fleet NTP points at the dead .69.
- [unwindpmp-profiling](memory/unwindpmp-profiling.md) — unwindpmp was BROKEN non-interactively (blank frames = max_width 0 on non-TTY, NOT symbols; plus waitpid __WALL race and die()-without-detach) — all three FIXED 2026-09-11.
