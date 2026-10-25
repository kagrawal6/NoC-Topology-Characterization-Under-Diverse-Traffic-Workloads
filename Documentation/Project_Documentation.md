# Project Documentation

**Network-on-Chip Topology Characterization Under Diverse Traffic Workloads**

Steve Cho and Kushal Agrawal · CMU 18-755 · gem5 / Garnet  
**Status:** Milestone 1 complete (Oct 2026) · Final presentation: early December

This is the main project write-up for anyone reading the repo: what we are doing, how the model works, what we measured, what is left, and what to read next. For build/run commands, see the [README](../README.md).

Work is shared; we split remaining tasks as we go.

---

## Contents

1. [Overview](#1-overview)
2. [Glossary](#2-glossary)
3. [The idea](#3-the-idea)
4. [Research questions](#4-research-questions)
5. [Method](#5-method)
6. [Why routing matters](#6-why-routing-matters)
7. [Milestone 1 results](#7-milestone-1-results)
8. [Where the graph falls short](#8-where-the-graph-falls-short)
9. [Done and left](#9-done-and-left)
10. [Real chips](#10-real-chips)
11. [Stretch goals](#11-stretch-goals)
12. [References](#12-references)
13. [Appendix: gem5 notes](#13-appendix-gem5-notes)

---

## 1. Overview

Modern chips pack many cores onto one die. Those cores talk over a small on-chip packet network — a **Network-on-Chip**. The wiring pattern (**topology**), the path rule (**routing**), and who talks to whom (**traffic**) together decide latency and how much load the network can take before it saturates.

A common shortcut is to judge topologies with graph metrics alone (average hops, betweenness, bisection width). That is incomplete: betweenness assumes traffic spreads evenly over *all* shortest paths, but real chips usually follow one fixed rule such as XY routing.

**This project** builds routing-aware predictions from the graph, checks them in cycle-accurate gem5/Garnet, and uses the comparison to explain when structure is enough — and when it is not (router contention, deadlock). We study Mesh, Torus, Ring, and Tree under gem5’s synthetic traffic patterns.

---

## 2. Glossary

| Term | Meaning |
|---|---|
| **Router / node** | Switch a core attaches to (Tree also has forwarding-only routers). |
| **Link** | Wire between routers; in our setup, one flit per cycle. |
| **Flit** | Fixed-size piece of a packet. Milestone 1 uses 1-flit packets. |
| **Topology** | Wiring shape: Mesh, Ring, Torus, Tree, … |
| **Routing** | Path rule (e.g. XY: finish X, then Y). |
| **Traffic pattern** | Who sends to whom (uniform, transpose, bit-complement, …). |
| **Injection rate (λ)** | Chance a core injects a packet each cycle. |
| **Latency / T₀** | Cycles to deliver a packet; T₀ is latency when the network is nearly empty. |
| **Saturation (Θ_sat)** | Highest injection rate before latency blows up or delivery collapses. Main comparison metric. |
| **Bisection width** | Fewest links to cut the network in half — rough capacity proxy. |
| **Betweenness** | Fraction of shortest paths through a link (assumes even multipath use). |
| **Channel load (γ)** | Expected traffic on a link under a given routing and traffic matrix. |
| **Θ\*** | Ideal saturation from the busiest link: `min(1, 1 / max γ)`. Upper bound. |
| **Virtual channel (VC)** | Separate buffer lane on one wire; also used for deadlock freedom (dateline). |
| **Deadlock** | Circular wait for buffers; nothing progresses. Rings and tori are prone to this without VC classes. |
| **Garnet** | gem5’s cycle-accurate Network-on-Chip model. |

---

## 3. The idea

Think of the chip as a city:

| City | Chip |
|---|---|
| Road map | Topology |
| GPS rule | Routing |
| Destinations | Traffic pattern |
| Gridlock | Saturation |
| Busiest road | Bottleneck link |

Gridlock starts when the **busiest road** fills up. That road depends on the map, the GPS rule, *and* where everyone is going — not on the map alone.

```
same wiring  +  different routing  →  different busiest link  →  different saturation point
```

We compute the busiest link under the **actual** routing rule, predict saturation, run Garnet, and compare.

---

## 4. Research questions

| | Question | After Milestone 1 |
|---|---|---|
| **RQ1** | At low load, does hop count explain latency? | **Yes** — R² = 1.00 |
| **RQ2** | Can we predict saturation from the graph **plus routing**? | **Mostly** — ~5% mean error; fails under deadlock / heavy contention |
| **RQ3** | Which properties explain win/lose per pattern, where does the model fail, and how do real chips relate? | **Open** — uniform order known; full write-up left |

Related follow-ups: match predicted hotspot links to Garnet link utilization; performance per wire / router cost.

---

## 5. Method

### Pipeline

1. **Build** the topology in Garnet (Mesh is built-in; we added Ring, Torus, Tree in `topologies/`).
2. **Analyze** the same wiring in NetworkX: hops, diameter, bisection, betweenness, routing-aware channel loads.
3. **Predict** zero-load latency from hops, and saturation from the busiest link.
4. **Simulate** topology × traffic pattern × injection rate in gem5.
5. **Compare** prediction to measurement; explain gaps; connect to commercial designs.

### Topologies (16 cores)

| | Mesh | Torus | Ring | Tree |
|---|---:|---:|---:|---:|
| Picture | 4×4 grid | Grid + wrap-arounds | Single cycle | Hierarchy; cores at leaves |
| Routers / links | 16 / 24 | 16 / 32 | 16 / 16 | 31 / 30 |
| Diameter | 6 | 4 | 8 | 8 |
| Avg hops | 2.67 | 2.13 | 4.27 | 6.53 |
| Bisection | 4 | 8 | 2 | 1 |
| Status | Built-in | Ours (watch deadlock) | Ours (**needs dateline**) | Ours |

Industry flavor: Mesh ≈ many CPU meshes; Torus ≈ TPU-style pods; Ring ≈ older Intel rings; Tree ≈ simplified hierarchical / fat-tree idea.

### Routing

| Topology | Rule |
|---|---|
| Mesh, Torus | XY (dimension-order): finish X, then Y |
| Ring | Shorter arc; random on ties |
| Tree | Only path: up to the shared ancestor, then down |

### Traffic

Milestone 1 used all eight Garnet patterns: `uniform_random`, `transpose`, `bit_complement`, `bit_reverse`, `bit_rotation`, `shuffle`, `tornado`, `neighbor`.

At 16 cores, tornado collapses to the same destinations as neighbor (identical results). They diverge at larger N.

Still interesting to add later: hotspot, GPU-style many-to-few, AI collectives, real traces.

### Assumptions

- Independent synthetic injection; 1-flit packets  
- Every link = 1 cycle (no wire-length penalty)  
- Identical routers (1-cycle delay, 4 VCs)  
- Same source→destination mapping across topologies for a given pattern  
- Tester and network clocks forced to 1 GHz  

### Models

**Low load (RQ1)** — from Dally & Towles:

```
T0  =  H_avg × (t_router + t_wire)  +  T_fixed
```

- `H_avg` — average hops under the routing and traffic in use  
- `t_router`, `t_wire` — per-hop delays (we use 1 + 1 = 2 cycles)  
- `T_fixed` — cost of entering and leaving the network  

Milestone 1 fit across all 32 topology×pattern points:

```
T0  =  2.01 × H_avg  +  4.98     (R² = 1.000)
```

**High load (RQ2)** — busiest-link bound:

1. Build traffic matrix `T` (who sends to whom).  
2. For each source→destination, follow the **real** routing path.  
3. Add that traffic onto every link on the path → channel loads `γ`.  
4. Ideal saturation:

```
Θ*  =  min( 1 ,  1 / max(γ) )
```

Capped at 1 because a core cannot inject more than one packet per cycle.

**Worked examples (16 cores, bit-complement):**

- **Tree:** every left→right packet shares the root cut → 8 packets on one link → Θ\* = 1/8 = **0.125**. gem5 matched.  
- **Mesh:** traffic can use 4 mid-chip cuts → Θ\* = **0.50**. gem5 matched.

### How we mark saturation in simulation

First injection rate where latency > **3 × T₀**, or fewer than **95%** of offered packets arrive. Sweep λ from 0.025 to 0.800 in steps of 0.025.

---

## 6. Why routing matters

Before the full gem5 sweeps, `analysis/prelim_channel_load.py` compared routing rules on the **same** 8×8 mesh graph:

| Routing on the same mesh | Uniform random | Bit-complement |
|---|---:|---:|
| XY (what chips use) | **0.50** | **0.25** |
| Random shortest-path choice each hop | 0.40 | 0.13 |
| Even split over all shortest paths (**betweenness**) | 0.37 | 0.11 |

Ideal throughput in packets/core/cycle (higher is better). Ignoring routing mispredicts by about **25%** (uniform) to **2.4×** (bit-complement). That is the core motivation for a routing-aware model.

---

## 7. Milestone 1 results

**1,024 Garnet runs:** 4 topologies × 8 patterns × 32 injection rates (16 cores).  
Figures: [`figures/networks.png`](../figures/networks.png), [`figures/prelim_results.png`](../figures/prelim_results.png).  
Data: `results/prelim_fine.csv`, `results/saturation_summary.csv`.

### RQ1 — low load

Hop count fully predicts latency. All 32 topology×pattern points lie on one line (R² = 1.000). Slope ≈ 2 matches one-cycle router + one-cycle link. Shorter-path topologies are always faster when the network is empty: **Torus best, Tree worst**.

### RQ2 — high load

Predicted vs simulated saturation (packets/core/cycle). Higher is better. Format: **predicted / simulated**.

| | Uniform | Transpose | Bit-compl. | Bit-rev. | Bit-rot. | Shuffle | Neighbor‡ |
|---|---|---|---|---|---|---|---|
| **Mesh** | 1.00 / 0.63 | 0.33 / 0.35 | 0.50 / 0.50 | 0.33 / 0.35 | 0.50 / 0.50 | 0.50 / 0.50 | 1.00 / 0.80 |
| **Torus** | 1.00 / 0.75 | 0.67 / 0.60 | 1.00 / 0.80 | 0.67 / 0.55 | 0.67 / 0.63 | 0.67 / 0.65 | 1.00 / 0.80 |
| **Ring** | 0.50 / 0.28† | 0.33 / 0.30 | 0.25 / 0.25 | 0.33 / 0.30 | 0.25 / 0.25 | 0.25 / 0.25 | 1.00 / 0.80 |
| **Tree** | 0.25 / 0.23 | 0.25 / 0.23 | 0.13 / 0.13 | 0.25 / 0.23 | 0.25 / 0.23 | 0.25 / 0.25 | 1.00 / 0.80 |

† Deadlock (not true capacity). ‡ Tornado ≡ neighbor at N = 16.

Mean absolute error ≈ **5.3%** when the network is deadlock-free. Exact hits on Tree and Mesh bit-complement.

### Uniform traffic order (simulated)

```
Tree 0.225  <  Ring 0.275†  <  Mesh 0.625  <  Torus 0.75
```

Same order as bisection width (1, 2, 4, 8). Winner still depends on traffic: under neighbor traffic Mesh ≈ Torus; under bit-complement Torus handles about **1.6×** more load than Mesh.

### Takeaways

1. Light traffic → hops decide.  
2. Heavy traffic → busiest link / bisection decide; the pattern picks the winner.  
3. The graph is strong but incomplete — see next section.  
4. Next: fix Ring/Torus deadlock, finish RQ3, map to real chips.

---

## 8. Where the graph falls short

| Effect | Why the topology graph misses it | Evidence |
|---|---|---|
| **Router / VC contention** | Crossbar, buffers, and arbitration are not in the graph | Mesh/Torus uniform: Θ\* = 1.0 but gem5 ≈ 0.63 / 0.75 |
| **Deadlock** | Circular buffer dependence needs VC classes / turn rules | Ring uniform freezes at 0.275 (predicted 0.50) |
| **Unequal wire delay** | We force every link to 1 cycle | Real wrap-arounds and root links are longer |
| **Multi-flit / protocols** | We use 1-flit synthetic traffic | Real coherence has asymmetric request/response |

These gaps are part of the story for RQ3, not just bugs to hide.

---

## 9. Done and left

### Done

- gem5 / Garnet running (including macOS `clang++-sdk` workaround)
- Ring, Torus, Tree topologies + sweep/analysis scripts
- 1,024-run Milestone 1 campaign and comparison to the routing-aware model
- Milestone 0 and Milestone 1 reports under `Documentation/`

### Left (toward early December)

| Priority | Item |
|---|---|
| **Must** | Dateline / VC classes for Ring (and Torus if needed); re-run those cases |
| **Must** | RQ3: explain win/lose per pattern; map findings to real chips |
| **Should** | Scale key cells to 64 cores; cost model (perf per link / wire) |
| **Should** | New traffic: hotspot, many-to-few, collectives |
| **Nice** | Warm-up in sweep; per-link γ vs utilization heatmaps; O1TURN / Valiant |
| **Stretch** | See [§11](#11-stretch-goals) |

### Timeline

| When | Focus |
|---|---|
| Now – early Nov | Deadlock fix; re-sweep Ring/Torus |
| Mid Nov | RQ3 + real-chip case study |
| Late Nov | 64-core / cost / new traffic as time allows; draft final slides |
| Early Dec | Final presentation and report |

**Fallbacks:** if deadlock slips, keep reporting Ring below saturation and flag it (as in Milestone 1). If 64-core is too slow, keep 16-core for extras.

---

## 10. Real chips

Sketches for the RQ3 case study (not claims that our 16-core synthetic setup equals these products):

| Design | Topology flavor | Angle relative to us |
|---|---|---|
| Older Intel rings → Xeon meshes | Ring then mesh | Ring cheap at small N; bisection stays 2; scales poorly — matches our Ring vs Mesh gap |
| Knights Landing / CPU meshes | 2D mesh + XY | Our Mesh baseline |
| AMD Infinity Fabric / chiplets | Hierarchical, expensive cross-die links | Motivation for heterogeneous link cost (stretch) |
| NVIDIA GPUs | Many cores → few memory controllers | Motivation for many-to-few traffic (Bakhoda) |
| Google TPU pods | Torus-like interconnect | Wrap-arounds help long-haul traffic (our bit-complement) |

For each case: what topology, what traffic they care about, which of our metrics would have pointed there, and what silicon still handles that our model ignores (power, wire delay, coherence).

---

## 11. Stretch goals

Nothing below is committed. Roughly ordered by usefulness if time appears.

**Near-term polish**

- Link-utilization heatmaps (predicted γ vs Garnet stats)  
- Warm-up / drain in the sweep  
- Cost tables: Θ_sat per link or per router  

**Topology and routing**

- Flattened Butterfly; fat-tree / Kite  
- O1TURN, Valiant, adaptive routing (RCA / odd-even)  
- Search for extra links under a wire budget using the fast Θ\* model  

**Traffic and workloads**

- Hotspot; GPU many-to-few; all-reduce schedules  
- Netrace or full-system Ruby; optional AI-sim coupling  

**Deeper modeling**

- Multi-flit packets; wire delay ∝ length  
- VC / buffer sweeps; simple queueing correction on top of γ  
- Learned predictor from graph features (only if we actually build it)  

**Artifacts**

- Upstream gem5 PRs for Ring/Torus/Tree  
- BookSim cross-check; optional RTL micro-router  

**Open questions worth revisiting**

- Is the Mesh/Torus uniform gap (0.63–0.75 vs Θ\* = 1) a stable contention factor across N?  
- Does dateline *placement* change measured Θ_sat, or only whether deadlock appears?  
- Can one figure show “when betweenness lies” across every (topology, pattern)?

---

## 12. References

Scholar links are search URLs (stable). Split into what we already rely on vs what we may use later.

### Already referenced / used in this project

These appear in our Milestone write-ups, models, or tooling.

| Work | Why we use it |
|---|---|
| [Dally & Towles (2004) — *Principles and Practices of Interconnection Networks*](https://scholar.google.com/scholar?q=Principles+and+Practices+of+Interconnection+Networks+Dally+Towles) | Latency and busiest-link throughput (Ch. 3); topology basics |
| [Enright Jerger, Krishna & Peh (2017) — *On-Chip Networks*, 2nd ed.](https://scholar.google.com/scholar?q=On-Chip+Networks+Enright+Jerger+Krishna+Peh) | Modern Network-on-Chip background; DOR; dateline VCs |
| [Dally & Seitz (1987) — Deadlock-Free Message Routing](https://scholar.google.com/scholar?q=Deadlock-Free+Message+Routing+in+Multiprocessor+Interconnection+Networks) | Basis for dateline / VC deadlock freedom |
| [Agarwal et al. (2009) — GARNET](https://scholar.google.com/scholar?q=GARNET+detailed+on-chip+network+model+full-system+simulator) | Our Network-on-Chip simulator model |
| [Binkert et al. (2011) — The gem5 Simulator](https://scholar.google.com/scholar?q=%22The+gem5+simulator%22+Binkert) | Simulation host |
| [Hagberg, Schult & Swart (2008) — NetworkX](https://scholar.google.com/scholar?q=Exploring+Network+Structure+Dynamics+and+Function+using+NetworkX) | Graph analysis library |
| [Bakhoda, Kim & Aamodt (2010) — Throughput-Effective On-Chip Networks for Manycore Accelerators](https://scholar.google.com/scholar?q=Throughput-Effective+On-Chip+Networks+for+Manycore+Accelerators) | GPU / many-to-few motivation (previous work) |
| [Pande et al. (2005) — Performance evaluation and design trade-offs for NoC interconnect architectures](https://scholar.google.com/scholar?q=Performance+evaluation+and+design+trade-offs+for+network-on-chip+interconnect+architectures) | Classic topology × traffic comparison (previous work) |
| [gem5 Garnet synthetic traffic docs](https://www.gem5.org/documentation/general_docs/ruby/garnet_synthetic_traffic/) | Patterns, flags, stats |

LaTeX cite keys in Milestone 1: `b1` Bakhoda · `b2` Pande · `b3` gem5 · `b4` GARNET · `b5` Dally & Towles book · `b6` Dally & Seitz · `b7` NetworkX · `ocn` Enright Jerger et al.

### Potential references

Useful if we extend the project; not cited as core evidence yet.

| Topic | Candidates |
|---|---|
| **History / vision** | Dally & Towles (2001) *Route Packets, Not Wires*; Benini & De Micheli (2002) |
| **Topology** | Balfour & Dally (2006); Leiserson fat-trees (1985); Kim et al. Flattened Butterfly (2007); Ogras & Marculescu (2006); Bharadwaj et al. Kite (2020); Dragonfly (Kim et al. 2008) |
| **Routing / VCs / adaptive** | Dally (1992) virtual channels; Glass & Ni turn model (1992); Duato adaptive deadlock theory; Seo et al. O1TURN (2005); Valiant & Brebner (1981); Gratz et al. RCA (2008) |
| **Industry** | Hoskote et al. Intel 80-core (2007); Sodani et al. KNL (2016); Naffziger et al. AMD chiplets (2021); Arunkumar et al. MCM-GPU (2017); Jouppi et al. TPU papers; Tilera / Wentzlaff |
| **Simulators / traces** | Lowe-Power et al. modern gem5; Jiang et al. BookSim 2 (2013); Netrace; Verilator / Yosys / OpenROAD for an RTL stretch |
| **Optional depth** | SMART NoC; CHIPPER (bufferless); router microarchitecture papers; chiplet NoC workshops (NOCS / HPCA tutorials) |

---

## 13. Appendix: gem5 notes

### Baseline settings (Milestone 1)

| Setting | Value |
|---|---|
| Cores | 16 (4×4); 64 planned |
| VCs | 4 |
| Router / link delay | 1 cycle |
| Link width | 128 bits |
| Packet size | 1 flit |
| Injection sweep | 0.025 … 0.800 |
| Saturation rule | latency > 3×T₀ or delivery < 95% |

### Useful `stats.txt` fields

| Stat | Meaning |
|---|---|
| `average_packet_latency` | End-to-end cycles |
| `average_hops` | Measured hop count |
| `packets_injected` / `packets_received` | Delivery check |
| `int_link_utilization` | Per-link use (for validating γ) |

Prefer `scripts/sweep.py` over hand-built commands — it symlinks `topologies/`, forces 1 GHz clocks, and converts `--sim-cycles` correctly (gem5 treats that argument in picoseconds).

### Gotchas we hit

| Finding | What we do |
|---|---|
| No Ring/Torus/Tree upstream | Implemented under `topologies/` |
| `outportComputeCustom()` is a stub | Hook for dateline / custom routing |
| VC = first free buffer | Need VC **classes** for deadlock freedom |
| Patterns expect square power-of-two N | Use 16 or 64 |
| Default clocks: network 2 GHz, tester 1 GHz | Force both to 1 GHz |
| Tree layout | `--mesh-rows=0` |
| tornado ≡ neighbor at N = 16 | Document; diverges at 64 |
| No warm-up yet | Still to add |

### Dateline fix (next implementation sketch)

1. Choose a dateline on the ring (or torus dimension).  
2. Packets that cross it move to a higher VC class.  
3. Keep the class order acyclic (Dally & Seitz / *On-Chip Networks*).  
4. Re-sweep Ring (and Torus); confirm delivery past the old freeze point.  
5. Compare new Θ_sat to Θ\*.

### Repo map

```
topologies/          Ring.py, Torus.py, Tree.py
scripts/sweep.py     parallel Garnet sweeps → results/
analysis/            channel-load model + plots
figures/             networks.png, prelim_results.png
results/             prelim_fine.csv, saturation_summary.csv, runs/
Documentation/       this file, milestones
clang++-sdk          macOS build workaround
gem5/                submodule
```
