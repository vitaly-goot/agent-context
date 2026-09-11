---
name: ec43-osd-profile-2026-09-11
type: project
---
**First trustworthy wall-clock profile of a classic BlueStore OSD under EC 4+3
load** (osd.35 on .231, 100 probes, offline-symbolized; see [[unwindpmp-profiling]]).
**Load:** pool ec43_opt (k=4 m=3, ec_optimizations), 4 co-located fio clients,
4k randwrite qd64 nj4 -> 491 MiB/s / 126k write IOPS cluster-wide; OSDs
~1040-1138% CPU (~11 of 96 cores). 69 threads/OSD.
**~84-90% of sampled thread-time is PARKED** (condvar/futex) — the box is far
from CPU-bound at this concurrency. Distribution of the ON-CPU ceph code:
  Encoding/buffer (ceph::buffer)  17.5%   RocksDB            13.9%
  Allocator/libstdc++             15.6%   BlueStore           8.4%
  Messenger/net                    5.0%   PG/OSD op path      4.6%
  **EC / erasure                   3.3%**  Locking            1.9%   CRC 0.9%
  other/unclassified              29.1%
**Key finding: EC encode is NOT the bottleneck (3.3%).** The cost is the
BlueStore/RocksDB metadata write path + buffer copying + allocation.
Hot leaves: boost intrusive rbtree ops 4.1%; rocksdb InlineSkipList
FindGreaterOrEqual 1.9% / FindSpliceForLevel 1.5% / MemTable KeyComparator 1.4%
(memtable insert dominates the KV path); hobject_t::operator<=> 1.7%;
ceph::buffer list::operator= 1.4% (a COPY) / ptr::append 1.4% / ptr::release 0.9%;
mempool::pool_t::adjust_count 1.5% + shard_t[] 1.5% (mempool accounting ~3%);
PerfCounters::inc+tinc ~1.9%; opentelemetry TraceState::GetDefault 0.9%.
=> cheap wins to investigate: mempool accounting, PerfCounters, otel tracing,
and the bufferlist copy — together several % of on-CPU time.
Raw + symbolized output: /a/agent-scratchpad/profiling/osd35_raw_n100*.txt
