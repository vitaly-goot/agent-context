---
name: crimson-vs-bluestore-baseline
type: project
---
**Classic BlueStore baseline (same HW, same 20.2.1 source WITH_CRIMSON off, same
40-OSD/size=5/client/workload) is FASTER AND stable — crimson has no perf case.**
Classic peak sweep nj4-32 x qd64 = 13.4k/43.9k/83.7k/113k/138k (still climbing =
single-client-limited). 25-min soak nj8 x qd64 = **105k IOPS sustained, 40/40 up
entire time, 0 crashes**, p50 1.6ms p99 34ms. vs crimson at same nj8 x qd64:
~38-45k for ~60s then crash. Classic ~2.6x at matched parallelism.
Classic debs built via build-with-container.py `-b build-classic-gcc13-debs`
(classic = WITH_CRIMSON ABSENT); ceph-volume BlueStore OSDs. Live cluster left up.
