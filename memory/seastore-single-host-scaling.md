---
name: seastore-single-host-scaling
type: project
---
**Single-host (size=1) 4K random-write IOPS scales on two axes.**
- OSD count: 1/2/4 OSD -> ~9k/31k/47-53k (~12k/OSD). A single OSD caps ~9-10k
  regardless of reactors; scale by adding OSDs, not reactors.
- Reactors/OSD: help up to a knee ~16-32 reactors, then flatten; past 48 physical
  cores throughput drops (looks like SMT contention — inferred, not measured).
Recommendation: 1 OSD per NVMe, ~9 reactors, <=48 cores. Fat OSDs (single 44-reactor
OSD confirmed) also wedge under sustained 4M writes — keep OSDs thin. Documented in
doc/dev/crimson/seastore-write-path-single-host-benchmark.rst.
