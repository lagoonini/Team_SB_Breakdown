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

# 3. General code explanation

## `solve_static_vrptw`: one-shot interface to the static VRPTW solver

```python
def solve_static_vrptw(instance, time_limit=3600, seed=1, id=None, tmp_dir=None, verbose=False):
```

### 3.1 Inputs

* `instance`: a dict describing the static VRPTW instance, containing fields :
  * `coords`, `demands`, `capacity`, `time_windows`,
    `service_times`, `duration_matrix`, `is_depot`, etc.
* `time_limit`: maximum runtime (in seconds) given to HGS.
* `seed`: random seed passed to the HGS solver.
* `id`: optional identifier used in the temporary filename (avoid clashes).
* `tmp_dir`: directory where the temporary `.vrptw` file will be written.
* `verbose`: if `True`, log the HGS command.

---

### 3.2 Early exits for very small instances

Inside the function there are two special cases:

1. **No customers (only depot):**

   ```python
   if instance['coords'].shape[0] <= 1:
       yield [], 0
       return
   ```

   Explanation:

   * If there is only the depot and no customers, the solution is empty and the cost is 0.

2. **Exactly one customer:**

   ```python
   if instance['coords'].shape[0] <= 2:
       solution = [[1]]
       cost = tools.validate_static_solution(instance, solution)
       yield solution, cost
       return
   ```

   Explanation:

   * Node `0` = depot, node `1` = the only customer.
   * The single route is `[[1]]` (implicitly depot → 1 → depot).
   * `validate_static_solution` recomputes the cost from the instance to be consistent.

For these small instances, the code **skips HGS entirely** and returns a trivial solution.

---

### 2.3 Writing the instance file for HGS

For larger instances, the function prepares an input file for the C++ solver:

```python
os.makedirs(tmp_dir, exist_ok=True)
instance_filename = os.path.join(
    tmp_dir,
    "problem_{}.vrptw".format(id)
) if id is not None else os.path.join(tmp_dir, "problem.vrptw")
tools.write_vrplib(instance_filename, instance, is_vrptw=True)
```

Steps:

* Ensure `tmp_dir` exists.
* Choose the filename:

  * `tmp_dir/problem_<id>.vrptw` if `id` is given,
  * otherwise `tmp_dir/problem.vrptw`.
* Call `tools.write_vrplib(...)` to convert the `instance` dict into a VRPLIB/VRPTW-style text file, with sections like:

  * `NODE_COORD_SECTION`
  * `DEMAND_SECTION`
  * time windows, service times, etc.

This text file is what the HGS C++ binary reads.

---

### 2.4 Selecting the HGS executable

```python
executable = os.path.join('hgs', 'hgsvrptw')
if platform.system() == 'Windows' and os.path.isfile(executable + '.exe'):
    executable = executable + '.exe'
assert os.path.isfile(executable), f"HGS executable {executable} does not exist!"
```

* On Linux/macOS it expects `hgs/hgsvrptw`.
* On Windows, if `hgs/hgsvrptw.exe` exists, it uses that.
* If the executable can’t be found, it raises an assertion error.

---

### 2.5 Building the HGS command

```python
hgs_cmd = [
    executable,
    instance_filename,
    str(int(max(time_limit - 1, 1))),
    '-seed', str(seed),
    '-veh', '-1',
    '-useWallClockTime', '1'
]
```

This corresponds to:

* `executable`: path to the HGS solver.
* `instance_filename`: path to the `.vrptw` instance.
* `str(int(max(time_limit - 1, 1)))`: time limit (in seconds) passed to HGS.
  They subtract 1s to leave a little margin for overhead.
* `-seed <seed>`: set the random seed.
* `-veh -1`: allow an unrestricted number of vehicles (solver chooses).
* `-useWallClockTime 1`: tells HGS to enforce the limit using wall-clock time.

Logging if `verbose`:

```python
if verbose == True:
    log(str(hgs_cmd))
```
---

### 2.6 Running HGS and parsing its output

```python
with subprocess.Popen(hgs_cmd, stdout=subprocess.PIPE, text=True) as p:
    routes = []
    for line in p.stdout:
        line = line.strip()
        if line.startswith('Route'):
            label, route = line.split(": ")
            route_nr = int(label.split("#")[-1])
            assert route_nr == len(routes) + 1, "Route number should be strictly increasing"
            routes.append([int(node) for node in route.split(" ")])
        elif line.startswith('Cost'):
            solution = routes
            cost = int(line.split(" ")[-1].strip())
            check_cost = tools.validate_static_solution(instance, solution)
            assert cost == check_cost, "Cost of HGS VRPTW solution could not be validated"
            yield solution, cost
            routes = []
        elif "EXCEPTION" in line:
            raise Exception("HGS failed with exception: " + line)
    assert len(routes) == 0, "HGS has terminated with imcomplete solution (is the line with Cost missing?)"
```

Explanation:

* Launches the HGS solver (`hgsvrptw`) in a subprocess.
* Iterates line by line over its stdout.

**Route lines**

For lines like:

`Route #1: 3 5 7 2`

it:

* splits label and route string at `": "`,
* extracts the route number (`#1` → 1),
* checks route numbers are strictly increasing (`1, 2, 3, ...`),
* parses the node indices into an integer list `[3, 5, 7, 2]`,
* appends that list to `routes`.

**Cost line**

For a line like:

`Cost 12345`

it:

* treats this as the end of the current solution,
* sets `solution = routes`,
* parses the last token as `cost = 12345`,
* recomputes `check_cost = tools.validate_static_solution(instance, solution)`,
* asserts that `cost == check_cost` (sanity check),
* `yield solution, cost` to the caller,
* resets `routes = []` to start collecting a potential next solution.

**Exceptions**

* If any line contains `"EXCEPTION"`, it raises a Python `Exception` to signal that HGS crashed or failed.

**Final consistency check**

* After the loop, it asserts `len(routes) == 0` to ensure that the solver did not terminate after printing routes but before printing a `Cost` line.

---

### Generator behavior

`solve_static_vrptw` is a **generator function**:

* It can yield multiple `(solution, cost)` pairs if the C++ solver prints several solutions (e.g. intermediate bests).
* In practice, most code using it does:

  ```python
  solutions = list(solve_static_vrptw(...))
  best_solution, best_cost = solutions[-1]
  ```

It keeps the **last** solution, which is usually the best one found before the time limit.

