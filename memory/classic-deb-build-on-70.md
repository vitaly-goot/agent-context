---
name: classic-deb-build-on-70
type: reference
---
**Rebuilding symbol-matched classic ceph-osd 20.2.1 debs on .70 (2026-09-10).**
Needed because the running build (08f015eb) has NO -dbg deb reachable (they were
on the dead .69). Driver: extracted the exact commands from
`build-with-container.py -d noble -b build-classic-gcc13-debs --env-file <env>
--ceph-version 20.2.1 -e debs --dry-run` and ran them by hand (to background,
add --network=host, and fix the boost fetch). Same GCC-13/SPDK=ON/IPO/RelWithDebInfo,
--dbg-package on. Artifacts land in build-classic-gcc13-debs/ubuntu/WORKDIR/*.deb
(incl. ceph-osd-dbg, ceph-base-dbg, ceph-common-dbg). Scripts: /a/agent-scratchpad/build/.
Gotchas hit (all real):
- **docker data-root MUST be on ZFS**: root fs is 7.9 GB. /etc/docker/daemon.json
  data-root=/usr/local/akamai/docker; `apt install docker.io` (v29, overlay2 on zfs).
- **image `docker build` needs --network=host** (the script omits it) or apt fails.
- **boost download from download.ceph.com is ~46 KB/s** from this lab (131 MB
  ~48 min). make-dist's download_from tries download.ceph.com THEN archives.boost.io;
  killing the slow wget (`docker exec CID pkill -f wget.*boost`) makes it fall
  through to archives.boost.io (~6 s). liburing/pmdk come from github (fast).
- **Skip make-dist on rebuilds**: make-debs.sh does `test -f ceph-$vers.tar.bz2 ||
  ./make-dist`. Stash the built orig tarball (a198.../agent-scratchpad/build/
  ceph-20.2.1.orig.tar.bz2, 277 MB) and cp it to /ceph/ceph-20.2.1.tar.bz2 before
  each run -> no re-download, straight to dpkg-buildpackage. (make-debs MOVES it
  into WORKDIR, so re-seed every time.) run-debs.sh does this automatically.
- **ccache = LOCAL at /a/ccache** (/usr/local/akamai/ccache, mounted -> /ccache,
  40 G, ccache.conf sets `remote_storage =` so NO redis). Do NOT pass
  `-eCCACHE_REMOTE_ONLY=false`: ccache 4.x rejects "false" as an invalid boolean
  and aborts EVERY compile. Set only CCACHE_DIR/CCACHE_BASEDIR/CCACHE_MAXSIZE env;
  put remote_only/remote_storage in ccache.conf. (The redis at 172.24.237.209:6379
  from ccache.osd.config is unreachable here anyway.)
- Cold cache -> full LTO compile is the multi-hour long pole (dpkg-buildpackage
  -j40 = NPROC/AKCEPH_PACKAGE_NPROC_MAX in env). Future rebuilds warm from /a/ccache.
See [[unwindpmp-profiling]] (why symbols) and [[test-cluster-infra]].
