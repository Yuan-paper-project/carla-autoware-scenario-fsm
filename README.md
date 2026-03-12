[README_EN.md](https://github.com/user-attachments/files/25948918/README_EN.md)
# CARLA–Autoware Scenario FSM: OpenSCENARIO State Machine Monitoring

[![CARLA](https://img.shields.io/badge/CARLA-0.9.x-blue)](https://github.com/carla-simulator/carla)
[![Scenario Runner](https://img.shields.io/badge/Scenario%20Runner-OpenSCENARIO-green)](https://github.com/carla-simulator/scenario_runner)
[![op_bridge](https://img.shields.io/badge/op__bridge-CARLA--Autoware-orange)](https://github.com/hatem-darweesh/op_bridge)

FSM-based OpenSCENARIO scenario execution modeling and visualization for **CARLA + Scenario Runner + OpenPlanner (op_bridge)** co-simulation with Autoware and ADAS validation.

This repository provides **FSM-based scenario execution monitoring** for scenario-based validation of ADAS/ADS, combining **CARLA Simulator**, **Scenario Runner**, and the **OpenPlanner bridge ([op_bridge](https://github.com/hatem-darweesh/op_bridge))** with Autoware, and mapping OpenSCENARIO Storyboard elements to explicit runtime states with ROS 2 logging and HTML visualization.

---

## Overview

- **Problem**: Traditional OpenSCENARIO/Scenario Runner toolchains offer limited observability of scenario execution; the activation and termination of Story/Act/Event/Action are mostly encapsulated inside the interpreter, making phase-wise analysis and alignment difficult.
- **Approach**: Map the Storyboard hierarchy in .xosc (Story → Act → ManeuverGroup → Maneuver → Event → Action) to observable runtime states, publish state changes via **ROS 2 topics**, and display structure and execution state in real time with an **HTML visualization tool**.

Related work: *From State Machines to Traffic Safety: Benchmarking Driver Assistance Systems Using Structured Scenario Models* (Forschungspraktikum Mengchen CAI, TUM).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CARLA Simulator (0.9.x)                                                 │
│  ┌─────────────────────┐    OpenSCENARIO (.xosc)                        │
│  │  Scenario Runner     │ ◄─────── Scenario description & behavior tree   │
│  │  (open_scenario.py)  │    StoryElementStatusToBlackboard → Blackboard │
│  └──────────┬──────────┘    + ROS2 Logger → /scenario_runner/log         │
└─────────────┼───────────────────────────────────────────────────────────┘
              │
              │  op_bridge (OpenPlanner CARLA–Autoware bridge)
              │  https://github.com/hatem-darweesh/op_bridge
              │  - op_ros2_agent.py: CARLA agent ↔ ROS 2 / Autoware
              │  - run_srunner_agent_ros2.sh and related scripts
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ROS 2 / Autoware Universe                                               │
│  - Perception, planning, control closed loop                             │
│  - Subscribe to /scenario_runner/log for element states (RUNNING/END/…)  │
└─────────────────────────────────────────────────────────────────────────┘
              │
              │  /scenario_runner/log (JSON)
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  xosc_html_visualizer.py                                                 │
│  - Parse .xosc for static structure (Init, Story, Act, ManeuverGroup,…)  │
│  - Subscribe to ROS 2 log, update FSM (IDLE/RUNNING/COMPLETED/CANCELLED) │
│  - Generate HTML: structure view + execution state view, auto-refresh    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Dependencies

| Component | Description |
|-----------|-------------|
| **CARLA** | Simulation environment (0.9.15 used in the paper; op_bridge officially supports 0.9.13—check version compatibility). |
| **Scenario Runner** | Executes OpenSCENARIO 1.x (.xosc), parses Storyboard and drives traffic participants in CARLA. [CARLA Scenario Runner](https://github.com/carla-simulator/scenario_runner) |
| **op_bridge** | **OpenPlanner ROS bridge**: connects CARLA with Autoware/OpenPlanner and provides the agent–ROS 2 mapping. **[op_bridge](https://github.com/hatem-darweesh/op_bridge)** supports CARLA 0.9.13, Scenario Runner, Python 2.7/3 and ROS 1; for Autoware/ROS 2 integration see `op_scripts/run_srunner_agent.sh` and the ROS 2 port (e.g. `run_srunner_agent_ros2.sh`). |
| **Autoware Universe** | Autonomous driving stack (perception, planning, control), closed loop with CARLA via op_bridge. |
| **ROS 2** | Used for the state log topic `/scenario_runner/log` and visualization subscription. |

Example environment variables (adjust paths for your setup; see op_bridge README for full list):

```bash
export CARLA_ROOT=/path/to/carla
export SCENARIO_RUNNER_ROOT=/path/to/scenario_runner
export LEADERBOARD_ROOT=/path/to/op_bridge   # or this repo
export TEAM_CODE_ROOT=/path/to/op_agent
export PYTHONPATH=$PYTHONPATH:${CARLA_ROOT}/PythonAPI
# ... see op_bridge README for the rest
```

---

## Repository Contents

| File | Description |
|------|-------------|
| **open_scenario.py** | Upstream (or interface-compatible) version from [CARLA scenario_runner](https://github.com/carla-simulator/scenario_runner) (`srunner/scenarios/open_scenario.py`). Converts OpenSCENARIO to a py_trees behavior tree and uses **StoryElementStatusToBlackboard** to write Story/Act/Scene/Maneuver/Event/Action state to the Blackboard. A modified version adds **ROS 2 logging** (see “Differences from Upstream” below). |
| **xosc_html_visualizer.py** | **HTML visualizer** for OpenSCENARIO: parses .xosc for hierarchy, subscribes to `/scenario_runner/log`, maintains FSM state and generates HTML (structure view + execution state view, with IDLE/RUNNING/COMPLETED/CANCELLED color coding and periodic refresh). |
| **README_changed open-scenario.md** | Describes **differences between a modified open_scenario.py and upstream** (in English): ROS2 logger, termination deduplication, simplified Init (controller + speed only), removal of weather and actor add/delete, etc. |

---

## Differences from Upstream open_scenario.py

The **open_scenario.py** in this repo is the upstream scenario_runner version. In practice, a **modified** version is often used for co-simulation; main changes:

1. **StoryElementStatusToBlackboard**
   - Add `ros2_logger = get_logger()`; in `initialise()` log `RUNNING`, in `terminate()` log `END`/`CANCEL` etc. as JSON to ROS 2.
   - Use a blackboard `termination_flag` at termination to avoid duplicate logs.

2. **Init behavior**
   - Keep only **ControllerAction** and **initial speed**; remove LateralAction, RoutingAction, weather, EntityAction, etc. (handled elsewhere or by op_bridge).

3. **Removed**
   - `_create_weather_behavior()`, `_initialize_actors()`, etc.; weather and actor add/delete are no longer handled in this file.

See **[README_changed open-scenario.md](README_changed%20open-scenario.md)** for a detailed comparison.

---

## How to Run

Run the following in **separate terminals** (start CARLA first, then Scenario Runner, then op_bridge/agent, then the visualizer).

**1. Start CARLA**
```bash
cd ~/CC/carla_0.9.15 && make launch
```

**2. Set environment variables and start Scenario Runner** (new terminal; load an OpenSCENARIO file, e.g. `CyclistCrossing.xosc`)
```bash
export CARLA_ROOT=${HOME}/CC/carla_0.9.15
export PYTHONPATH="${CARLA_ROOT}/PythonAPI/carla/":${PYTHONPATH}
export PYTHONPATH=$PYTHONPATH:${CARLA_ROOT}/PythonAPI/carla/dist/carla-0.9.15-py3.10-linux-x86_64.egg
cd ~/CC/op_carla/scenario_runner
python3 scenario_runner.py --openscenario srunner/examples/CyclistCrossing.xosc --timeout 1600
```

**3. Start op_bridge and Autoware agent** (new terminal; connect CARLA and Autoware)
```bash
source /opt/ros/humble/setup.bash
source ~/CC/autoware/install/setup.bash
. ~/CC/op_carla/op_bridge/op_scripts/run_srunner_agent_ros2.sh
```

**4. Start HTML visualizer** (new terminal; subscribes to `/scenario_runner/log` and generates/refreshes HTML)
```bash
python3 ~/CC/op_carla/scenario_runner/srunner/tools/xosc_html_visualizer.py ~/CC/op_carla/scenario_runner/srunner/examples/CyclistCrossing.xosc
```
Open the generated `xosc_fsm.html` in a browser (path is printed; or run `python -m http.server 8000` in that directory and open the URL) to see the static structure and live execution state.

> For other scenarios, replace `CyclistCrossing.xosc` with e.g. `FollowLeadingVehicle.xosc`, `PedestrianCrossingFront.xosc`, or `IntersectionCollisionAvoidance.xosc`.

---

## Scenarios & ADAS Validation

Representative .xosc scenarios used in the paper and experiments:

- **FollowLeadingVehicle.xosc** / **OscControllerExample.xosc** — ACC car-following
- **CyclistCrossing.xosc** / **PedestrianCrossingFront.xosc** — AEB / VRU collision avoidance
- **IntersectionCollisionAvoidance.xosc** — Intersection conflict

The FSM allows alignment of semantic phases (e.g. “leading vehicle deceleration”, “pedestrian/cyclist starts crossing”, “ego braking”) with OpenSCENARIO Events and Actions, supporting phase-wise KPIs and regression testing.

---

## References

- Paper: *From State Machines to Traffic Safety: Benchmarking Driver Assistance Systems Using Structured Scenario Models* (Y. Gao, M. Cai, J. Betz, TUM).
- Public repo: [https://github.com/MengchenCAI/carla-autoware-scenario-fsm](https://github.com/MengchenCAI/carla-autoware-scenario-fsm)
- **OpenPlanner CARLA–Autoware bridge**: [**op_bridge**](https://github.com/hatem-darweesh/op_bridge) — OpenPlanner ROS bridge for CARLA and Scenario Runner.
- [CARLA Scenario Runner](https://github.com/carla-simulator/scenario_runner) — OpenSCENARIO support.
- [ASAM OpenSCENARIO](https://www.asam.net/standards/detail/openscenario/) — Scenario description standard.

---

## License

The **open_scenario.py** in this repository follows the original CARLA scenario_runner license (e.g. MIT). For other code and documentation, see the LICENSE file in the repository root, if present.
