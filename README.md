[README.md](https://github.com/user-attachments/files/25948902/README.md)
# CARLA–Autoware Scenario FSM: OpenSCENARIO State Machine Monitoring

[![CARLA](https://img.shields.io/badge/CARLA-0.9.x-blue)](https://github.com/carla-simulator/carla)
[![Scenario Runner](https://img.shields.io/badge/Scenario%20Runner-OpenSCENARIO-green)](https://github.com/carla-simulator/scenario_runner)
[![op_bridge](https://img.shields.io/badge/op__bridge-CARLA--Autoware-orange)](https://github.com/hatem-darweesh/op_bridge)

基于有限状态机（FSM）的 OpenSCENARIO 场景执行建模与可视化，用于 **CARLA + Scenario Runner + OpenPlanner (op_bridge)** 与 Autoware 的联合仿真与 ADAS 验证。

This repository provides **FSM-based scenario execution monitoring** for scenario-based validation of ADAS/ADS, combining **CARLA Simulator**, **Scenario Runner**, and the **OpenPlanner bridge ([op_bridge](https://github.com/hatem-darweesh/op_bridge))** with Autoware, and mapping OpenSCENARIO Storyboard elements to explicit runtime states with ROS 2 logging and HTML visualization.

---

## 概述 / Overview

- **问题**：传统 OpenSCENARIO/Scenario Runner 对场景执行过程的可观测性不足，Story/Act/Event/Action 的激活与终止多封装在解释器内部，难以做阶段化分析与对齐。
- **方案**：将 .xosc 中的 Storyboard 层次（Story → Act → ManeuverGroup → Maneuver → Event → Action）映射为可观测的运行时状态，通过 **ROS 2 话题** 发布状态变更，并用 **HTML 可视化工具** 实时展示结构与执行状态。

相关研究见论文：*From State Machines to Traffic Safety: Benchmarking Driver Assistance Systems Using Structured Scenario Models* (Forschungspraktikum Mengchen CAI, TUM).

---

## 系统架构 / Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CARLA Simulator (0.9.x)                                                 │
│  ┌─────────────────────┐    OpenSCENARIO (.xosc)                        │
│  │  Scenario Runner     │ ◄─────── 场景描述与行为树执行                   │
│  │  (open_scenario.py)  │    StoryElementStatusToBlackboard → Blackboard │
│  └──────────┬──────────┘    + ROS2 Logger → /scenario_runner/log         │
└─────────────┼───────────────────────────────────────────────────────────┘
              │
              │  op_bridge (OpenPlanner CARLA–Autoware bridge)
              │  https://github.com/hatem-darweesh/op_bridge
              │  - op_ros2_agent.py: CARLA agent ↔ ROS 2 / Autoware
              │  - run_srunner_agent_ros2.sh 等脚本
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  ROS 2 / Autoware Universe                                               │
│  - 感知、规划、控制闭环                                                    │
│  - 订阅 /scenario_runner/log 获取场景元素状态 (RUNNING / END / CANCEL)     │
└─────────────────────────────────────────────────────────────────────────┘
              │
              │  /scenario_runner/log (JSON)
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  xosc_html_visualizer.py                                                 │
│  - 解析 .xosc 得到静态结构 (Init, Story, Act, ManeuverGroup, Event, …)   │
│  - 订阅 ROS 2 log，更新 FSM 状态 (IDLE / RUNNING / COMPLETED / CANCELLED) │
│  - 生成 HTML 页面：结构视图 + 执行状态视图，定时刷新                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 依赖与相关项目 / Dependencies

| 组件 | 说明 |
|------|------|
| **CARLA** | 仿真环境（文中使用 0.9.15，与 op_bridge 搭配时需注意版本，op_bridge 官方支持 0.9.13）。 |
| **Scenario Runner** | 执行 OpenSCENARIO 1.x (.xosc)，解析 Storyboard 并驱动 CARLA 中的交通参与者。 [CARLA Scenario Runner](https://github.com/carla-simulator/scenario_runner) |
| **op_bridge** | **OpenPlanner ROS 桥接**：连接 CARLA 与 Autoware/OpenPlanner，提供 agent 与 ROS 2 的映射。 **[op_bridge](https://github.com/hatem-darweesh/op_bridge)** 支持 CARLA 0.9.13、Scenario Runner、Python 2.7/3 与 ROS 1；与 Autoware/ROS 2 的集成可参考其 `op_scripts/run_srunner_agent.sh` 及 ROS 2 移植版本（如 `run_srunner_agent_ros2.sh`）。 |
| **Autoware Universe** | 自动驾驶栈（感知、规划、控制），与 CARLA 通过 op_bridge 闭环。 |
| **ROS 2** | 用于状态日志话题 `/scenario_runner/log` 与可视化订阅。 |

环境变量示例（与 op_bridge 文档一致，路径请按本机修改）：

```bash
export CARLA_ROOT=/path/to/carla
export SCENARIO_RUNNER_ROOT=/path/to/scenario_runner
export LEADERBOARD_ROOT=/path/to/op_bridge   # 或本仓库
export TEAM_CODE_ROOT=/path/to/op_agent
export PYTHONPATH=$PYTHONPATH:${CARLA_ROOT}/PythonAPI
# ... 其余见 op_bridge README
```

---

## 本仓库内容 / Repository Contents

| 文件 | 说明 |
|------|------|
| **open_scenario.py** | 基于 [CARLA scenario_runner](https://github.com/carla-simulator/scenario_runner) 中 `srunner/scenarios/open_scenario.py` 的**上游版本**（或兼容接口）。用于将 OpenSCENARIO 转为 py_trees 行为树，并用 **StoryElementStatusToBlackboard** 把 Story/Act/Scene/Maneuver/Event/Action 的状态写入 Blackboard。修改版会在此基础上增加 **ROS 2 日志**（见下方「与上游的差异」）。 |
| **xosc_html_visualizer.py** | OpenSCENARIO **HTML 可视化**：解析 .xosc 得到层次结构，订阅 `/scenario_runner/log`，维护 FSM 状态并生成 HTML（结构视图 + 执行状态视图，支持 IDLE/RUNNING/COMPLETED/CANCELLED 颜色区分，定时刷新）。 |
| **README_changed open-scenario.md** | 对 `open_scenario.py` **修改版与上游的差异说明**（英文）：ROS2 logger、终止去重、Init 精简（仅控制器+速度）、移除天气与演员增删等。 |

---

## 与上游 open_scenario.py 的差异 / Differences from Upstream

本仓库中的 **open_scenario.py** 为上游 scenario_runner 的版本；在实际联合仿真中通常会使用一个**修改版**，主要变化包括：

1. **StoryElementStatusToBlackboard**
   - 增加 `ros2_logger = get_logger()`，在 `initialise()` 中打 `RUNNING`，在 `terminate()` 中打 `END`/`CANCEL` 等 JSON 到 ROS 2。
   - 终止时用 blackboard 的 `termination_flag` 去重，避免重复日志。

2. **Init 行为**
   - 仅保留 **ControllerAction** 与 **初始速度**；移除 LateralAction、RoutingAction、天气行为、EntityAction 等（由外部或 op_bridge 侧处理）。

3. **移除**
   - `_create_weather_behavior()`、`_initialize_actors()` 等，不再在本文件中处理天气与演员增删。

详细对照见 **[README_changed open-scenario.md](README_changed%20open-scenario.md)**。

---

## 运行流程示例 / How to Run

以下步骤需在**多个终端**中依次执行（先启动 CARLA，再启动 Scenario Runner，再启动 op_bridge/agent，最后启动可视化）。

**1. 启动 CARLA**
```bash
cd ~/CC/carla_0.9.15 && make launch
```

**2. 设置环境变量并启动 Scenario Runner**（新开终端，加载 OpenSCENARIO 文件，例如 `CyclistCrossing.xosc`）
```bash
export CARLA_ROOT=${HOME}/CC/carla_0.9.15
export PYTHONPATH="${CARLA_ROOT}/PythonAPI/carla/":${PYTHONPATH}
export PYTHONPATH=$PYTHONPATH:${CARLA_ROOT}/PythonAPI/carla/dist/carla-0.9.15-py3.10-linux-x86_64.egg
cd ~/CC/op_carla/scenario_runner
python3 scenario_runner.py --openscenario srunner/examples/CyclistCrossing.xosc --timeout 1600
```

**3. 启动 op_bridge 与 Autoware 端 agent**（新开终端，连接 CARLA 与 Autoware）
```bash
source /opt/ros/humble/setup.bash
source ~/CC/autoware/install/setup.bash
. ~/CC/op_carla/op_bridge/op_scripts/run_srunner_agent_ros2.sh
```

**4. 启动 HTML 可视化**（新开终端，订阅 `/scenario_runner/log` 并生成/刷新 HTML）
```bash
python3 ~/CC/op_carla/scenario_runner/srunner/tools/xosc_html_visualizer.py ~/CC/op_carla/scenario_runner/srunner/examples/CyclistCrossing.xosc
```
在浏览器打开生成的 `xosc_fsm.html`（脚本会输出路径；或在该目录执行 `python -m http.server 8000` 后访问），即可看到静态结构与实时执行状态。

> 其他场景：将上述 `CyclistCrossing.xosc` 替换为 `FollowLeadingVehicle.xosc`、`PedestrianCrossingFront.xosc`、`IntersectionCollisionAvoidance.xosc` 等即可。

---

## 场景与 ADAS 验证 / Scenarios & ADAS Validation

论文与实验中使用的代表性 .xosc 场景包括：

- **FollowLeadingVehicle.xosc** / **OscControllerExample.xosc** — ACC 跟车
- **CyclistCrossing.xosc** / **PedestrianCrossingFront.xosc** — AEB/VRU 避让
- **IntersectionCollisionAvoidance.xosc** — 交叉口冲突

通过 FSM 可将「前车减速」「行人/自行车开始横穿」「 ego 制动」等语义阶段与 OpenSCENARIO 的 Event/Action 对齐，便于阶段化 KPI 与回归测试。

---

## 参考文献与链接 / References

- 论文：*From State Machines to Traffic Safety: Benchmarking Driver Assistance Systems Using Structured Scenario Models* (Y. Gao, M. Cai, J. Betz, TUM).
- 公开仓库：[https://github.com/MengchenCAI/carla-autoware-scenario-fsm](https://github.com/MengchenCAI/carla-autoware-scenario-fsm)
- **OpenPlanner CARLA–Autoware 桥接**：[**op_bridge**](https://github.com/hatem-darweesh/op_bridge) — OpenPlanner ROS bridge for CARLA and Scenario Runner.
- [CARLA Scenario Runner](https://github.com/carla-simulator/scenario_runner) — OpenSCENARIO support.
- [ASAM OpenSCENARIO](https://www.asam.net/standards/detail/openscenario/) — 场景描述标准.

---

## 许可证 / License

本仓库中的 `open_scenario.py` 遵循 CARLA scenario_runner 的原始许可（如 MIT）；其余代码与文档的许可见仓库根目录 LICENSE 文件（如有）。
