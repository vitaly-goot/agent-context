---
name: unwindpmp-profiling
type: project
---
**Wall-clock profiling of ceph-osd with unwindpmp (markhpc/uwpmp, /a/uwpmp,
binary /a/uwpmp/build/unwindpmp, built 2026-09-10 on .70, commit 2758308).**
Usage: `unwindpmp -p PID -n SAMPLES [-s ms] [-v inverted] [-t thresh] [-b libunwind|libdw] [-j jobs]`.
- libunwind backend (default) attaches ONE THREAD AT A TIME (PTRACE_ATTACH ->
  waitpid -> unw_step loop -> PTRACE_DETACH); the OSD is never fully frozen.
  `-j` only applies to libdw. `-b libdw` hangs >10 min per sample on ceph-osd.
- Symbols: libunwind resolves separate debug info only through .gnu_debuglink,
  searching <bindir>/, <bindir>/.debug/, /usr/lib/debug/<bindir>/<link name>;
  no build-id lookup. Recipe: extract ceph-osd-dbg + ceph-base-dbg debs to
  /a/agent-scratchpad/profiling/dbg/usr/lib/debug, symlink /usr/lib/debug there,
  and link each mapped file's debuglink name -> .build-id/xx/yyyy.debug.
  The dbg debs MUST match the running build-id (readelf -n /usr/bin/ceph-osd).
  On .70 no -dbg deb matches (08f015eb); debs on .70: /root/debs-new = 08f015eb
  (installed), /root/debs-classic = d1849e73, /root/debs202 = 32138a07.
- Robustness bugs (src/tracer/unwind_tracer.cc): `waitpid(tid,NULL,0)` on a
  non-leader thread returns ECHILD at once (needs __WALL) so unwinding races the
  stop -> ESRCH -> die(); every die() exits WITHOUT PTRACE_DETACH; no SIGINT
  handler, no per-thread timeout. Bigger risk under load than idle.
- 2026-09-10: hosts .69 and .68 were lost during "profile under load" attempts
  (root cause not established from .70; see [[host-safety-core-pattern]] for the
  root-fs-fill path). Next attempt rule: single probe (`-n 1`), libunwind, run
  under `timeout` + `systemd-run -p MemoryMax=`, core_pattern on ZFS, watch OSD
  thread states for `T` afterwards, and prefer an OSD on a host that can be lost
  rather than the working host.
