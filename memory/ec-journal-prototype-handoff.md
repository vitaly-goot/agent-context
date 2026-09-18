---
name: ec-journal-prototype-handoff
type: project
---
**HANDOFF GOAL: prototype a durable write-coalescing journal ("log-structured
machinery") in Ceph's EC backend, to eliminate partial-stripe read-modify-write
for small writes.** Everything below is measured unless marked otherwise.

## READ THIS FIRST — the gate. Do not start building yet.
Two measurements decide whether this prototype is worth its 2-4 engineer-quarters:
1. **Production write-size distribution.** Unmeasured by anyone. If most writes are
   >= the full stripe, there is no problem to solve. Ceph already records
   `osd.op_w_latency_in_bytes_histogram` per OSD; the exporter is written and
   tested (see [[measurement-harness]]). The KPI is the share of writes below the
   full-stripe boundary. **The T-shirt requirements imply ~15 KB average I/O**
   (see [[ec-capacity-efficiency-constraints]]), NOT the 4 KB everyone benchmarks.
2. **Write locality.** [[ec-extent-cache-not-the-lever]]: an 18x larger EC extent
   cache (70 MiB -> 1,281 MiB, confirmed in the mempool) changed device reads by
   **-1.7%, i.e. noise**. A journal coalesces over a longer window than an LRU so
   this is not a strict refutation, but **if 18x more cache finds no neighbours,
   the write stream has none to find.** Value depends entirely on production
   having locality that uniform-random fio does not.
If (1) shows mostly large/aligned writes, or (2) shows no locality, **the correct
outcome is to not build this** and say so.

## Why it would help (the measured problem)
[[ec43-knee-and-bottleneck]], [[ec43-alignment-curve]]: on EC 4+3 with
stripe_unit 4096, full stripe = 16 KiB. A 4 KiB write touches 1 of 4 data chunks
but all 3 parity must be recomputed -> read old data/parity back. Measured on a
**pure-write** workload: **43,077 device reads/s per host**, 18.4x byte
amplification, and ~9.5 ms of the 10.33 ms op latency is EC peer coordination
(local BlueStore is only ~0.5 ms). Aligning to 16 KiB drops device reads to
**exactly 0** and gives **3.4x** bandwidth. The journal's job is to get that
full-stripe behaviour without requiring the application to align.

## Do NOT redesign what already exists
See [[ec-write-path-code-map]]. The optimized EC path is not naive; it already has
**true partial writes**, **adaptive parity-delta writes**, and **ECExtentCache**
(in-memory coalescing). Both jerasure and isa advertise all three capability
flags, so none of it is plugin-gated. The 43k reads/s is what SURVIVES all that.
There is **no journal/log-structured machinery anywhere in osd/ or os/**.

## Design sketch (option B of three; A and C were rejected)
Make `ECExtentCache` **durable** and give it a **write-back policy**:
- partial-stripe write -> append to a per-PG journal -> ack client -> mark extents dirty
- flusher: when an object has a full stripe dirty (or timer/pressure), compute
  parity and issue ONE full-stripe write, no read-back; then trim the journal
- read path: merge base object + un-flushed journal (the cache already merges in memory)
- peering replays the journal
**Effort 2-4 engineer-quarters = 6-12 person-months.** Happy path is only ~4-6
weeks; the rest is correctness surface: on-disk format + versioning,
crash-consistent replay wired into peering, scrub (flush-first or journal-aware),
backfill/recovery of un-flushed extents, snapshot/clone interaction, journal-full
back-pressure, teuthology coverage. **This estimate is inference from reading the
code, not measurement** — unlike the rest of this file.
Rejected alternatives: **A. librbd PWL** (`librbd/cache/pwl/` already ships a
log-structured persistent cache, but it is librbd-only and we run krbd, and the
cache is client-local); **C. full log-structured object layout** (changes the
RADOS object model scrub/recovery/snapshots/omap depend on — years or a fork).

## Build it UPSTREAM, not in the fork
`src/osd/ECSwitch.h` describes itself as a *temporary* switcher and states the
legacy implementation is to be deleted. This area is actively moving upstream.
A journal carried downstream would make the rebase burden permanent, and we
already carry a fork (see [[ceph-repo-and-docs]]).

## Design input you must not miss
If production adopts **EC 18+2 for the 90% efficiency target**, full stripe
becomes **72 KiB** instead of 16 KiB ([[ec-capacity-efficiency-constraints]]).
Every write under 72 KiB then pays RMW — including the ~15 KB implied workload.
**That makes the journal far MORE valuable, and it is the strongest argument for
building it.** Conversely at 4+3 with a 16 KB workload, simple alignment may be
enough and the journal is unnecessary. Decide the EC profile before the prototype.

## Negative results — do not re-litigate
- [[bluestore-deferred-writes-negative]] BlueStore deferred writes: **-13.1% IOPS**,
  amplification 15.9x->20.2x. Wrong layer (local commit is 0.5 ms of 10.33 ms).
- [[ec-extent-cache-not-the-lever]] 18x extent cache: -1.7%.
- ISA-L: <1% end to end in BOTH regimes ([[ec43-large-write-profile]]); also NOT a
  flag — erasure-code profile is fixed at pool creation, so it needs a new pool
  and a data migration.
- Large writes are already near optimal: 2.17x amplification vs a 1.75x floor, and
  **device-bound** ([[ec43-large-write-knee]]). The journal targets SMALL writes only.
