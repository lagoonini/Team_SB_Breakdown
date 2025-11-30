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

* Dynamic-flavored variant of HGS (not necessarily used in the main pipeline)

`node_selector/`

* `node_selector.py` + `DispatchModel.py` + checkpoint

`priority_setter/`

* `priority_setter.py` + `PriorityModel.py` + checkpoint
