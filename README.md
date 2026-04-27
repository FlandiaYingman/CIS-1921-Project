# Imma Hungry — Food Rescue Optimizer

CIS 1921 Final Project (Spring 2026)
**Authors:** Harry Li, Nabo Yu

A constraint-driven vehicle-routing study for the
[Share Food Program](https://sharefoodprogram.org/) of Greater Philadelphia.
The notebook compares three solvers (greedy, regret-2 metaheuristic,
Google OR-Tools CP/Routing) on a real-world Philadelphia dataset and
reports how each scales with instance size, what binding constraints
limit deliveries, and which approach delivers the most food per mile and
per vehicle-hour.

## Problem in one paragraph

A fleet of refrigerated vehicles starts at a central depot, picks up
perishable food from a set of donors, and drops it at neighborhood
pantries. Every donation must reach a pantry within a 2-hour spoilage
window; every vehicle has a weight capacity; every pantry has a daily
weight cap and a per-nutrient cap; every donor and pantry has a service
time window; every vehicle must return to the depot before the 8-hour
horizon closes. We want to deliver as much food as possible while
respecting all of these constraints.

## File layout

```
project.ipynb                         # main deliverable (run top-to-bottom)
README.md                             # this file
data/
  business_licenses.csv               # OpenDataPhilly: synthetic donor pool
  free_meal_sites.csv                 # OpenDataPhilly: pantries / meal sites
  route_matrix.csv                    # haversine fallback distances
  route_matrix_osrm.csv               # OSRM driving distances (CSV cache)
  route_matrix_osrm_50d_14p.npz       # OSRM distance + duration matrix (binary cache)
```

## Dependencies

Python 3.11+ with:

- `numpy`, `pandas`, `polars`
- `matplotlib`, `seaborn`
- `ortools` (≥ 9.7)
- `requests` (only needed if rebuilding the OSRM cache from scratch)

A pinned conda environment lives in `.conda/` next to the notebook.

```powershell
# from the project root
conda create -p .\.conda python=3.11 -y
.\.conda\Scripts\activate
pip install numpy pandas polars matplotlib seaborn ortools requests jupyter
```

## How to run

1. Open `project.ipynb` in VS Code (or `jupyter lab`).
2. Select the `.conda` interpreter.
3. **Run All**. The full sweep (60 instances × 3 solvers) takes a few
   minutes because OR-Tools is given a 10-second budget per instance.
4. Cached OSRM matrices in `data/` are loaded automatically; no network
   calls are made on a normal run.

## What the notebook contains

1. **Data loading** — pulls business licenses (donors) and free meal
   sites (pantries) from OpenDataPhilly, geocodes them, samples 50
   donors and 14 pantries.
2. **OSRM route matrix** — driving distance + duration for every
   donor↔pantry, depot↔donor, depot↔pantry pair (cached on disk).
3. **Greedy solver** — nearest-feasible insertion, one trip per
   vehicle, no rebalancing.
4. **Heuristic solver** — regret-2 insertion across vehicles, free
   pantry choice per donor, chained multi-trip routes, intra-route
   2-opt polish.
5. **OR-Tools solver** — CP/Routing with virtual pickup-delivery
   nodes, weight + time dimensions, 2-hour spoilage clock as an
   inter-pair constraint, per-pantry weight + nutrient caps,
   `PARALLEL_CHEAPEST_INSERTION` first solution and
   `GUIDED_LOCAL_SEARCH` metaheuristic.
6. **Common interface** — `SolverResult` dataclass and
   `make_instance(n_donors, n_pantries, n_vehicles, seed, ...)`
   parameterized generator.
7. **Scaling sweep** — `n_donors ∈ {10, 20, 35, 50}` × 5 seeds × 3
   solvers, with per-instance wall-clock timing.
8. **Binding-constraint diagnostic** — for every donor a greedy run
   fails to serve, classify the lowest-priority constraint that broke
   the assignment (vehicle capacity, pantry capacity, nutrient cap,
   donor / pantry / depot time window, spoilage clock, infeasible arc,
   no pantry assigned, no vehicle fits).
9. **User-configurable constraints** — final knob cell that lets a
   reader vary vehicle capacity, nutrient cap fraction, spoilage
   window, and depot horizon and re-run all three solvers on a single
   instance.
10. **Plots** — six-panel scaling + efficiency dashboard plus the
    binding-constraint bar chart.

## Reproducing the headline numbers

At `n_donors = 50`, `n_pantries = 14`, `n_vehicles = 5`, seed `1921`:

| Solver     | Food (lbs) | Distance (km) | Wall clock |
| ---------- | ---------- | ------------- | ---------- |
| Greedy     | ~675       | ~172          | < 0.01 s   |
| Heuristic  | ~1047      | ~380          | ~0.4 s     |
| OR-Tools   | ~392       | ~168          | 10 s (cap) |

The greedy solver is constrained mainly by pantry capacity (it commits
to the nearest pantry without rebalancing). OR-Tools struggles inside
the 10-second budget because the search space grows quickly with
virtual pickup-delivery nodes. The regret-2 heuristic finds the
strongest food/lbs trade-off in the time budget we tested.

## License

Course project; data sources are public via OpenDataPhilly.
