---
name: ec43-osd-profile-2026-09-11
type: project
---
**Wall-clock profile of a classic BlueStore OSD under EC 4+3 load** (osd.35 on
.231, **500 probes**, offline-symbolized WITH addr2line -i inline expansion;
method in [[offline-symbolization-method]]). Supersedes the earlier 100-probe
numbers (inline expansion + 5x samples moved several categories materially).
**Load:** pool ec43_opt (k=4 m=3, ec_optimizations), 4 co-located fio clients,
4k randwrite qd64 nj4 -> 490 MiB/s / 125k write IOPS cluster-wide; OSDs
~1040-1138% CPU (~11 of 96 cores). 69 threads/OSD. 13660 addrs, 92% resolved.
**91.2% of sampled thread-time is PARKED** (libc condvar/futex waits) -- the box
is nowhere near CPU-bound at this concurrency. ON-CPU ceph code splits as:
  BlueStore      23.9%   Buffer/encode  17.4%   OSD op/types 14.5%
  RocksDB        12.8%   **EC 7.8%**            Alloc/stdlib  5.4%
  Msgr/net        4.7%   Scheduler       2.5%   mempool       2.4%
  CRC             2.3%   Locking         2.1%   PerfCounters  1.9%
  Tracing(otel)   0.7%   unclassified    1.6%
**EC encode is NOT the bottleneck (7.8%)**; BlueStore + the KV/metadata write
path dominate. (An earlier name-regex pass said 3.3% and had 29% unclassified --
both wrong; see the classification rules below.)
Hot leaves: boost::intrusive rbtree minimum 7.9% (BlueStore extent/blob maps);
rocksdb InlineSkipList FindGreaterOrEqual 3.3% + FindSpliceForLevel 1.2%
(memtable insert dominates the KV path); atomic load 1.8%;
**ceph::buffer list::operator= 1.8% (a COPY)**; ptr_node::dispose 1.3%;
hobject_t::operator<=> 1.3%; PerfCounters::inc 1.2%; mempool adjust_count 1.2%;
maybe_inline_memcpy 0.9%; denc_lba 0.6%.
=> **~5% of on-CPU time is pure accounting/telemetry** (mempool 2.4 +
PerfCounters 1.9 + otel 0.7) plus the bufferlist copy -- cheap wins to chase.
**Classification that works** (analyze_profile.py): classify by addr2line SOURCE
FILE first (catches crc32_iscsi_01.asm etc.), then symbol name, then CALLER
attribution for generic boost/std frames (this is what reveals the rbtree ops as
BlueStore). Unclassified 29% -> 1.6%.
Artifacts: /a/agent-scratchpad/profiling/osd35_raw_n500{,.sym}.txt + analyze_profile.py
