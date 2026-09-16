---
name: ec43-large-write-profile
type: project
---
**Large-write (1 MiB) profile of EC 4+3, and it REFUTES my earlier claim that
"ISA-L matters once alignment removes the RMW stall."** Measured 2026-09-16,
osd.35 on .231, 500 probes, offline-symbolized; 4 clients, 1m randwrite qd16 nj4,
ec43_opt, recovery+scrub quiesced.
**Workload contrast (per host / per OSD):**
                      4 KiB            1 MiB
  client           512 MiB/s 131k     **12 GiB/s** 12.6k IOPS
  device reads/s      43,077            **0**  (full stripe, no RMW)
  device write MB/s      904            3,106
  byte amplification    18.4x           **2.17x** (EC 4+3 floor is 1.75x)
  OSD CPU each          ~1140%          **~410%**  (fewer ops -> per-op overhead collapses)
  cluster net          ~5 Gbit/s        ~25 Gbit/s
  op_w latency        10.33 ms          15.28 ms (prepare 0.12 -> **1.49 ms** = encode)
**Profile: the OSD is MORE idle at 12 GiB/s than at 512 MiB/s.**
  parked 91.2% / on-CPU 8.8%   ->   parked **97.2%** / on-CPU **2.8%**
  subsystem        4K on-CPU -> wall | 1M on-CPU -> wall
  BlueStore          23.9%    2.10%  |  17.3%    0.48%
  Buffer/encode      17.4%    1.53%  |  21.0%    0.59%
  OSD op/types       14.5%    1.28%  |   7.2%    0.20%
  RocksDB            12.8%    1.13%  |  12.9%    0.36%
  **EC / erasure      7.8%    0.69%  |  11.8%    0.33%**
  **CRC               2.3%    0.20%  |  14.3%    0.40%**  <- biggest riser
EC math is finally VISIBLE by name at 1M: jerasure's Galois-field routines
`gf_w8_split_multiply_region_sse` 3.9% and `gf_multby_one` 2.7% of on-CPU.
**But EC's share of WALL time FELL, 0.69% -> 0.33%**, because on-CPU collapsed
from 8.8% to 2.8%. So ISA-L is worth <1% end-to-end at large writes too -- even
less than at 4K. My "it matters once aligned" caveat was wrong; the <1% figure
([[ec43-osd-profile-2026-09-11]]) stands for both regimes.
**Caveat:** 12 GiB/s with 4 clients is almost certainly CLIENT-limited (July hit
22-23 GB/s with 10 clients), so the OSD is not being pushed. Even doubling on-CPU
to saturate keeps EC ~0.66% of wall.
