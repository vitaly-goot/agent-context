---
name: ec-optimizations-ab-result
type: project
---
**Does `ec_optimizations` ("fast EC") help? YES — ~29% on 4k partial-stripe
random writes.** Clean A/B, 2026-09-12: ec43_base vs ec43_opt are identical
pools (erasure profile ec43 k=4 m=3, size 7 min_size 5, stripe_width 16384,
pg_num 512, ec_overwrites) differing ONLY in the `ec_optimizations` flag.
Same run conditions: 4k randwrite, nj4 qd64, 4 co-located clients (the knee,
see [[ec43-knee-and-bottleneck]]).
                       client IOPS   client BW    dev r/s   dev w/s  (per host)
  ec43_opt  (opt ON)      132.5k      518 MiB/s    44,864    87,185
  ec43_base (opt OFF)     102.9k      402 MiB/s    44,025   100,678
  => **+28.8% IOPS / +28.9% BW**
**How it wins: less write amplification, NOT fewer RMW reads.**
Per client 4k write (9 hosts x per-host device I/O / client IOPS):
  total device I/Os : opt **8.97** vs base **12.66**  (-29%)
  device WRITES     : opt **5.92** vs base **8.81**   (-33%)
  device READS      : opt 3.05 vs base 3.85 (-21%); absolute read rate is
                      UNCHANGED (~44k/s per host either way).
So ec_optimizations does not avoid the partial-stripe read-modify-write reads —
it cuts the write/parity traffic per op. Alignment still matters far more:
16 KB full-stripe writes drop device reads 75x and give 3.4x bandwidth
(see [[ec43-knee-and-bottleneck]]).
