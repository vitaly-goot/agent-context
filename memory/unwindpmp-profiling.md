---
name: unwindpmp-profiling
type: project
---
**unwindpmp (markhpc/uwpmp) was BROKEN for non-interactive use; fixed 2026-09-11
in /a/uwpmp (local commit 22967f0). Three real defects:**
1. **"Thread headers but blank frames" is NOT a symbolization problem.**
   `UwpmpCtx` ignored `ioctl(TIOCGWINSZ)`'s return, so when stdout is not a TTY
   (redirect/pipe/agent harness) `ws_col==0` -> `max_width=0` -> `fprint()` does
   `line.substr(0, 0)` -> EVERY frame line prints as "". Thread headers survive
   because `UwpmpThread::print()` uses std::cout directly. **This is why it
   "works on Ubuntu 22" (run in a terminal) and looks broken from a script — it
   is TTY vs non-TTY, NOT a distro/u24 issue.** Workaround on an unpatched
   binary: always pass `-w 200`.
2. `waitpid(tid, NULL, 0)` lacked `__WALL`. Threads are CLONE_THREAD tasks, so
   waitpid returns ECHILD immediately instead of waiting for the ptrace-stop;
   unwindpmp then unwinds a NOT-yet-stopped thread -> garbage frames and
   **SIGSEGV** on busy many-threaded targets (ceph-osd, ~1000 threads). Timing
   dependent, which is why it can appear to work on smaller/slower systems.
3. Every `die()` between PTRACE_ATTACH and PTRACE_DETACH exited WITHOUT
   detaching -> tracee threads can be left in group-stop (T) forever = a frozen
   OSD. Fixed: never die() in trace_tid, detach+skip on any error.
Also now checks `unw_get_proc_name()` and falls back to the raw pc, so unwind
failures are distinguishable from symbol failures.
**Verified**: on a 200-thread test target, full symbolized call graphs, exit 0,
0 stranded T threads. Binary: /a/uwpmp/build/unwindpmp (original kept as
unwindpmp.orig); build tree /a/uwpmp/build-fix. Build needs libelf-dev+libdw-dev
— build it inside the ceph-build container (no gcc on the hosts).
Symbols DO still require a build-id-matched -dbg (see [[classic-deb-build-on-70]]),
extracted to ZFS and linked via .gnu_debuglink — see setup_symbols.sh in
/a/agent-scratchpad/profiling/. libunwind resolves ONLY via .gnu_debuglink.
