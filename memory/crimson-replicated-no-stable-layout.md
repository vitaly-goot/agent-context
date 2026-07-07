---
name: crimson-replicated-no-stable-layout
type: project
---
**Under replication (size=5), crimson/SeaStore has NO stable layout.** Every
config crashes an OSD within ~1-4 min of sustained write load; layout only
changes *when*, not *whether*:
- 5x4x50 (40 OSD, 160 reactors): first crash ~63s into 4K soak.
- 2x10x120 (16 OSD, 160 reactors): 2 crashes during precondition (~3.7min).
- 2x4x120 (16 OSD, 64 reactors, 30GB/shard): crash during precond (~2.5min).
- 1x10 unbounded-mem (8 OSD): precond survived, crash 36s into soak.
"Fewer total reactors = stable" DISPROVED (64-reactor crashed sooner than 160
with the most memory). OSD count / reactors / memory don't fix it. Only
crash-free runs were single-host size=1 (no replication path exercised).

Two upstream `ceph_abort` paths cause it (crimson has NO per-shard fault
isolation — one shard abort kills the whole OSD):
1. SeaStore: `omap_load_extent` -> `crimson::ct_error::assert_all("Invalid error
   in omap_load_extent")` (omap_btree_node_impl.h). Same family as
   transaction_manager.h:1094 "extent checksum inconsistent".
2. Messenger: `ceph_abort_msg("TODO")` at FrameAssemblerV2.cc:430 — kernel-RBD
   late frame-abort receive path never implemented.
Both CONFIRMED present+unfixed in upstream ceph/ceph v21.3.0 (clone at
/usr/local/akamai/upstream/ceph); git blame = upstream authors 2023/2025, zero
Akamai changes. No newer build fixes them. Follow-on: abort leaves SeaStore
on-disk inconsistent -> crash-loop on restart = permanent loss; recovery osdmap
churn serialized on shard-0/PRIMARY_CORE singleton -> stalls -> wedge; cold-boot
cephx blocks rejoin. Needs upstream fixes before any rollout.
