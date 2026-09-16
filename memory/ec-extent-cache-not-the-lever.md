---
name: ec-extent-cache-not-the-lever
type: project
---
**An 18x larger EC extent cache does NOT reduce read-modify-write reads: -1.7%
(noise). Measured 2026-09-16. This lowers the expected value of building a
durable write-coalescing journal for THIS workload.**
`ec_extent_cache_size` (default 10 MiB *per OSD shard*, x8 shards = ~80 MiB/OSD)
is read ONCE in the OSDShard constructor -- no config observer, no resize path --
so changing it needs an OSD restart.
**Experiment design (clean, worth reusing):** raise the cache on ONE OSD only and
compare its NVMe against the other four on the SAME host, under the SAME load, at
the SAME moment. osd.35..39 map 1:1 to nvme1n1..nvme5n1 on .231, so nvme1n1 is the
subject and nvme2-5 are simultaneous controls. Eliminates run-to-run variance.
  osd.35 ec_extent_cache_size 10 MiB -> 160 MiB/shard, restart, recover to clean.
  Cache really grew: mempool 70.2 MiB -> **1,280.6 MiB** (8 x 160 MiB). Confirmed used.
  device reads/s   osd.35   controls(mean)   ratio
    baseline        8,244      8,839         0.9327
    16x cache       8,142      8,880         0.9168
  => normalised change **-1.7%** = noise.
**Why:** the benchmark is uniform-random 4 KiB over 100 GiB images x4 clients, so
temporal locality is ~nil. The cache's own header says it targets "small
sequential writes" -- this workload has none. Consistent with the batching
break-even ([[ec43-alignment-curve]]): you need ~1.2 updates per 16 KiB block and
uniform-random over 100 GiB delivers essentially zero.
**Implication:** a durable journal (design option B) coalesces over a longer
window than an LRU, so it is not strictly refuted -- but if 18x more cache finds
no neighbours, the write stream has none to find. Build it only if the PRODUCTION
write-size/locality data says otherwise. Measure first ([[ec43-knee-and-bottleneck]]).
Also noted: both jerasure AND isa advertise PARTIAL_READ/PARTIAL_WRITE/PARITY_DELTA
plugin flags, so PDW is already active on the jerasure pool; ISA-L unlocks nothing
structural, it is only faster math.
