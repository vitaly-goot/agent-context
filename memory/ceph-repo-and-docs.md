---
name: ceph-repo-and-docs
type: project
---
**Repo:** /usr/local/akamai/ceph (= /a/ceph), branch build/toolchain-selection-20.2.2.
**Benchmark docs** (in doc/dev/crimson/, in the toctree):
- seastore-write-path-single-host-benchmark.rst (size=1 OSD/core scaling)
- seastore-write-path-replicated-benchmark.rst (size=5 results + classic baseline)
**Commits:** HEAD d22824b8ef7 (both docs) -> 8d40b23e2dc (crimson boot-fix) ->
437c5d93821. Both top commits re-authored to **Vitaly Goot <vgoot@akamai.com>**,
Claude Co-Authored-By lines stripped; repo git config set to that identity.
**Boot-fix** (crimson multi-OSD boot crash, purged-peer across osdmap batch) is
committed and submitted upstream: https://github.com/ceph/ceph/pull/69972.
History was rewritten (squash + author) -> branch needs **force-push** to
github.com/vitaly-goot/ceph.git. Backup ref: backup-before-author-rewrite = c348ac0402d.
Classic debs: build-classic-gcc13-debs/ubuntu/WORKDIR/ceph-osd_20.2.1-1noble_amd64.deb.
Upstream clone (v21.3.0) for the ceph_abort confirmation: /usr/local/akamai/upstream/ceph.
