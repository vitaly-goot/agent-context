---
name: ec43-load-setup
type: project
---
**"The EC benchmark setup" on the live cluster:** 9 co-located clients = the OSD
hosts, hostidx h0..h8 in osd-id order (.68=h0 .70=h1 .71=h2 .72=h3 .228=h4
.229=h5 .230=h6 .231=h7 .232=h8). Per host two 100 GiB images, all in pool
rbdmeta as the RBD metadata pool with data on the EC pool: b43_h${n}_{0,1}
(ec43_base), o43_h${n}_{0,1} (ec43_opt), i43_h${n}_{0,1} (ec43_opt_isa);
preconditioned. As of 2026-09-10 every host still has its o43 pair mapped
(krbd, exclusive lock held; on .70: o43_h1_0=/dev/rbd0, o43_h1_1=/dev/rbd1).
fio driver = /root/p3_client_fio.sh (args: outdir name rw bs qd nj rt mix devs):
`taskset -c 48-95 fio --filename=/dev/rbd0:/dev/rbd1 --ioengine=libaio --direct=1
--time_based --ramp_time=5 --randrepeat=0 --group_reporting --output-format=json`.
Last run (2026-09-03, /root/loadout/mix4k_w70.json on .70): randrw 4k qd64 nj4
rwmixread=30 (70 % WRITE) 300 s, per-client 6.0k read + 14.0k write IOPS,
lat 8.9 / 14.5 ms. Cluster was fully idle on 2026-09-10 ("nothing is going on").
Fleet orchestration needs ssh (key missing on .70, see [[test-cluster-infra]]).
