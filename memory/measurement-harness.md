---
name: measurement-harness
type: reference
---
**Working benchmark + profiling harness, all in `/a/agent-scratchpad/profiling/`
on .70 (= /usr/local/akamai/agent-scratchpad/profiling).** Reuse it; it took a
week and three wedged hosts to get right.

## Load generation
- `fio_runner.sh <name> <rw> <bs> <qd> <nj> <rt> <mix>` — backgrounds fio on the
  local host against `/dev/rbd0:/dev/rbd1`, pinned `taskset -c 48-95` so the
  co-resident OSDs keep the lower cores. `fio_dev.sh` takes an explicit device list;
  `fio_point.sh` is the BLOCKING variant for sweeps.
- `sweep.sh` — concurrency sweep; `start_load.sh` / `stop_load.sh` — fleet-wide.
- Each client maps its own images; see [[ec43-load-setup]].

## Probes
- `iostat_probe.sh <sec>` — per-NVMe r/s w/s MB/s + `%util`, and per-NIC Mbit/s,
  straight from /proc (no sysstat dependency).
- `qdepth_probe.sh <sec>` / `bigprobe.sh <sec>` — **aqu-sz** (average queue depth),
  await, util, cluster-net tx.
- **`%util` is USELESS for NVMe** — it read 95-99% at every point including
  aqu-sz=1.09. Always judge saturation by **aqu-sz**, not %util.

## Profiling (see [[offline-symbolization-method]] — read it before profiling)
- `deploy_runtime_only.sh <ip> <osd>` — installs the symbol-matched runtime debs
  and restarts ONE OSD; deliberately removes `/usr/lib/debug` on the target.
- `capture_raw.sh <osd> <n>` — `unwindpmp -R` (raw PCs) under
  `systemd-run --scope -p MemoryMax=4G`, plus /proc/PID/maps and per-module build-ids.
- `symbolize_offline.py <prof> <maps> <modules> <dbgroot> [out]` — run INSIDE the
  container with the dbg tree bind-mounted at /usr/lib/debug (DWZ alt-files use
  absolute paths). Uses `addr2line -p -i -f -C`. ~92% resolve.
- `analyze_profile.py <sym.txt>` — classifies by addr2line SOURCE FILE first, then
  symbol name, then **caller attribution** for generic boost/std frames and libc
  leaves. That last rule is what puts rbtree work under BlueStore and what the
  parked/on-CPU split rests on. Unclassified went 29% -> 1.6%.
- Debug files for build-id ea46b25d are extracted at `profiling/dbg-local/`.

## Prometheus exporter for production write sizes
`ceph_iosize_exporter.py [--port 9284|--once]` + `ceph-iosize-exporter.service`
+ `prometheus-scrape-snippet.yml`. Collapses the latency axis of
`osd.op_w_latency_in_bytes_histogram` into a real Prometheus histogram.
**`le` is the bucket's inclusive max**, so partial-stripe share for a 16 KiB stripe
uses `le="16383"`, NOT 16384. `_count` reconciles exactly with `op_w` and `_sum`
with `op_w_in_bytes` — use that as the correctness check.

## Method rules learned the hard way
1. **Quiesce before measuring**: `ceph osd set noscrub nodeep-scrub nobackfill norecover`,
   then VERIFY an idle baseline of ~0 device reads. Backfill/deep-scrub were
   injecting **56,000 reads/s per host at idle** and silently ruined a whole sweep.
   Unset every flag afterwards — `norecover` left on is dangerous.
2. **One-OSD-vs-same-host-controls**: to test an OSD-level knob, change ONE OSD and
   compare its NVMe against the other four on the same host, same load, same
   moment (osd.35..39 map 1:1 to nvme1n1..nvme5n1 on .231). Kills run-to-run variance.
3. **Verify fio is actually running on every client** before trusting a sample —
   a bogus 32k point came from sampling during ramp.
4. **Bypass the wedged mgr**: `ceph -m 198.19.32.228,198.19.34.70 ...` or commands hang.
5. **Sweep client COUNT, not just per-client concurrency** — our 131k 4K "knee" may
   be a 4-client ceiling, not a cluster ceiling. Adding jobs to the same clients
   cannot distinguish the two. UNRESOLVED; see [[ec43-knee-and-bottleneck]].
