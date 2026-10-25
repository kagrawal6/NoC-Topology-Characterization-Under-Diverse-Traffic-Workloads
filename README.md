# Network-on-Chip Topology Characterization Under Diverse Traffic Workloads

Predict Network-on-Chip performance from **topology + routing + traffic**, then check against **gem5/Garnet**.

We model Mesh, Torus, Ring, and Tree as graphs, predict low-load latency from hop count and saturation from the busiest link, and validate with cycle-accurate sweeps (~1k runs at 16 cores in Milestone 1; routing-aware saturation within ~5% of simulation).

**Team:** Steve Cho, Kushal Agrawal (CMU 18-755)

More detail (theory, results, stretch goals, references): **[Documentation/Project_Documentation.md](Documentation/Project_Documentation.md)**

## Setup

```bash
git submodule update --init --recursive

# macOS: export CXX=$(pwd)/clang++-sdk   # if default clang++ fails
cd gem5 && scons build/NULL/gem5.opt -j$(sysctl -n hw.ncpu) && cd ..
pip install networkx numpy scipy matplotlib pandas
```

Needs a C++ toolchain and [scons](https://scons.org/). Binary: `gem5/build/NULL/gem5.opt`.

## Run

```bash
# Analytical Θ* / graph metrics (no gem5)
python analysis/prelim_channel_load.py

# Garnet sweep → CSV
python scripts/sweep.py --num-cpus 16 --out results/prelim_fine.csv

# Milestone 1 injection grid
python scripts/sweep.py --num-cpus 16 --rates $(seq 0.025 0.025 0.8) \
  --out results/prelim_fine.csv

# Plots
python analysis/plot_networks.py
python analysis/prelim_results.py results/prelim_fine.csv
```

`scripts/sweep.py` symlinks `topologies/` into gem5 and sets clocks correctly. Useful flags: `--topologies`, `--patterns`, `--rates`, `--vcs`, `--jobs`, `--timeout`.

## Layout

| Path | Contents |
|---|---|
| `topologies/` | Ring, Torus, Tree (Mesh is built into gem5) |
| `scripts/` | `sweep.py` |
| `analysis/` | Channel-load model + plotting |
| `results/` | CSVs (+ `runs/`) |
| `figures/` | `networks.png`, `prelim_results.png` |
| `Documentation/` | Project docs and milestones |
