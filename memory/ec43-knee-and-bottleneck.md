---
name: ec43-knee-and-bottleneck
type: project
---
**EC 4+3 (ec43_opt, stripe_width=16384) 4k-randwrite knee and the real bottleneck.**
Measured 2026-09-12 on the 44-OSD cluster, 4 co-located non-mon clients, qd=64.
**Knee = nj4 (1024 outstanding) -> ~131k IOPS / 512 MiB/s, 7.8 ms avg.** Past the
knee throughput is pinned flat while latency explodes:
  nj1  256 out   78.6k  307 MiB/s   3.2 ms   p99   9 ms
  nj2  512       105.3k 411 MiB/s   4.8 ms   p99  26 ms
  nj4  1024      130.9k 512 MiB/s   7.8 ms   p99  54 ms   <- KNEE
  nj8  2048      131.2k 512 MiB/s  19.5 ms   p99 659 ms
  nj16 4096      131.0k 512 MiB/s  48.1 ms   p99 167 ms
  nj32 8192      129.3k 506 MiB/s  65.4 ms   p99 1070 ms
**NOT bound by CPU, device or network** at the knee:
- OSD CPU ~11.4 of 96 cores (12%); 91% of OSD threads parked.
- NVMe: aqu-sz ~1.7-2.2, await 60-85 us. (`%util`=100% is MEANINGLESS for NVMe --
  it only means never-idle; queue depth is the real signal.)
- Cluster net: ~5 Gbit/s of 100 GbE (5%).
**Latency breakdown of the 10.33 ms op_w (osd.35 perf dump deltas):**
  before queue 0.014 | queued 0.257 | prepare 0.121 | **process 10.021**
  BlueStore stages total ~0.5 ms (prepare .135, kv_queued .359, kv_sync .197,
  aio_wait .055, io_done/kv_done/finishing ~0).
  => local storage is ~0.5 ms; **~9.5 ms is EC peer coordination / RMW round-trips.**
  Also throttle-osd_client_messages get_or_fail_fail ~1.8k/s = message back-pressure.
**Proof it is partial-stripe read-modify-write** (4k write touches 1 of 4 data
chunks -> must re-read + recompute 3 parity):
  bs=4k : client 486 MiB/s / 124k IOPS; per host **43,822 device READS/s** + 86,208 w/s
  bs=16k: client **1.6 GiB/s** / 103k IOPS; per host **583 reads/s** + 104,552 w/s
  => full-stripe writes cut device reads **75x** and give **3.4x** client bandwidth.
  Byte amplification 18.4x (4k) vs 5.4x (16k); EC 4+3 theoretical floor is 1.75x.
**Implication:** for small random writes EC 4+3 is RMW-bound, not resource-bound.
Align to 16 KB, or use replication (rep5 hit 138k IOPS from a SINGLE client).
Adding CPU/faster NVMe will not move the 131k ceiling.
See [[ec43-osd-profile-2026-09-11]] (the profile was taken AT this knee).
