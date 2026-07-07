# Crimson/SeaStore vs classic BlueStore — replicated write evaluation

Agent session context. Work spanned 2026-07-03 → 2026-07-07.

## Goal
Evaluate crimson/SeaStore for a fleet rollout: find the best OSD/reactor/memory
layout for replicated write IO, then establish a classic-BlueStore baseline on
the same hardware for an apples-to-apples comparison.

## Bottom line
- **Single-host (size=1):** 4K random-write IOPS scales with **OSD count** (a
  single OSD caps ~9–10k) and with **reactors up to a ~16–32 knee**; never
  oversubscribe 48 physical cores. → run 1 OSD/NVMe, ~9 reactors.
- **Replicated (size=5) crimson: NO stable layout.** Every config (5×4×50,
  2×10×120, 2×4×120, 1×10 unbounded-mem) crashes an OSD in **~1–4 min** of
  sustained write load. Memory is *not* the cause (crashes with no `--memory`
  cap). OSD count / reactors / memory don't fix it.
- **Classic BlueStore, same 40-OSD/size=5/client/workload:** **105k IOPS
  sustained 25 min, 0 crashes**, p50 1.6ms p99 34ms; peak 138k. vs crimson
  ~40k for ~60s then crash. Classic is ~2.6× at matched parallelism AND stable.
  **No performance case for crimson today** — the abort bugs are pure downside.

## Root cause (two upstream `ceph_abort` paths — crimson has no per-shard fault isolation)
1. **SeaStore metadata:** `omap_load_extent` → `crimson::ct_error::assert_all("Invalid error in omap_load_extent")`
   in `src/crimson/os/seastore/omap_manager/btree/omap_btree_node_impl.h` (~line 510/554).
   Bad OMap B-tree extent read aborts the OSD. Same family as
   `transaction_manager.h:1094` "extent checksum inconsistent".
2. **Messenger:** `ceph_abort_msg("TODO")` in `src/crimson/net/FrameAssemblerV2.cc:430` —
   the kernel-RBD-client *late frame-abort* receive path was never implemented
   (literal TODO). Triggered by the krbd load client.
- **Both confirmed present + unfixed in UPSTREAM ceph/ceph v21.3.0** (clone at
  `/usr/local/akamai/upstream/ceph`), a full major version ahead of the 20.2.x
  build. git blame: FrameAssembler = Yingxin Cheng 2023, omap assert_all =
  Xuehan Xu 2025. Zero Akamai commits touch those files.
- Cascade: an abort under load can leave SeaStore on-disk inconsistent → OSD
  crash-loops on restart (extent-checksum) = permanent loss; recovery osdmap
  churn is serialized per-OSD on the `PRIMARY_CORE=0` singleton
  (`src/crimson/osd/shard_services.h`, `with_singleton`→`invoke_on(PRIMARY_CORE)`)
  → reactor stalls → cluster wedge. Plus cold-boot cephx blocks rejoin on aged
  clusters.

## Key numbers
- Single-host size=1: 1/2/4 OSD → ~9k/31k/47–53k 4K IOPS (~12k/OSD).
- 45-OSD size=5 crimson peak: ~110k client (×5 = ~550k OSD-side, ~12k/OSD) — transient, pre-crash.
- 4M single-client ceiling: ~7.2 GiB/s (parallelism-limited; more clients needed for 4M).
- Classic peak sweep (nj4→32 × qd64): 13.4k / 43.9k / 83.7k / 113k / 138k (still climbing).
- Classic soak nj8×qd64: 105k sustained 25 min.
- Memory: per-shard cap = `--memory / reactors`; footprint ~5 GB/shard (~2.15 LRU);
  budget ≥ ~10–12 GB/reactor (old 27G/OSD = 3 GB/shard OOMs).

## Infrastructure
- **8 OSD hosts (Cariboe):** 198.19.34.69/.70/.71/.72, 198.19.32.228/.229/.230/.231.
  Each: AMD EPYC 7643 (48c/96t), 5× 1.7 TB NVMe (nvme1n1..nvme5n1), 251 GB RAM, 100 GbE.
- **Client/driver:** 198.19.34.68 (single client; fine for 4K, bottlenecks on 4M).
- **Excluded:** .232 (permanent OSD loss — crashed + cold-boot cephx, dropped);
  .233 (down); 198.18.140.7 (u22 client candidate, no 10.x cluster-net interface).
- **SSH:** `ssh -i /root/.ssh/bits.key root@198.19.x` (mon_host uses 10.19.x cluster net).
- **Mons:** a198-19-34-69, a198-19-34-70, a198-19-32-228; mgr on .69.
- **Current live cluster:** classic BlueStore, 40 OSD (5/host via ceph-volume),
  pool `rep5` size=5, fsid `e953bbfb-caf6-4033-aed3-1a2b393021cd`, ~105k-capable,
  left UP for further testing.

## Repo / docs / commits
- Repo: `/usr/local/akamai/ceph` (= `/a/ceph`), branch `build/toolchain-selection-20.2.2`.
- Docs (in `doc/dev/crimson/`, in the toctree):
  - `seastore-write-path-single-host-benchmark.rst` (size=1 OSD/core scaling)
  - `seastore-write-path-replicated-benchmark.rst` (size=5 results + classic baseline)
- HEAD `d22824b8ef7` (both docs) → parent `8d40b23e2dc` (boot-fix) → `437c5d93821`.
  Author on both rewritten to **Vitaly Goot <vgoot@akamai.com>**; Claude
  Co-Authored-By lines stripped. Repo git config set to that identity (persistent).
- Boot-fix submitted upstream: https://github.com/ceph/ceph/pull/69972
- Branch history was rewritten (squash + author) → needs **force-push** to
  `github.com/vitaly-goot/ceph.git`. Backup ref: `backup-before-author-rewrite` = `c348ac0402d`.
- Uncommitted/untracked in working tree (NOT part of doc commit): modified
  submodules (build drift), `ceph-menv/*` deletions, `src/script/custom/config-classic.env`, build dirs.

## Build artifacts
- Crimson debs: `/usr/local/akamai/ceph/build-crimson-gcc13-debs` (WITH_CRIMSON=1, crimson-osd).
- Classic debs: `/usr/local/akamai/ceph/build-classic-gcc13-debs/ubuntu/WORKDIR`
  (`ceph-osd_20.2.1-1noble_amd64.deb`, classic BlueStore, ~26 MB, 0 seastar symbols).
- Upstream clone: `/usr/local/akamai/upstream/ceph` @ v21.3.0.

## Scratchpad / harness
`/tmp/claude-0/-usr-local-akamai/acff6945-.../scratchpad/rebuild/` — provision
scripts (provision8.sh 5×4×50, provision2x.sh, provision1x.sh, provision-classic*.sh),
soak drivers (soakstop*.sh halt-on-first-crash, classictest.sh), keyrings, nodes*.txt, logs.

## Repeatable test plan (next steps)
1. Crimson cluster: 8 hosts × 5 OSD (40), size=5, 1 OSD/NVMe.
2. Warm up / precondition: ~1 TiB via 4 MiB sequential writes.
3. Soak: 4K random write, sweep concurrency, measure IOPS + latency.
4. Same layout with classic OSD → baseline + comparison.

## Gotchas learned
- **`WITH_CRIMSON=0` is TRUTHY** in build-with-container.py (`bool("0")==True`) →
  a *classic* build needs it ABSENT, not `=0`.
- Image `docker build` lacks `--network=host` (only `docker run` gets `--extra`);
  apt fails → pre-build/re-label the build image, or the make-debs step reuses it.
- `ceph-volume` refuses devices with partitions → had to remove a stale, unmounted
  `nsds_volume_group` LVM off nvme1–5 (safe: OS root = nvme0n1, /usr/local/akamai = ZFS tank).
- Crimson pinned cores (`crimson_seastar_cpu_cores`) are fixed at `mkfs` →
  changing layout needs purge+reprovision (which is why the boot-fix mattered).
- In-place OSD reprovision reliably wedges the cluster → **fresh mon redeploy per config**.
- `fio` `rados` ioengine hangs on crimson → drive via kernel-mapped RBD + `libaio`.
- `.68` reboot wipes the iptables egress rule for the cluster net (re-add to reach mons).
