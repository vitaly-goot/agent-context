---
name: test-cluster-infra
type: reference
---
**8 OSD hosts (Cariboe):** 198.19.34.69/.70/.71/.72, 198.19.32.228/.229/.230/.231.
Each: AMD EPYC 7643 (48c/96t), 5x 1.7 TB NVMe (nvme1n1..nvme5n1), 251 GB RAM, 100 GbE.
**Client/driver:** 198.19.34.68 (single client — fine for 4K, bottlenecks on 4M
bandwidth; more clients needed for 4M).
**Excluded:** .232 (permanent OSD loss — crashed + cold-boot cephx, dropped);
.233 (down); 198.18.140.7 (u22 client candidate, no 10.x cluster-net interface).
**SSH:** `ssh -i /root/.ssh/bits.key root@198.19.x`. Mons: a198-19-34-69,
a198-19-34-70, a198-19-32-228 (mon_host = 10.19.34.69,10.19.34.70,10.19.32.228);
mgr on .69. fio rados engine hangs on crimson -> drive via kernel-mapped RBD +
libaio. .68 reboot wipes the iptables egress rule for the cluster net (re-add).
**Current live cluster:** classic BlueStore, 40 OSD (5/host), pool rep5 size=5,
fsid e953bbfb-caf6-4033-aed3-1a2b393021cd, ~105k-capable, left UP.
