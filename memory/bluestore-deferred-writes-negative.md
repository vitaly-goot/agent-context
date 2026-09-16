---
name: bluestore-deferred-writes-negative
type: project
---
**BlueStore deferred writes make EC 4+3 small writes WORSE: -13.1% IOPS. Tested
2026-09-16, do not re-try without a new reason.**
`bluestore_prefer_deferred_size_ssd` 0 -> 16384, cluster-wide, A/B at the knee
(4k randwrite nj4 qd64, 4 clients, ec43_opt, recovery+scrub quiesced, idle
baseline 0 reads/s, fio verified 20/20 both runs).
                    deferred OFF   deferred ON     delta
  client IOPS         124.66k        108.36k      **-13.1%**
  client BW           487 MiB/s      423 MiB/s     -13.1%
  device reads/s       42,976         37,275       -13.3%
  device writes/s      85,396         75,477       -11.6%
  device write MB/s     903.7          996.5      **+10.3%**
  deferred_queued          0/s        9,488/s     (path confirmed engaged)
  kv_sync               4,767/s        4,867/s      +2%
**Mechanism, visible in the data: fewer but FATTER device writes.** Write IOPS
fell 11.6% while write BYTES rose 10.3% -- the double-write signature (data goes
to the RocksDB WAL and then to its final location). Amplification per client 4 KiB
op went **15.9x -> 20.2x (+27%)**. The cluster is amplification-bound
([[ec43-knee-and-bottleneck]]), so trading device IOPS (which were idle at
queue depth ~2) for more bytes is a straight loss.
**Why it was never going to help:** deferred writes act on the LOCAL BlueStore
commit, which is only ~0.5 ms of the 10.33 ms op latency; the ~9.5 ms is EC peer
coordination, a layer above. And the EC shard write is exactly 4096 B =
bluestore_min_alloc_size_ssd, so there is no sub-allocation RMW for deferral to
remove in the first place.
**Prediction check:** direction was right (no help) but the magnitude band was
wrong -- predicted "<=5% change", actual -13.1%. Reverted with
`ceph config rm osd bluestore_prefer_deferred_size_ssd`.
