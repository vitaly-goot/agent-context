---
name: ec-capacity-efficiency-constraints
type: project
---
**Product/finance require 90% storage efficiency; production can have 22 hosts.
This constrains the EC profile, and the EC profile sets the RMW threshold that
the journal prototype exists to fix.** (Stated 2026-09-17/18.)

## The geometry
efficiency = k/(k+m); 90% means m = k/9. Host-level failure domain => k+m <= hosts.
  m=1 -> 9+1  (10 shards)   m=2 -> 18+2 (20 shards)   m=3 -> 27+3 (30 shards)
**At 22 hosts, 18+2 fits** (20 <= 22): exactly 90% raw with 2-failure tolerance.
27+3 does not. At the old 11 hosts, 90% would have forced m=1 (one failure, zero
redundancy during rebuild) — not shippable.
  profile  shards  raw eff  failures  FULL STRIPE  sellable(est)
   4+3        7     57.1%      3        16 KiB        ~44%
   8+3       11     72.7%      3        32 KiB        ~56%
  12+3       15     80.0%      3        48 KiB        ~62%
  16+3       19     84.2%      3        64 KiB        ~65%
  **18+2     20     90.0%      2        72 KiB        ~69%**
  20+2       22     90.9%      2        80 KiB        ~70%

## raw vs sellable — 90% SELLABLE is arithmetically impossible
sellable = raw x fill-limit x rebuild-headroom x imbalance.
Measured/observed on our cluster: `full_ratio 0.95` / `nearfull 0.85`;
OSD fill spread `MIN/MAX VAR 0.94/1.10` (fullest OSD +10%); rebuild reserve ~1/N.
  77 TiB raw -> x0.571 (4+3) = 44 TiB -> x0.95 fill -> x0.91 imbalance
              -> x0.955 rebuild = **~37 TiB sellable (~48%)**, or ~42% at 85% fill.
Ceph's own `MAX AVAIL` for ec43_opt reads **39 TiB** (it already applies full_ratio
and imbalance). The deduction product is ~0.77-0.83 regardless of profile, so
**even at 100% raw efficiency sellable tops out near 80%.** Widening EC cannot
reach 90% sellable. Only data reduction (compression/dedup) or a commercial
overcommit assumption can. Confirm which definition finance means.

## The double penalty nobody has priced
The T-shirt table encodes a **flat 1,250 IOPS per usable TB** at every size
(500TB->625k, 1000->1.25M, 4000->5M, 8000->10M). So improving space efficiency
RAISES the IOPS requirement proportionally, while EC LOWERS IOPS delivered:
  model      usable   IOPS vs 3R   IOPS per usable TB
  3R          33.3%     1.00        979  = 78% of target (the PoC result)
  EC 4+3      57.1%    ~0.72       ~411  = ~33% of target
  EC 18+2     90.0%    ~0.29 (extrapolated!)  ~105 = ~8% of target
=> **the 90% efficiency requirement and the IOPS/usable-TB requirement are in
severe conflict.** The 18+2 row rests on extrapolating 7->11 shards costing 42%;
20 shards is UNTESTED — directional only.

## The number that matters for the journal
**18+2 makes the full stripe 72 KiB**, 4.5x the 16 KiB of 4+3. Every write below
72 KiB then pays read-modify-write — including the **~15 KB average I/O implied by
the T-shirt requirements themselves** (divide IOPS into throughput: Small
1.25M/150Gb = 15 KB; Medium 5M/600Gb = 15 KB; Large 10M/1200Gb = 15 KB;
X-Small = 20 KB). At 4+3 a 15-16 KB workload is nearly ideally matched to the
16 KiB stripe; at 18+2 it is entirely partial-stripe.
**So: choose 4+3 and alignment may suffice; choose 18+2 and the journal becomes
the thing that makes it viable.** Decide the profile before building.

## Untested wide-profile performance — the obvious next experiment
20-shard EC has never been measured. It CAN be measured now without 22 hosts:
create an 18+2 pool with `crush-failure-domain=osd` (44 OSDs >= 20 shards). Failure
semantics won't match production but shard fan-out, the 72 KiB stripe and the RMW
threshold are faithful. Run the harness ([[measurement-harness]]) against it.
