---
name: test-cluster-infra
type: reference
---
**Live cluster (since 2026-09-03, redeployed; supersedes fsid e953bbfb/a276ddc3):**
fsid 8f7cb5d2-8beb-43bb-8a57-5b4bed75684a, ceph 20.2.1 tentacle sha c44ec591ab6
(= HEAD of /a/ceph branch build/toolchain-selection-20.2.2), classic BlueStore,
RelWithDebInfo, debs from /root/debs-new on each host (ceph-osd build-id
08f015eb13557c24b18966378748b9aeeba5e551, stripped, NO -dbg deb kept anywhere
reachable). 9 OSD hosts x 5 NVMe = 45 OSDs, all up/in, all PGs active+clean:
.68=osd.0-4 .70=osd.5-9 .71=osd.10-14 .72=osd.15-19 .228=osd.20-24
.229=osd.25-29 .230=osd.30-34 .231=osd.35-39 .232=osd.40-44 (osd id = 5*hostidx).
Mons .68/.70/.228 (quorum .228 leader + .70; mon.68 out since 2026-09-05);
mgr active on .68. Pools: rbdmeta(rep3) ec43_base(3) ec43_opt(4, ec_optimizations)
ec43_opt_isa(5) rep5(6). OSD unit: /usr/bin/ceph-osd -f --id N, runs as root,
cwd=/, LimitCORE=infinity (see [[host-safety-core-pattern]]).
**Each host:** AMD EPYC 7643 48c/96t, 251 GB RAM + 251 GB swap, 5x Micron 7400
1.7 TB, root fs 7.9 GB (tiny), ZFS tank/ula at /usr/local/akamai (= /a, ~470 GB).
**Lost hosts (2026-09-10, while profiling under load with unwindpmp):**
.69 (former orchestration/build host, NOT in this cluster) = no ping on 198.19/10.x
-> hard down; the matching -dbg debs and build tree (build-classic-gcc13-debs)
lived there. .68 = pings, its OSDs/mgr still serve, but sshd resets at
kex_exchange_identification (cannot fork children: full root fs / mem / pids?).
**Working from .70 now.** No SSH private key on .70 (bits.key was on .69);
ssh to .71 -> publickey denied. /agent-context clone + /a/uwpmp on .70.
.233 down for weeks; 198.18.140.7 unusable (no cluster net).
