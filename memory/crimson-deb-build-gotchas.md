---
name: crimson-deb-build-gotchas
type: reference
---
**build-with-container.py (ceph 20.2.1 debs) gotchas.**
- `WITH_CRIMSON=0` is TRUTHY (`bool("0")==True`) -> a CLASSIC build needs it
  ABSENT from the env-file, not `=0`. Crimson build = env-file WITH_CRIMSON=1.
- Image `docker build` step lacks `--network=host` (only `docker run`/make-debs
  gets `--extra=--network=host`), so apt fails ("Unable to locate package") ->
  pre-build/re-label the build image, then -e debs reuses it.
- `--ceph-version 20.2.1` required (else `dch: -1 is not a valid version`).
- Verify classic: ceph-osd ~26 MB, 0 seastar symbols, 8787 BlueStore/rocksdb
  strings. Crimson: crimson-osd ~43 MB, seastar reactor symbols present.
- Separate build dirs: build-crimson-gcc13-debs, build-classic-gcc13-debs.
- ceph-volume needs clean devices — remove stale unmounted LVM (e.g.
  nsds_volume_group off nvme1-5; OS root=nvme0n1, /usr/local/akamai=ZFS tank, safe).
