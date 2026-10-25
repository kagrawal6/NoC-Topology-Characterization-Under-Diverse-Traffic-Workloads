# Network-on-Chip Topology Characterization Under Diverse Traffic Workloads

Predict Network-on-Chip performance from **topology + routing + traffic**, then check the predictions in **gem5/Garnet**.

We model Mesh, Torus, Ring, and Tree as graphs (NetworkX), predict low-load latency from hop count and saturation from the busiest link, and validate against cycle-accurate Garnet sweeps. Milestone 1: ~1k runs on 16 cores; routing-aware saturation within ~5% of simulation (Ring deadlock is a known gap).

**Team:** Steve Cho, Kushal Agrawal (CMU 18-755)

For ideas, theory, results, stretch goals, and references, see **[Documentation/Project_Documentation.md](Documentation/Project_Documentation.md)**.

---

## Setup

```bash
git submodule update --init --recursive

# Build gem5 Garnet (NULL / standalone). On macOS, use the SDK clang wrapper if needed:
#   export CXX=$(pwd)/clang++-sdk
cd gem5
scons build/NULL/gem5.opt -j$(sysctl -n hw.ncpu)   # or -j$(nproc) on Linux
cd ..

pip install networkx numpy scipy matplotlib pandas
```

Requires a working C++ toolchain and [scons](https://scons.org/). Binary expected at `gem5/build/NULL/gem5.opt`.

---

## How to run

### Analytical predictions (no gem5)

```bash
python analysis/prelim_channel_load.py
```

Prints graph metrics and ideal saturation Θ\* under different routing assumptions.

### gem5 / Garnet sweep

```bash
# Full-ish sweep → results/prelim_fine.csv
python scripts/sweep.py --num-cpus 16 --out results/prelim_fine.csv

# Finer injection grid (Milestone 1 style)
python scripts/sweep.py --num-cpus 16 \
  --rates $(seq 0.025 0.025 0.8) \
  --out results/prelim_fine.csv

# Subset example
python scripts/sweep.py --topologies Mesh_XY Torus --patterns uniform_random \
  --rates 0.05 0.1 0.2 --jobs 4 --out results/smoke.csv
```

Custom topologies in `topologies/` are symlinked into gem5 automatically. Useful flags: `--vcs`, `--sim-cycles`, `--timeout`, `--jobs`.

### Plots

```bash
python analysis/plot_networks.py                          # figures/networks.png
python analysis/prelim_results.py results/prelim_fine.csv # figures/prelim_results.png + summary CSV
```

### Single Garnet run (manual)

```bash
./gem5/build/NULL/gem5.opt configs/example/garnet_synth_traffic.py \
  --network=garnet --num-cpus=16 --num-dirs=16 \
  --topology=Mesh_XY --mesh-rows=4 --routing-algorithm=1 \
  --synthetic=uniform_random --injectionrate=0.05 --inj-vnet=0 \
  --sys-clock=1GHz --ruby-clock=1GHz --sim-cycles=20000000
```

(Run from `gem5/`, or prefer `scripts/sweep.py`, which fixes clocks and tick conversion.)

---

## Repo layout

```
topologies/     Ring, Torus, Tree (Mesh is built into gem5)
scripts/        sweep.py
analysis/       channel-load model + plotting
results/        CSV outputs (+ runs/)
figures/        networks.png, prelim_results.png
Documentation/  Project_Documentation.md, milestones
```

---

## More info

**[Documentation/Project_Documentation.md](Documentation/Project_Documentation.md)** — primary living doc (glossary, RQs, done/left, results tables, stretch goals, reading list, gem5 gotchas).
