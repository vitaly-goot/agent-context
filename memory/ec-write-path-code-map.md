---
name: ec-write-path-code-map
type: reference
---
**Where Ceph 20.2.1 Tentacle's EC write path lives** (repo /a/ceph = /usr/local/akamai/ceph,
branch build/toolchain-selection-20.2.2). Read before touching the EC code.

## The two implementations
`src/osd/ECSwitch.h` switches between them per pool on `allows_ecoptimizations()`
(the `ec_optimizations` pool flag). Files without `L` = **optimized**; files with
`L` suffix (`ECBackendL.cc`, `ECCommonL.cc`, `ECTransactionL.cc`, `ECExtentCacheL.*`)
= **legacy**. The header states the switcher and the legacy version are TEMPORARY
and to be deleted — treat this area as actively moving upstream.

## Where read-modify-write is decided
`src/osd/ECTransaction.cc` (~1047 lines), the write-planning function:
- `sinfo.supports_partial_writes()` — legacy pads every write to a full stripe:
  *"If partial writes are not supported, pad out to_write to a full stripe"* (~line 53).
  The optimized path does not.
- builds `reads` = extents that must be read back to recompute parity
- also builds `pdw_reads` = **parity-delta-write** read set (old data + old parity)
- ~line 206 picks between them: *"opt for whichever is less reads"*, with fallbacks
  when a shard would need reconstruct. `to_read = std::move(pdw_reads)` on PDW.
- `ec_pdw_write_mode` (global.yaml.in, dev-level): 0=optimal 1=never 2=when possible

## The cache to make durable
`src/osd/ECExtentCache.{h,cc}` — per-shard cache of recent reads/writes plus a
per-OSD-shard LRU. Its own header says it targets *"small sequential writes"*.
- `LRU(uint64_t max_size)` — size captured ONCE in the `OSDShard` constructor
  (`src/osd/OSD.cc` ~line 11141, `ec_extent_cache_lru(cct->_conf.get_val<uint64_t>("ec_extent_cache_size"))`).
  **No config observer, no resize path -> changing `ec_extent_cache_size` needs an
  OSD restart.** Default 10 MiB per OSD shard, x8 shards = ~80 MiB/OSD.
- uses mempool `mempool_ec_extent_cache`, so occupancy is visible via
  `ceph daemon osd.N dump_mempools` — that is how we proved the 18x bump took effect.
- There are **no hit/miss perf counters**. Instrumenting them is a cheap first task.

## Plugin capability flags (NOT pool flags)
`src/osd/ECUtil.h` ~660-680: `supports_partial_reads()`, `supports_partial_writes()`,
`supports_parity_delta_writes()` all test `plugin_flags` against
`FLAG_EC_PLUGIN_{PARTIAL_READ,PARTIAL_WRITE,PARITY_DELTA}_OPTIMIZATION`
(defined `src/erasure-code/ErasureCodeInterface.h` ~651-689).
**Both `erasure-code/jerasure/ErasureCodeJerasure.h` and `erasure-code/isa/ErasureCodeIsa.h`
advertise all three**, as do shec/clay/lrc. So PDW is already active on a jerasure
pool and switching to ISA-L unlocks nothing structural.

## What does NOT exist
`grep -rl "ec_journal\|EcJournal\|log_structured" osd/ os/` -> **nothing**. There is
no journal or log-structured machinery in the OSD.

## What DOES exist, client-side
`src/librbd/cache/pwl/` — a real log-structured persistent write cache
(`LogEntry.h`, `LogMap.h`, `SyncPoint.h`), driven by `rbd_persistent_cache_mode`
/ `_size` / `_path` in `common/options/rbd.yaml.in`. **librbd only — we run krbd**,
and the cache is client-local (un-flushed data dies with the client).

## Stripe geometry
full stripe = `k x osd_pool_erasure_code_stripe_unit` (default 4096, and 4096 is
effectively the floor since it matches `bluestore_min_alloc_size_ssd`).
4+3 -> 16 KiB · 8+3 -> 32 KiB · 18+2 -> 72 KiB · 20+2 -> 80 KiB.
Any write smaller than that is partial-stripe and pays RMW.
