---
name: host-safety-core-pattern
type: feedback
---
**Before any risky attach/crash experiment on a fleet host: point core dumps at
ZFS.** Hosts ship `kernel.core_pattern=core` (relative -> daemon cwd) and the
ceph-osd unit has LimitCORE=infinity with cwd=/. One OSD crash (RSS 6-8 GB)
writes `core` into the 7.9 GB root fs and fills it; a full root breaks sshd,
mon (self-stops at mon_data_avail_crit) and journald -> host looks "lost".
Fix (runtime, per host): `sysctl -w kernel.core_pattern=/usr/local/akamai/coredumps/core.%e.%p.%t`
(dir exists, ZFS has ~470 GB). Same hazard for the profiler itself: run it from a
ZFS cwd. Also: /var/log/ceph logs were rotated 2026-09-04 and never reopened
(0-byte files) -> use journalctl -u 'ceph-*' for daemon output on these hosts.
