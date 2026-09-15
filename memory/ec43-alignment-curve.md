---
name: ec43-alignment-curve
type: project
---
**Stripe-alignment curve for EC 4+3 (ec43_opt, stripe_unit 4096, k=4 -> FULL
STRIPE = 16384 B).** 2026-09-15, 4 co-located clients, nj4 qd64 (the knee),
randwrite. **Recovery + scrub quiesced (nobackfill/norecover/noscrub/nodeep-scrub)
and idle baseline verified at 0 reads/s before measuring** -- an earlier sweep was
discarded because backfill/deep-scrub was injecting ~56,000 reads/s per host at
complete idle, which swamped the RMW signal.
  bs     client IOPS   client BW    dev reads/s   dev writes/s   4K-equiv updates/s
  4k       131.14k      512 MiB/s     43,077        86,265        131.1k  (1.00x)
  8k       119.14k      931 MiB/s     26,934        93,028        238.3k  (1.82x)
  16k      110.69k      1.7 GiB/s          **0**   105,522        442.8k  (3.38x)
  32k       97.05k      3.0 GiB/s          **0**    96,452        776.4k  (5.92x)
  64k       76.26k      4.7 GiB/s          **0**    89,990       1220.2k  (9.31x)
**Device reads hit exactly ZERO at 16 KB and stay there** -- precisely the
predicted k x stripe_unit full-stripe threshold. 8 KB (2 of 4 chunks) still reads,
but ~37% less than 4 KB.
**For application design (can the app batch writes into bigger blocks?):**
break-even vs today's 4k path, i.e. how many logical 4 KB updates must land in
one block for the larger write to pay:
  8 KB block : 131.14/119.14 = **1.10 updates** -> then 1.82x
  16 KB block: 131.14/110.69 = **1.18 updates** -> then 3.38x
The bar is very low: barely more than one coalesced update makes it a win, so
app-side buffering pays off with even weak write locality. But a strict
read-modify-write in the app with NO batching (1 update per block) is a LOSS
(0.84x at 16k). See [[ec43-knee-and-bottleneck]].
