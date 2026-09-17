---
name: ec43-large-write-knee
type: project
---
**Large (1 MiB) writes on EC 4+3 ARE device-bound — evidenced by queue depth, not
by a guessed datasheet number.** Measured 2026-09-17, 4 clients, 1m randwrite on
ec43_opt, recovery+scrub quiesced, .231 instrumented.
  point      outst   GiB/s  devMB/s  MB/s/drive  aqu-sz   marginal
  qd8  nj2      64     8.5    2014      403       1.09
  qd16 nj2     128    12.0    2575      515       2.07   x2 -> +41% tput
  qd16 nj4     256    15.0    2991      598       5.89   x2 -> +25% tput
  qd32 nj4     512    15.0    3048      610       6.40   x2 -> **+0%**
  qd64 nj4    1024    17.0    3365      673      19.41   x2 -> +13%, x3.0 aqu
  qd64 nj8    2048    18.0    3769      754      68.93   x2 -> +6%,  x3.6 aqu
**Queue depth rises 63x (1.09 -> 68.93) while throughput rises 2.1x** — textbook
device saturation. Ceiling ~18 GiB/s cluster / **~754 MB/s per NVMe**. Efficient
operating point is **qd16 nj4 (256 outstanding, 15 GiB/s, aqu 5.9)**; past it you
pay 3-4x queue depth for 6-13% throughput.
NOT network (32.1 of 100 Gbit/s = 32%) and NOT CPU (418% of 9600% = 4.4%).
`%util` was 95-99% at EVERY point including aqu=1.09 — useless for NVMe, use aqu-sz.
**This re-establishes a claim I had retracted.** I first said "device-bound" from
bad arithmetic (mixed 8+3 throughput with 4+3 amplification, plus an assumed
Micron spec) and withdrew it. The queue-depth evidence supports the same
conclusion without needing any datasheet. Also made draining an OSD for a raw
device benchmark unnecessary.
**Key inversion vs small writes:** at 4 KiB the devices were IDLE (aqu ~2) so
reducing write amplification bought nothing. At 1 MiB the devices are SATURATED,
so the ~21% excess amplification (2.12x measured vs 1.75x EC 4+3 floor, see
[[ec43-large-write-profile]]) now converts almost 1:1 into client throughput.
That makes amplification the one software lever left for large writes; everything
else (ISA-L, ec_optimizations, alignment, more cores) is already spent.
