# Project Documentation

**Team:** Steve Cho (`stevecho`) and Kushal Agrawal (`kushalag`)  
**Course:** 18-755 (CMU) — Network-On-Chip project  
**Tool:** gem5 simulator + Garnet NoC model  
**Full title:** *From Graph Structure to Saturation: Predicting Network-on-Chip Performance from Topology, Routing, and Traffic*  
**Status:** Milestone 1 complete (Oct 2026). Final presentation: first week of December.  
**Purpose of this file:** Primary shared resource for ideas, understanding, formulas, results, stretch goals, references, and “what’s next.” Prefer updating this over scattering notes elsewhere.

Work is shared; we split remaining tasks as we go (no fixed person ownership in this doc).

---

## Table of contents

1. [One-paragraph pitch](#1-one-paragraph-pitch)
2. [Key terms](#2-key-terms-glossary)
3. [The big idea (city analogy)](#3-the-big-idea-city-analogy)
4. [Conceptual foundations](#4-conceptual-foundations)
5. [Research questions](#5-research-questions)
6. [Progress: done / left](#6-progress-done--left)
7. [Early analytical result (routing vs betweenness)](#7-early-analytical-result-routing-vs-betweenness)
8. [Approach in detail](#8-approach-in-detail)
9. [Milestone 1 results (keep handy)](#9-milestone-1-results-keep-handy)
10. [Real chips: case-study ideas](#10-real-chips-case-study-ideas)
11. [Timeline (remaining)](#11-timeline-remaining)
12. [Stretch goals (expansive)](#12-stretch-goals-expansive)
13. [Open ideas & brainstorming](#13-open-ideas--brainstorming)
14. [How to present / résumé](#14-how-to-present--résumé)
15. [Reading list & reference library](#15-reading-list--reference-library)
16. [Appendix A: Formulas & worked examples](#appendix-a-formulas--worked-examples)
17. [Appendix B: gem5 / Garnet cheat sheet](#appendix-b-gem5--garnet-cheat-sheet)
18. [Appendix C: Repo layout & scripts](#appendix-c-repo-layout--scripts)
19. [Appendix D: Interview talking points](#appendix-d-interview-talking-points)

---

## 1. One-paragraph pitch

Modern chips (CPUs, GPUs, AI accelerators) contain dozens of cores that constantly send data to each other over a tiny on-chip network called a **Network-on-Chip (NoC)**. How you wire the cores together (the **topology**) strongly affects how fast the chip runs. People usually compare topologies using simple graph numbers such as “how many hops between the farthest two cores.” **We show that those numbers are not enough.** You also need to know **which path each message takes (routing)** and **who talks to whom (traffic)**. We build predictions from the graph *plus* the routing, check them against a cycle-by-cycle simulator (gem5/Garnet), and use the results to explain real design choices made by Intel, AMD, NVIDIA, and Google.

**30-second version:**
> We studied why some on-chip networks outperform others. Standard graph metrics can be wrong by more than 2× if they ignore routing. We built a routing-aware model, validated it against gem5 within about 5%, extended gem5 with Ring/Torus/Tree, and found where the graph fails — especially Ring deadlock.

---

## 2. Key terms (glossary)

| Term | Plain-English meaning |
|---|---|
| **NoC** | On-chip packet network connecting cores, caches, memory controllers, accelerators. |
| **Node / router** | A stop on the network. Each core plugs into a router (Tree also has forwarding-only routers). |
| **Link / channel** | A wire between two routers. Capacity is typically one **flit** per cycle in our setup. |
| **Flit** | Flow-control unit: a fixed-size piece of a packet. Our Milestone 1 packets are **1 flit**. |
| **Topology** | The wiring pattern: Mesh, Ring, Torus, Tree, Flattened Butterfly, Crossbar, … |
| **Routing** | Rule that picks a path. **XY / DOR** = go X first, then Y. Shortest-path Ring = shorter arc. |
| **Deterministic vs adaptive** | Deterministic always picks the same path for (src,dst); adaptive can change with congestion. |
| **Traffic pattern** | Who sends to whom. *Uniform random*, *transpose*, *bit-complement*, *neighbor*, … |
| **Injection rate λ** | Probability a core injects a packet each cycle (e.g. 0.1 ≈ once every 10 cycles). |
| **Hop count** | Number of links a packet crosses. |
| **Latency** | Cycles from injection to reception. **T₀** = zero-load (nearly empty network) latency. |
| **Throughput** | Packets delivered per core per cycle. Often ≈ min(λ, saturation) until the network clogs. |
| **Saturation throughput Θ_sat** | Highest injection rate before latency explodes / delivery collapses. **Main comparison metric.** |
| **Bisection width** | Min number of links you must cut to split the network in half. Rough capacity proxy. |
| **Betweenness centrality** | Graph score: fraction of shortest paths through a node/edge. High = likely bottleneck *if* traffic uses all shortest paths equally. |
| **Channel load γ_ℓ** | Expected traffic through link ℓ under a given routing + traffic matrix (at λ = 1). |
| **Θ\*** | Ideal saturation from busiest link: `min(1, 1 / max γ_ℓ)`. Upper bound; contention lowers real Θ_sat. |
| **Virtual channel (VC)** | Separate buffer lane on the same physical link. Avoids head-of-line blocking; enables **dateline** deadlock freedom. |
| **Deadlock** | Circular wait for buffers; no packet progresses. Rings/tori are classic cases without VC classes. |
| **Livelock** | Packets keep moving but never reach destination (less common with deterministic shortest-path routing). |
| **Garnet** | gem5’s detailed cycle-accurate NoC model (routers, VCs, links, synthetic traffic tester). |
| **Ruby** | gem5 cache-coherence / memory system that can sit above Garnet (full-system path; stretch). |

---

## 3. The big idea (city analogy)

Think of the chip as a **city**:

| City | Chip |
|---|---|
| Road map | Topology |
| GPS rule every driver follows | Routing |
| Where everyone is going | Traffic pattern |
| Gridlock | Saturation |
| Busiest road | Bottleneck link (max γ_ℓ) |

The city gridlocks when its **single busiest road** fills up. To predict gridlock you need the map, the GPS rule, *and* the destinations.

Standard graph analysis (**betweenness**) quietly assumes *drivers spread evenly over every shortest route*. Real chips do not. A mesh with XY routing sends everyone “horizontal first,” which piles traffic onto different roads than “pick a random shortest path.”

```
Same road map  +  different GPS rule  =  different busiest road  =  different gridlock point
```

**Our project:** compute the busiest link using the *real* routing rule, predict when the network saturates, check that in gem5, then explain wins/losses and connect to commercial chips.

---

## 4. Conceptual foundations

### 4.1 Two regimes

| Regime | What dominates | Graph proxy | Our RQ |
|---|---|---|---|
| **Low load** (λ → 0) | Path length + fixed NI cost | Average hops H_avg | RQ1 |
| **High load** (near saturation) | Waiting on the hottest link (+ router internals) | Busiest-link load max γ_ℓ → Θ\* | RQ2 |
| **Across both** | Why A beats B under pattern P; when graph fails | Hops, bisection, betweenness, γ maps; deadlock | RQ3 |

### 4.2 Zero-load latency (RQ1)

From Dally & Towles (2004), Ch. 3:

\[
T_0 = H_{\mathrm{avg}}\,(t_r + t_w) + T_{\mathrm{fixed}}
\]

- \(H_{\mathrm{avg}}\): mean hops under the routing + traffic of interest  
- \(t_r, t_w\): per-hop router and wire delay (we use 1 + 1 = 2 cycles)  
- \(T_{\mathrm{fixed}}\): enter/leave network (NI / serialization overhead)

**Milestone 1 fit (all 32 topology×pattern points):**  
\(T_0 = 2.01\,H_{\mathrm{avg}} + 4.98\) with \(R^2 = 1.000\).

### 4.3 Busiest-link saturation (RQ2)

1. Build the traffic matrix \(T\) (who sends to whom; row-stochastic).  
2. For every source–destination with \(T_{sd} > 0\), follow the **actual routing path**.  
3. Add \(T_{sd}\) to every link on that path → channel loads \(\gamma_\ell\).  
4. Ideal throughput:

\[
\Theta^{*} = \min\!\left(1,\;\frac{1}{\max_\ell \gamma_\ell}\right)
\]

Cap at 1 because a core cannot inject more than one packet per cycle in our model.

**Worked Tree + bit-complement (16 cores):** every left→right packet climbs through the single root cut → 8 packets share that link → \(\Theta^{*} = 1/8 = 0.125\). gem5 matched **exactly**.

**Worked Mesh + bit-complement:** same traffic can use 4 vertical mid-cuts → ~2 packets/link → \(\Theta^{*} = 0.5\). gem5 matched **exactly**.

### 4.4 Why betweenness alone is not enough

Betweenness ≈ “fraction of *all* shortest paths through this edge,” which is like load under **uniform multipath** routing. Real chips often use **one** deterministic path (XY). On the same 8×8 mesh graph, changing only the routing rule changes ideal Θ\* a lot (see §7). That gap is the intellectual core of the project.

### 4.5 Where the graph *cannot* win

| Effect | Why graph misses it | Our evidence |
|---|---|---|
| **Router / VC contention** | Crossbar, buffers, arbitration not in the topology graph | Mesh/Torus uniform: Θ\*=1.0 but gem5 ~0.63 / 0.75 |
| **Deadlock** | Circular buffer dependence; needs VC *classes* / turn rules | Ring uniform freezes at 0.275 (pred. 0.50) |
| **Unequal wire delay** | We force every link = 1 cycle | Real Torus wrap / Tree root wires are longer |
| **Multi-flit / protocol** | We use 1-flit synthetic | Real coherence has request/response asymmetry |

Failures are features for RQ3 and for interviews: “here is what structure *doesn’t* capture.”

---

## 5. Research questions

| # | Question | Status after Milestone 1 |
|---|---|---|
| **RQ1** | At low traffic, is latency explained by hop count? | **Yes** — \(R^2 = 1.00\), slope ≈ 2 |
| **RQ2** | Can we predict saturation from graph **+ routing**? | **Mostly** — mean error ~5.3%; fails on deadlock / contention |
| **RQ3** | Which properties explain win/lose per pattern; where does the model fail; how do real chips relate? | **Open** — order known for uniform; full write-up + case study left |
| **Q3′** | Do predicted *busiest links* match simulated link utilization? | Partially; per-link validation still open |
| **Q5** | Best performance **per wire / per router cost**? | Not started |

---

## 6. Progress: done / left

### Done (through Milestone 1)

- gem5 / Garnet built and running (including Mac `clang++-sdk` workaround)
- **Ring, Torus, Tree** topologies implemented (gem5 ships Mesh only)
- Sweep + analysis: `scripts/sweep.py`, `analysis/prelim_channel_load.py`, `plot_networks.py`, `prelim_results.py`
- **1,024 simulations:** 4 topologies × 8 traffic patterns × 32 injection rates (16 cores)
- Graph vs gem5:
  - Low load: \(T_0 = 2.01 H + 4.98\), \(R^2 = 1.000\)
  - High load: busiest-link Θ\* within **~5.3%** of measured saturation (excl. deadlock pathology)
  - Gaps: router contention; **Ring deadlock** at 0.275 (predicted 0.50)
- Hop sanity check: gem5 avg hops vs NetworkX table within ~1%
- Milestone 0 + Milestone 1 report/slides completed (`Documentation/`)

### Left (toward final, early December)

| Priority | Item | Notes |
|---|---|---|
| **Must** | Dateline / VC classes for Ring (and Torus if needed); re-run affected sweeps | Hook: Garnet `outportComputeCustom` + VC allocation |
| **Must** | RQ3: explain win/lose per pattern with graph metrics; map to real chips | Figures + narrative for final slides |
| **Should** | Scale key experiments to **64 cores** | Same patterns; watch sim time |
| **Should** | Cost model: Θ_sat / #links, latency × wire, area proxy | “Best per dollar of silicon” |
| **Should** | New traffic: hotspot, many-to-few (GPU), ring all-reduce (AI) | Ties to Bakhoda / training collectives |
| **Nice** | Warm-up cycles in sweep; tornado≠neighbor at larger N; O1TURN / Valiant | Cleaner methodology |
| **Nice** | Per-link utilization vs predicted γ_ℓ heatmaps | Strengthens RQ2 story |
| **Stretch** | See [§12](#12-stretch-goals-expansive) |

---

## 7. Early analytical result (routing vs betweenness)

Before full gem5 sweeps, `analysis/prelim_channel_load.py` showed that on the **same** 8×8 mesh, changing only the routing rule changes ideal throughput:

| Routing rule on the same 8×8 mesh | Uniform random | Bit-complement |
|---|---:|---:|
| XY routing (what real chips use) | **0.50** | **0.25** |
| Random choice at every hop (table-style) | 0.40 | 0.13 |
| Spread evenly over all shortest paths (**betweenness assumption**) | 0.37 | 0.11 |

*(Packets per core per cycle. Higher is better.)*

Plain betweenness can be wrong by **~25%** (uniform) to **~2.4×** (bit-complement) if it ignores routing. That motivated the routing-aware model validated in Milestone 1.

---

## 8. Approach in detail

### 8.1 Pipeline

1. **Build** topologies in gem5/Garnet (Mesh built-in; we added Ring, Torus, Tree under `topologies/`).
2. **Analyze** the same wiring as a NetworkX graph: hops, diameter, degree, bisection, betweenness, routing-aware γ_ℓ.
3. **Predict:** \(T_0\) from hops; \(\Theta^{*}\) from busiest link.
4. **Simulate** every topology × pattern × λ; record latency, hops, delivery, (later) link stats.
5. **Compare** prediction vs simulation; explain; connect to commercial NoCs.

### 8.2 Topologies

| Topology | Picture | Graph (16-core) | Real-world flavor | Status |
|---|---|---|---|---|
| **Mesh** | 4×4 grid | 16 routers, 24 links, diam 6, avg hops ~2.67, bisection 4 | Intel Xeon / many CPU meshes; Tilera | Done (built-in) |
| **Torus** | Mesh + wrap-arounds | 16 r, 32 links, diam 4, avg hops ~2.13, bisection 8 | Google TPU pods (system-level torus); research chips | Done (ours); watch deadlock |
| **Ring** | Single cycle | 16 r, 16 links, diam 8, avg hops ~4.27, bisection 2 | Older Intel ring interconnects | Done (ours); **needs dateline** |
| **Tree** | Binary hierarchy; cores at leaves | 31 routers (16+15), 30 links, diam 8, avg hops ~6.53, bisection 1 | Fat-tree / hierarchical AI fabrics (simplified) | Done (ours) |
| Flattened Butterfly | High-radix row/col | — | Research high-radix NoCs | Stretch |
| Crossbar | Fully connected | — | Ideal / small clusters | Optional reference |
| Dragonfly / Kite / custom | — | — | HPC / chiplet fabrics | Stretch |

**Tree note:** gem5 needs `--mesh-rows=0` (or equivalent) so it does not assume a mesh layout.

### 8.3 Routing (what we use today)

| Topology | Rule |
|---|---|
| Mesh / Torus | XY (dimension-order): finish X, then Y |
| Ring | Shorter arc; ties broken randomly |
| Tree | Unique path: up to LCA, then down |

All are **shortest-path** and mostly **deterministic** (random only on ties). Stretch: O1TURN, Valiant, adaptive (RCA, etc.).

### 8.4 Traffic patterns

**Built-in (done in Milestone 1):**

| Pattern | Idea | Stresses |
|---|---|---|
| `uniform_random` | Any → any | Average-case; bisection matters |
| `transpose` | (x,y) → (y,x) | Diagonal / permutation hot spots |
| `bit_complement` | Opposite corner of chip | Long-haul; mid-cuts / root |
| `bit_reverse` / `bit_rotation` / `shuffle` | Bit-permute destinations | Structured permutations |
| `tornado` | Far along one dimension | At N=16 same as neighbor (see note) |
| `neighbor` | Adjacent core | Locality; everyone looks good |

**Tornado note:** With 16 cores, tornado’s destination function collapses to the same as neighbor — identical results. At 64+ they diverge; keep for scale-up.

**Still to add (should / stretch):**

| Pattern | Why |
|---|---|
| Hotspot | One popular destination (directory / MC) |
| Many-to-few / few-to-many | GPU → memory controllers (Bakhoda) |
| Ring / tree all-reduce | AI training collectives |
| Trace-driven (Netrace, PARSEC via Ruby) | “Synthetic wasn’t lying” |

### 8.5 Assumptions (Milestone 1)

- Synthetic Bernoulli injection; independent cores  
- Single-flit packets  
- Every link = 1 cycle (no wire-length penalty)  
- Identical routers: 1-cycle delay, 4 VCs  
- Same (src→dst) mapping across topologies for a given pattern  
- Clocks forced to **1 GHz** for tester and Ruby/network (see gotchas)

### 8.6 Saturation detection (operational)

Mark saturation at the first λ where:

- average latency > **3 × T₀**, or  
- packets received < **95%** of offered  

Sweep λ from **0.025 … 0.800** in steps of 0.025.

---

## 9. Milestone 1 results (keep handy)

### 9.1 Structural properties (16 cores)

| | Mesh | Torus | Ring | Tree |
|---|---:|---:|---:|---:|
| Routers | 16 | 16 | 16 | 31 |
| Links | 24 | 32 | 16 | 30 |
| Degree | 2–4 | 4 | 2 | 1–3 |
| Diameter (hops) | 6 | 4 | 8 | 8 |
| Avg hops (core pairs) | 2.67 | 2.13 | 4.27 | 6.53 |
| Bisection width | 4 | 8 | 2 | 1 |

Figures: `figures/networks.png`, `figures/prelim_results.png`.

### 9.2 Uniform traffic order (simulated saturation)

Tree **0.225** < Ring **0.275**† < Mesh **0.625** < Torus **0.75**  
(† Ring = deadlock freeze, not true capacity.)

Same order as bisection width (1, 2, 4, 8).

### 9.3 Predicted / simulated saturation (packets/core/cycle)

Higher is better. Simulated values on 0.025 grid.

| | Uniform | Transpose | Bit-compl. | Bit-rev. | Bit-rot. | Shuffle | Neighbor‡ |
|---|---|---|---|---|---|---|---|
| **Mesh** | 1.00 / 0.63 | 0.33 / 0.35 | 0.50 / 0.50 | 0.33 / 0.35 | 0.50 / 0.50 | 0.50 / 0.50 | 1.00 / 0.80 |
| **Torus** | 1.00 / 0.75 | 0.67 / 0.60 | 1.00 / 0.80 | 0.67 / 0.55 | 0.67 / 0.63 | 0.67 / 0.65 | 1.00 / 0.80 |
| **Ring** | 0.50 / 0.28† | 0.33 / 0.30 | 0.25 / 0.25 | 0.33 / 0.30 | 0.25 / 0.25 | 0.25 / 0.25 | 1.00 / 0.80 |
| **Tree** | 0.25 / 0.23 | 0.25 / 0.23 | 0.13 / 0.13 | 0.25 / 0.23 | 0.25 / 0.23 | 0.25 / 0.25 | 1.00 / 0.80 |

† Deadlock. ‡ Tornado ≡ neighbor at N=16.

**Headline numbers:** mean |pred − sim| ≈ **5.3%** when the network is deadlock-free; exact hits on Tree/Mesh bit-complement.

### 9.4 Takeaways for final narrative

1. Low load → **hops win** (Torus best, Tree worst).  
2. High load → **busiest link / bisection** dominate; traffic decides the winner (neighbor: Mesh≈Torus; bit-complement: Torus ~1.6× Mesh).  
3. Graph is powerful but incomplete: **contention** and **deadlock**.  
4. Next story beat: fix deadlock → remeasure Ring/Torus → RQ3 + industry map.

---

## 10. Real chips: case-study ideas

Use these as RQ3 “why industry chose X” sketches (cite papers in §15).

| Product / line | Topology flavor | Traffic intuition | Our angle |
|---|---|---|---|
| **Intel** ring → mesh (e.g. older client rings; Xeon meshes) | Ring cheap at small N; mesh scales | Rings deadlock-prone without care; bisection stays 2 | Matches our Ring vs Mesh uniform gap |
| **Intel Knights Landing / mesh** | 2D mesh | Cache / directory traffic | XY + mesh = our baseline |
| **AMD Infinity Fabric / chiplets** | Hierarchical / multi-die | Cross-die is expensive | Stretch: heterogeneous link costs |
| **NVIDIA GPUs** | Crossbar / mesh-like + many-to-few to L2/MC | Many cores → few MCs | Add many-to-few pattern; cite Bakhoda |
| **Google TPU** | Torus-ish pods / ICI | Bulk synchronous / collective-ish | Torus wrap helps long-haul (our bit-complement) |
| **Tilera / manycore research** | Mesh | General CMP | Classic mesh story |
| **HPC (Cray, etc.)** | Dragonfly, fat-tree | Global permutations | Stretch topology |

**Case-study template (for slides):**  
(1) What topology? (2) What traffic do they care about? (3) Which of our metrics would have predicted that? (4) What did our sim get wrong that silicon must still handle (power, wire delay, protocol)?

---

## 11. Timeline (remaining)

| When | Focus | Checkpoint |
|---|---|---|
| Now – early Nov | Dateline VCs; re-run Ring/Torus | Deadlock gone past predicted saturation |
| Mid Nov | RQ3 analysis + real-chip case study | One figure/story per open question |
| Late Nov | 64-core / cost model / new traffic (as time allows) | Draft final slides + report |
| First week of Dec | Final presentation + polished report | Submit |

**Fallbacks:** if deadlock slips, report Ring *below* saturation and flag deadlock explicitly (already honest in Milestone 1). If 64-core is too slow, keep 16-core for extras and show 64 only for 1–2 key cells.

---

## 12. Stretch goals (expansive)

Ordered roughly by “wow / effort” for recruiting and final polish. Pick freely; nothing here is committed.

### A. Must-adjacent (almost should)

| # | Goal | What | Why it matters |
|---|---|---|---|
| A1 | **Dateline VC Ring/Torus** | VC classes crossing a dateline (Dally & Seitz) | Unlocks honest Ring/Torus Θ_sat |
| A2 | **Link-utilization heatmaps** | Compare γ_ℓ vs Garnet `int_link_utilization` | Visual RQ2 proof |
| A3 | **Warm-up + drain** | Ignore first/last K cycles | Cleaner latency curves |
| A4 | **Cost / efficiency tables** | Θ_sat per link, per router, per “wire budget” | Design-facing conclusion |

### B. Topology & routing

| # | Goal | What | Why |
|---|---|---|---|
| B1 | **Flattened Butterfly** | Implement in Garnet + graph model | Classic high-radix paper (Kim et al.) |
| B2 | **Fat-tree / MECS / Kite** | One more hierarchical or irregular topology | Shows breadth |
| B3 | **O1TURN** | Two deterministic tables, random pick | Simple multipath; betweenness story 2.0 |
| B4 | **Valiant routing** | Random intermediate then DOR | Throughput vs latency trade |
| B5 | **Adaptive routing (RCA / odd-even)** | Congestion-aware | Interview classic |
| B6 | **Custom topology search** | Use fast Θ\* model to add K links under budget | “Design automation” punchline |

### C. Traffic & workloads

| # | Goal | What | Why |
|---|---|---|---|
| C1 | **Hotspot** | p% traffic to one node | Directory / MC stress |
| C2 | **Many-to-few** | GPU-style | Bakhoda relevance |
| C3 | **Collectives** | Ring/tree all-reduce schedules | AI training narrative |
| C4 | **Netrace / full-system Ruby** | Real or nearly-real traces | External validity |
| C5 | **SCALE-Sim / ASTRA-sim coupling** | Export AI traffic into Garnet | NVIDIA/TPU flavor |

### D. Modeling depth

| # | Goal | What | Why |
|---|---|---|---|
| D1 | **Multi-flit packets** | 5-flit / 9-flit sweeps | Serialization + HOL effects |
| D2 | **Unequal wire delay** | Delay ∝ Manhattan / hierarchy level | Punish long Torus wraps honestly |
| D3 | **VC / buffer depth sweep** | 1–8 VCs, depth 1–4 | Contention story quantitative |
| D4 | **Learned predictor** | Regress Θ_sat, T₀ from graph features | ML + arch crossover |
| D5 | **Queueing correction** | M/D/1 or mean-value analysis on top of γ_ℓ | Close Mesh uniform 1.0→0.63 gap |

### E. Hardware / systems (high recruiting value)

| # | Goal | What | Why |
|---|---|---|---|
| E1 | **Upstream gem5 PRs** | Ring/Torus/Tree (+ traffic fixes) | Public, reviewable artifact |
| E2 | **RTL micro-router** | SystemVerilog + Verilator; optional Yosys/OpenROAD | Area/timing on résumé |
| E3 | **Chiplet / heterogeneous links** | Slow/expensive inter-die links in Garnet | AMD/NVIDIA narrative |
| E4 | **BookSim 2 cross-check** | Same topology in BookSim | Simulator-agnostic claim |
| E5 | **FPGA soft NoC** | Tiny mesh on cheap FPGA | Demo video energy |

### F. Communication / packaging

| # | Goal | What | Why |
|---|---|---|---|
| F1 | **Interactive plot dashboard** | Streamlit/Observable over CSVs | Exploratory RQ3 |
| F2 | **Repro capsule** | One script: build → sweep subset → figures | Artifact badge energy |
| F3 | **Blog / README deep-dive** | Public write-up of city analogy + results | Recruiting link |

---

## 13. Open ideas & brainstorming

Capture half-baked thoughts here so they are not lost. Promote to §6/§12 when we commit.

- **Contention factor:** Is Mesh/Torus uniform gap (~0.63/0.75 vs 1.0) roughly constant across N? If yes, report Θ_sat ≈ α(N)·Θ\*.
- **Betweenness error map:** For every (topo, pattern), plot Θ\*_XY / Θ\*_betweenness — a single “when betweenness lies” figure.
- **Router radix vs hop trade:** Tree has low degree but terrible root; Flattened Butterfly opposite — cost model needed.
- **Dateline placement:** Does dateline *location* change measured Θ_sat or only deadlock threshold?
- **Asymmetric traffic:** Read-heavy vs write-heavy (request small, response large) — second virtual network.
- **Security / side channels:** Controllable contention as a timing channel (probably out of scope; cool interview tangent).
- **Power:** Activity factor × link toggles from γ_ℓ — optional energy proxy without full power model.
- **Naming:** Keep title focused on “structure → saturation”; avoid overselling ML unless we do D4.

---

## 14. How to present / résumé

### Slide spine

1. **Problem:** chips are networks; wiring choice matters.  
2. **Shortcut that fails:** plain betweenness ignores routing (§7 table + city analogy).  
3. **Approach:** graph predict → gem5 check → compare.  
4. **Results:** latency curves; pred vs sim; Ring deadlock.  
5. **Real chips:** Intel ring→mesh, TPU torus, GPU many-to-few.  
6. **What we built:** topologies + sweep + analysis.  
7. **Next / stretch:** dateline, RQ3, optional wow item from §12.

### Example résumé bullets (update numbers as we go)

- Extended gem5/Garnet with Ring, Torus, and Tree topologies and a parallel synthetic-traffic sweep (~1k cycle-accurate runs).  
- Built a routing-aware NoC performance model that predicts saturation within ~5% of simulation; showed plain betweenness can mispredict mesh throughput by up to ~2.4×.  
- Characterized Mesh/Torus/Ring/Tree under 8 traffic patterns; identified router contention and Ring deadlock as graph-blind failure modes.

---

## 15. Reading list & reference library

Papers listed by **author (year)**. Prefer reading ★ first. Links go to Google Scholar search (stable). Add notes in the “Our use” column as we go.

### 15.1 Core (read / re-read)

| ★ | Paper | Our use |
|---|---|---|
| ★★★ | **Dally & Towles (2004)** — [*Principles and Practices of Interconnection Networks*](https://scholar.google.com/scholar?q=Principles+and+Practices+of+Interconnection+Networks+Dally+Towles) | Bible: Ch. 3 latency & throughput; topology chapters; flow control |
| ★★★ | **Enright Jerger, Krishna & Peh (2017)** — [*On-Chip Networks*, 2nd ed.](https://scholar.google.com/scholar?q=On-Chip+Networks+Enright+Jerger+Krishna+Peh) | Modern NoC survey-book; DOR; VCs; dateline; industry examples |
| ★★★ | **Dally & Seitz (1987)** — [Deadlock-Free Message Routing](https://scholar.google.com/scholar?q=Deadlock-Free+Message+Routing+in+Multiprocessor+Interconnection+Networks) | Theoretical basis for dateline / VC escape paths |
| ★★ | **Dally & Towles (2001)** — [Route Packets, Not Wires](https://scholar.google.com/scholar?q=%22Route+Packets%2C+Not+Wires%22) | Historical NoC manifesto |
| ★★ | **Agarwal et al. (2009)** — [GARNET](https://scholar.google.com/scholar?q=GARNET+detailed+on-chip+network+model+full-system+simulator) | Our simulator’s NoC paper |
| ★★ | **Binkert et al. (2011)** — [The gem5 Simulator](https://scholar.google.com/scholar?q=%22The+gem5+simulator%22+Binkert) | Full-system host |
| ★★ | **Hagberg, Schult & Swart (2008)** — [NetworkX](https://scholar.google.com/scholar?q=Exploring+Network+Structure+Dynamics+and+Function+using+NetworkX) | Graph toolkit we use |
| ★ | **Lowe-Power et al. (2020)** — gem5 20.0+ updates | Cite if discussing modern gem5 |

### 15.2 Topology comparison & evaluation

| Paper | Our use |
|---|---|
| **Pande et al. (2005)** — [Performance evaluation and design trade-offs for NoC interconnect architectures](https://scholar.google.com/scholar?q=Performance+evaluation+and+design+trade-offs+for+network-on-chip+interconnect+architectures) | Classic multi-topology × traffic study (Milestone “previous work”) |
| **Balfour & Dally (2006)** — [Design tradeoffs for tiled CMP on-chip networks](https://scholar.google.com/scholar?q=Design+tradeoffs+for+tiled+CMP+on-chip+networks+Balfour) | Concentrated mesh, express channels, cost |
| **Benini & De Micheli (2002)** — [Networks on Chips: A New SoC Paradigm](https://scholar.google.com/scholar?q=Networks+on+Chips+A+New+SoC+Paradigm+Benini) | Early NoC vision |
| **Leiserson (1985)** — [Fat-trees](https://scholar.google.com/scholar?q=Fat-trees+universal+networks+Leiserson) | Hierarchical bandwidth idea behind “Tree but fair” |
| **Kim, Balfour & Dally (2007)** — [Flattened Butterfly](https://scholar.google.com/scholar?q=Flattened+Butterfly+Topology+for+On-Chip+Networks) | Stretch topology B1 |
| **Ogras & Marculescu (2006)** — [It’s a small world after all](https://scholar.google.com/scholar?q=It%27s+a+small+world+after+all+NoC+Ogras) | Application-specific / irregular topologies |
| **Bharadwaj et al. (2020)** — [Kite](https://scholar.google.com/scholar?q=Kite+NoC+Bharadwaj) | Modern irregular / chiplet-friendly NoC |
| **Kim et al. (2008)** — [Technology-driven, highly-scalable dragonfly](https://scholar.google.com/scholar?q=Dragonfly+topology+Kim+Dally) | System-scale; analogy for hierarchical cost |

### 15.3 Routing, VCs, deadlock, adaptivity

| Paper | Our use |
|---|---|
| **Dally (1992)** — [Virtual-channel flow control](https://scholar.google.com/scholar?q=Virtual-channel+flow+control+Dally) | Why VCs exist beyond deadlock |
| **Glass & Ni (1992)** — [The turn model](https://scholar.google.com/scholar?q=The+turn+model+for+adaptive+routing+Glass+Ni) | Deadlock-free adaptive without VCs |
| **Duato (1993/1995)** — deadlock theory for adaptive routing | Deeper theory if we go adaptive |
| **Seo et al. (2005)** — [O1TURN](https://scholar.google.com/scholar?q=O1TURN+near-optimal+turn+model+Seo) | Simple multipath stretch |
| **Valiant & Brebner (1981)** — [Universal schemes for parallel communication](https://scholar.google.com/scholar?q=Valiant+Brebner+universal+schemes+parallel+communication) | Randomization for load balance |
| **Gratz et al. (2008)** — [Regional Congestion Awareness (RCA)](https://scholar.google.com/scholar?q=Regional+Congestion+Awareness+Gratz) | Adaptive routing stretch |
| **Ma et al. (2011/2012)** — whole-packet forwarding / DBAR etc. | Optional adaptive reading |

### 15.4 Traffic, GPUs, accelerators

| Paper | Our use |
|---|---|
| **Bakhoda, Kim & Aamodt (2010)** — [Throughput-Effective On-Chip Networks for Manycore Accelerators](https://scholar.google.com/scholar?q=Throughput-Effective+On-Chip+Networks+for+Manycore+Accelerators) | GPU many-to-few; Milestone previous work |
| **Bakhoda et al. (2009)** — GPGPU-Sim | Context for GPU traffic |
| **Abts et al. / Google** — TPU interconnect papers | Torus / ICI case study |
| **Jouppi et al. (2017, 2020, 2023)** — TPU v2/v3/v4 ISCA/etc. | Industry torus / pod story |
| **Nakahara / Ai training collectives surveys** | All-reduce traffic stretch |
| **ASTRA-sim / Themis / similar** | Collective scheduling frameworks |

### 15.5 Industry & product architecture

| Paper / doc | Our use |
|---|---|
| **Hoskote et al. (2007)** — Intel 80-core teraflops processor | Early mesh manycore |
| **Sodani et al. (2016)** — Knights Landing | Mesh + directory |
| **Intel Xeon / mesh whitepapers** | Ring→mesh narrative |
| **Naffziger et al. (2021)** — AMD chiplet / Infinity Fabric | Multi-die cost |
| **Arunkumar et al. (2017)** — MCM-GPU | Multi-chip module GPU NoC |
| **NVIDIA GPU whitepapers** (arch-dependent) | Crossbar/mesh + L2 slices |
| **Wentzlaff et al. / Tilera** | Mesh CMP |

### 15.6 Simulators & tooling

| Resource | Our use |
|---|---|
| [gem5 Garnet synthetic traffic docs](https://www.gem5.org/documentation/general_docs/ruby/garnet_synthetic_traffic/) | Flags, patterns, stats |
| **Jiang et al. (2013)** — [BookSim 2.0](https://scholar.google.com/scholar?q=BookSim+2.0+Jiang) | Alternate cycle-accurate NoC sim |
| **Netrace** (Hestness / Keckler) | Trace-driven stretch |
| **OpenROAD / Yosys / Verilator** | RTL stretch toolchain |

### 15.7 Graph metrics & network science (optional depth)

| Paper / idea | Our use |
|---|---|
| Freeman betweenness; Brandes algorithm | How NetworkX computes betweenness |
| Flow / cut theorems (max-flow min-cut) | Tie bisection to capacity |
| Spectral gap / expanders | “Why random-ish topologies route well” (advanced) |

### 15.8 Potential references (skim if relevant; not committed)

- Sanchez et al. — SCOC / Jigsaw (cache + interconnect co-design)  
- Krishna / Peh — single-cycle routers, SMART NoC  
- Moscibroda & Mutlu — residual packet lifetimes / HoL  
- Fallin et al. — CHIPPER (bufferless)  
- Michelogiannakis / Dally — microarchitecture of routers  
- Passas / Katevenis — cost models for NoCs  
- Recent chiplet NoC workshops (NOCS, HPCA tutorials)  
- CMU / Peh / Krishna course notes if available  

*(Add PDFs or local notes under `Documentation/` as we collect them; don’t commit copyrighted PDFs if the course forbids it.)*

### 15.9 Citation keys we already use in LaTeX

| Key | Work |
|---|---|
| `b1` | Bakhoda, Kim, Aamodt (2010) |
| `b2` | Pande et al. (2005) |
| `b3` | Binkert et al. gem5 (2011) |
| `b4` | Agarwal et al. GARNET (2009) |
| `b5` | Dally & Towles book (2004) |
| `b6` | Dally & Seitz (1987) |
| `b7` | NetworkX (2008) |
| `ocn` | Enright Jerger, Krishna, Peh (2017) |

---

## Appendix A: Formulas & worked examples

### A.1 Empty-network latency

\[
T_0 \approx H_{\mathrm{avg}}(t_r + t_w) + T_{\mathrm{fixed}}
\]

Milestone 1 empirical: \(T_0 = 2.01 H_{\mathrm{avg}} + 4.98\).

### A.2 Ideal saturation

\[
\Theta^{*} = \min\bigl(1,\; 1 / \max_\ell \gamma_\ell\bigr)
\]

Real Θ_sat ≤ Θ\* (contention, imperfect VC use, deadlock).

### A.3 Channel load sketch

For unit injection from every source, with traffic matrix \(T\):

\[
\gamma_\ell = \sum_{s,d} T_{sd}\cdot \mathbf{1}[\ell \in \mathrm{path}(s\to d)]
\]

Implementation: `analysis/prelim_channel_load.py` (also mirrors Garnet destination functions).

### A.4 Bit-complement intuition (16-core)

- **Tree:** one root cut → γ_max = 8 → Θ\* = 0.125 (sim 0.13 ≈ 0.125 on grid).  
- **Mesh:** four mid vertical links share load → Θ\* = 0.5 (sim 0.50).  
- **Torus:** wrap-arounds shorten paths / add cuts → higher Θ\* (pred 1.0 / sim 0.80).

---

## Appendix B: gem5 / Garnet cheat sheet

### B.1 Example command

```bash
./build/NULL/gem5.opt configs/example/garnet_synth_traffic.py \
  --network=garnet --num-cpus=16 --num-dirs=16 \
  --topology=Mesh_XY --mesh-rows=4 --routing-algorithm=1 \
  --synthetic=uniform_random --injectionrate=0.05 --inj-vnet=0 \
  --sys-clock=1GHz --ruby-clock=1GHz \
  --sim-cycles=20000000
```

Prefer `scripts/sweep.py` (sets clocks; converts `--sim-cycles` correctly).

### B.2 Useful stats (`stats.txt`)

| Stat | Meaning |
|---|---|
| `average_packet_latency` | End-to-end cycles |
| `average_hops` | Measured hop count |
| `packets_injected` / `packets_received` | Delivery check |
| `int_link_utilization` | Per-link use (for γ validation) |

### B.3 Baseline settings (Milestone 1)

| Setting | Value |
|---|---|
| Cores | 16 (4×4); 64 planned |
| VCs | 4 |
| Router / link delay | 1 cycle |
| Link width | 128 bits |
| Packet size | 1 flit |
| Injection sweep | 0.025 … 0.800 |
| Saturation rule | latency > 3×T₀ or delivery < 95% |

### B.4 Gotchas we hit

| Finding | What we do |
|---|---|
| No Ring/Torus/Tree upstream | Implemented under `topologies/` |
| `outportComputeCustom()` is a stub | Place for dateline / custom routing |
| VC = first free | Need VC **classes** for deadlock freedom |
| Patterns need square power-of-two N | Use 16 or 64 |
| `--sim-cycles` is in picoseconds | Sweep script converts |
| Default clocks: network 2 GHz, tester 1 GHz | Force both to **1 GHz** |
| Tree + `--mesh-rows` | Use 0 / non-mesh layout |
| tornado ≡ neighbor at N=16 | Document; diverge at 64 |
| No warm-up | Still to add |
| ProcessPool vs threads on macOS | Sweep uses thread pool where needed |

### B.5 Dateline implementation sketch (next work)

1. Pick a dateline link (or set of links) on the ring/torus dimension.  
2. Packets that **cross** the dateline bump to a higher VC class.  
3. Ensure escape path / class ordering is acyclic (Dally & Seitz / OCN book).  
4. Re-sweep Ring (+ Torus) uniform and confirm delivery stays healthy past old freeze point.  
5. Compare new Θ_sat to Θ\*.

---

## Appendix C: Repo layout & scripts

```
Network-On-Chip/
├── gem5/                      submodule (Garnet_standalone / NULL build)
├── topologies/                Ring.py, Torus.py, Tree.py
├── scripts/sweep.py           parallel injection sweeps → results/
├── analysis/
│   ├── prelim_channel_load.py routing-aware Θ* (analytical)
│   ├── plot_networks.py       topology figures
│   └── prelim_results.py      pred vs sim plots / tables
├── figures/                   networks.png, prelim_results.png
├── results/
│   ├── prelim_fine.csv        full 1024-run data
│   ├── saturation_summary.csv per (topo, pattern) summary
│   └── runs/                  per-run dirs (often gitignored)
├── clang++-sdk                macOS build workaround
└── Documentation/
    ├── Project_Documentation.md  ← this file (primary living doc)
    ├── Milestone_0.pdf
    ├── 18755_Milestone1.pdf
    └── Latex_files/Milestone_1/
```

**Typical workflow**

```bash
# analytical Θ* / betweenness comparison
python analysis/prelim_channel_load.py

# gem5 sweep (long)
python scripts/sweep.py   # see script for flags

# plots from CSVs
python analysis/prelim_results.py
python analysis/plot_networks.py
```

---

## Appendix D: Interview talking points

Use as a checklist; deepen one bullet into a 5-minute whiteboard.

1. **What is an NoC and why not a bus?** Bandwidth, concurrency, scalability.  
2. **Mesh vs torus vs ring vs tree** — hops, bisection, wiring cost.  
3. **XY routing** — deadlock-free on mesh why? (turn restrictions / acyclic CDG).  
4. **Why rings need datelines** — circular dependence; VC classes.  
5. **Throughput bound from critical channel** — derive Θ\* on a whiteboard.  
6. **Betweenness vs routing-aware load** — our 2.4× example.  
7. **What cycle-accurate sim adds** — contention, VCs, timing; what it still abstracts.  
8. **GPU many-to-few** — bottleneck near MCs; topology implications.  
9. **Chiplet era** — expensive long links change the optimal topology.  
10. **Our concrete artifact** — gem5 topologies, 1k-run sweep, 5% model, known failure modes.

---

## Document changelog

| Date | Change |
|---|---|
| Oct 2026 | Initial proposal + Steve/Kushal split (later removed) |
| Oct 2026 | Post–Milestone 1 sync: done/left, results, no individual ownership |
| Oct 2026 | Expanded into living primary resource: theory, full results, stretch catalog, reference library, appendices |
| Oct 2026 | Renamed from Project_Proposal.md → Project_Documentation.md |

*When you learn something durable (a gotcha, a paper, a decision), add it here.*
