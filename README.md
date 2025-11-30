# 1. Project structure breakdown:

* `solver.py` – main entry point
* `solver_static.py` – static baseline + HGS interface
* `solver_dynamic.py` – dynamic strategy using HGS + ML
* `environment.py` – environment abstraction & dynamic VRPTW simulator
* `tools.py` – I/O, VRPLIB utilities, validation, cost
* `requirements.txt`, `run.sh`, `install.sh`, `LICENSE.md`, `README.md`

`hgs/`

* C++ implementation of the static HGS VRPTW solver

`hgs_dynamic/`

* Dynamic variant of HGS
`node_selector/`

* `node_selector.py` + `DispatchModel.py` + checkpoint

`priority_setter/`

* `priority_setter.py` + `PriorityModel.py` + checkpoint




# 2. Algorithm behavior - static part

1. **Start**:

   * Give the Python script a VRPTW instance
   * Python converts it to the C++ expected text format.

2. **Optimization**:

   * C++ HGS:

     * reads the instance (Params),
     * creates an initial population of feasible/infeasible individuals,
     * iterates: selection → crossover → Split → local search → population update,
     * adaptively tunes penalties for capacity & time windows,
     * maintains the best feasible solution.

3. **End**:

   * when the time limit is reached, HGS prints:

     * the set of routes (each line is a route, starting/ending at depot),
     * the total distance / travel time.
   * Python parses that and returns:

     * a list of routes as lists of customer indices,
     * the numeric cost.
