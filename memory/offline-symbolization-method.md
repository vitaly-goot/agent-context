---
name: offline-symbolization-method
type: feedback
---
**NEVER let a profiler read big .debug files in-process on a fleet host — it
wedges the host. Symbolize OFFLINE instead.** Reading the ~220MB (2GB expanded)
ceph-osd .debug via .gnu_debuglink inside libunwind exhausted memory on .71 and
.232: sshd then fails at `kex_exchange_identification` (it forks per connection),
while already-running daemons keep serving. Proven by controlled A/B on .232:
100 probes WITHOUT debug info = clean; 1 probe WITH = instant wedge.
**Method that works** (scripts in /a/agent-scratchpad/profiling/):
1. Target runs the build-id-matched binary but has **NO /usr/lib/debug**
   (`deploy_runtime_only.sh <ip> <osd>`: installs only ceph-osd/ceph-base/
   librados2 under a policy-rc.d guard, restarts ONE osd, removes /usr/lib/debug).
2. `capture_raw.sh <osd> <n>`: `unwindpmp -R` (raw PCs, never calls
   unw_get_proc_name) under `systemd-run --scope -p MemoryMax=4G`; also saves
   /proc/PID/maps and each mapped ELF's build-id.
3. `symbolize_offline.py <prof> <maps> <modules> <dbgroot> [out]` on the analysis
   host, run INSIDE the container with the dbg tree bind-mounted at
   /usr/lib/debug (DWZ .gnu_debugaltlink uses absolute paths) and --memory=16g.
   pc -> mapping -> file offset -> PT_LOAD -> ELF vaddr -> addr2line. ~91% resolve.
**Gotcha:** resolve libc frames with the TARGET's own libc (copy it back) —
build-ids differ per host and a mismatched libc yields garbage. Even then,
non-exported glibc internals land on the nearest export
(`__nptl_death_event@@GLIBC_PRIVATE` ~76%) = parked condvar/futex threads, NOT
real work. Always run every profiler invocation under a MemoryMax cap.
