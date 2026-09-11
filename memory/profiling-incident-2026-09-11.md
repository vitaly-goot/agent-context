---
name: profiling-incident-2026-09-11
type: project
---
**Profiling osd.10 on .71 wedged .71 and appeared to melt the cluster. Most of
the "meltdown" was a reporting artifact. What actually happened:**
- Ran unwindpmp (unpatched) on osd.10 -> it SEGFAULTED; .71's sshd then started
  refusing at `kex_exchange_identification` (identical to .68). `core_pattern`
  had already been pointed at ZFS on .71, and it wedged ANYWAY -> **the
  "core dump fills the 7.9GB root fs" theory is FALSE / not sufficient.** The
  host-wedge mechanism is still UNKNOWN (candidates: fork/PID or memory
  exhaustion, a blocked core dump leaving D-state, /run|/tmp tmpfs). Needs
  console on .71 to settle. .71's OTHER OSDs (11-14) kept serving fine.
- `ceph -s` then showed 22 OSDs down / 948 PGs inactive / 14.7% degraded. **That
  was a STALE OSDMAP, not reality.** Root cause: mon paxos could not commit
  WRITES (reads worked) while mon.70 had **5.47s clock skew**
  (mon_clock_drift_allowed=1.0). Once writes committed the map caught up to the
  truth: **2689 active+clean, 44/45 up**. Only osd.10 was genuinely down.
  LESSON: if ceph shows mass OSD-down but the OSD processes are alive and idle
  on reachable hosts, suspect a frozen osdmap / mon write stall before believing it.
- **Fleet-wide time problem:** chrony on EVERY host has exactly one NTP source,
  `10.19.34.69` = the DEAD .69. All hosts free-run; external NTP (udp/123) is
  firewall-blocked (EPERM). Fixed .70 by stepping it to match leader .228
  (skew 5.47s -> -0.001s). Drift is ~0.6 s/day, so it re-exceeds 1.0s in ~1.7
  days. DURABLE FIX NEEDED: make a live host the local stratum reference
  (chrony `local stratum 10`) and point the others at it.
- `ceph` CLI hangs because mon+mgr both live on wedged .68 -> always use
  `ceph -m 198.19.32.228,198.19.34.70 ...`.
See [[unwindpmp-profiling]] for the tool bugs (now fixed) and
[[host-safety-core-pattern]] (whose core-dump theory this partly disproves).
