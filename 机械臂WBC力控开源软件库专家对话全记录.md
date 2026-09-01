# 机械臂 WBC 力控开源软件库专家对话全记录

> 本文档为"机械臂 WBC（Whole-Body Control）力控开源软件库选型"主题的专家对话全记录，涵盖：WBC 力控开源库全景介绍、轮式双臂人形机器人选型、ARC-OPT 与 qm_door 适用性分析、速度级/力矩级 WBC 实现机制对比、完整框架学习路径推荐、mc_rtc 任务库种类与 WBC 构造方式详解。所有问答内容均按对话原文整理输出。

[TOC]

## 1. 机械臂 WBC 力控开源软件库咨询

本章为咨询开端，用户询问机械臂 WBC 力控的开源软件库，助手按"完整 WBC 力控框架 / 底层依赖 / 选型建议"三个层次给出全景回答。

**用户**：
有没有机械臂wbc力控的开源软件库？

**助手**：
有的，机械臂/机器人的 WBC（Whole-Body Control）力控开源库生态比较成熟，按用途分类如下：

### 完整 WBC 力控框架（可直接用于真实机器人）

| 库 | 维护方 | 语言 | 特点 |
|---|---|---|---|
| [mc_rtc](https://github.com/jrl-umi3218/mc_rtc) | CNRS-AIST (JRL) | C++ / Python | 实时全身控制框架，任务库丰富（质心、姿态、接触力、末端力等任务），文档和教程最全，支持机械臂、人形、双臂协同，配 mc_rtc_ros / mc_udp，上手最平滑 |
| [TSID](https://github.com/stack-of-tasks/tsid) | INRIA | C++（有 Python 绑定） | 任务空间逆动力学，基于 Pinocchio，直接优化出接触力 + 关节力矩，学术引用多，适合做控制算法研究 |
| [OpenSoT + iDynTree + XBot2](https://github.com/ADVRHumanoids) | IIT 意大利理工 | C++ | 完整实时控制栈（CartesI/O 任务接口），HyQ 等机器人用过 |
| [OCS2](https://github.com/leggedrobotics/ocs2) | ETH Zürich | C++ / Python | MPC + WBC 一体，足式机器人为主（ANYmal、Unitree 开源版），框架通用、可配到臂 |
| [Drake](https://github.com/RobotLocomotion/drake) | Toyota Research Institute | C++ / Python | 含 `InverseDynamicsController`、全身动力学与 QP 求解，偏仿真与规划，也有真实硬件接口 |
| [mjctrl (MuJoCo MPC)](https://github.com/google-deepmind/mujoco_mpc) | Google DeepMind | C++ / Python | 2024 年开源，全身 MPC 力控，效果很好，最适合在 MuJoCo 仿真里快速验证，也能接真实硬件 |

### 底层依赖（通常配套使用）

-   [Pinocchio](https://github.com/stack-of-tasks/pinocchio) — INRIA，刚体动力学/运动学，WBC 的数学底座
-   [Crocoddyl](https://github.com/loco-3d/crocoddyl) — INRIA，接触轨迹优化与 MPC，配 Pinocchio
-   QP 求解器：[qpOASES](https://github.com/coin-or/qpOASES)、[OSQP](https://github.com/osqp/osqp)、[ProxQP](https://github.com/Simple-Robotics/proxsuite)

### 选型建议

-   **想快速上手、文档好、带机械臂/双臂力控例子** → mc_rtc（最推荐）
-   **做算法研究、发论文** → TSID 或 Crocoddyl + Pinocchio
-   **在仿真里验证效果** → MuJoCo MPC 或 Drake
-   **足式机器人 + MPC** → OCS2

一个提醒：WBC 通常指"全身（含基座/多臂）协同"的力控；如果你只是**单臂**的力控（阻抗/导纳控制），那用 ros2_control 的阻抗控制器或 franka_ros 这类轻量方案更合适，不需要上 WBC 框架。

---

## 2. 轮式双臂人形机器人选型

本章聚焦用户的具体场景——轮式双臂人形机器人，助手给出首选方案 mc_rtc、备选路线对比，以及"基座与臂分层"的工程实践建议。

**用户**：
如果是轮式双臂人形机器人，用哪种库合适？

**助手**：
轮式双臂人形（浮动基座 + 双臂力控 + 移动操作）这个场景，最合适的是 **mc_rtc**，理由如下：

### 首选：mc_rtc

1.  **浮动基座支持是"一等公民"**：mc_rtc 直接支持 free-flyer（6 自由度自由基座）和 wheeled 基座建模，轮式人形是它常见的应用对象（其开发单位 CNRS-AIST 就有轮式双臂机器人实验平台，社区里 TIAGo、轮式 HRP 等都有公开配置/示例）。
2.  **双臂任务库成熟**：姿态任务、质心任务、末端力任务、接触力任务（CoMTask 类）都可以多任务优先级叠加，双臂搬运/推拉/阻抗行为很好组合。
3.  **实时性**：C++ 内核实时调度（RTT 组件），配 EtherCAT 等总线即可上真机；也提供 MC UDP 接口方便单独写控制器。
4.  **上手成本低**：教程文档完善，YAML 配置机器人，Python API 可快速仿真验证。

### 备选路线（按你的目标）

| 场景 | 推荐 |
|---|---|
| 发论文、研究全身动力学优化 | **TSID**（浮动基座 + 接触力 + 关节力矩一体化 QP）|
| 足式为主、顺带移动操作 | OCS2（但轮式场景略重）|
| 仿真验证全身 MPC | MuJoCo MPC / Drake |
| **工程量产、不追求严格 WBC** | **ROS 2 + MoveIt 2 + ros2_control 阻抗/导纳控制器** |

### 工程实践建议

轮式双臂做力控，业界（丰田 HSR、PAL TIAGo 这类产品）通常的务实做法是：

-   **基座与臂分层**：轮式基座用独立的运动学控制（差速/全向平台速度控制），双臂用笛卡尔阻抗/导纳控制，中间用"基座运动补偿"把基座漂移反馈给臂的参考系 —— 不一定需要把基座动力学放进同一个 WBC 优化里。
-   **如果要求全身协调**（比如推重物时基座主动配合发力、或双臂+基座同时规划），那就用 mc_rtc 的浮动基座 + 接触力任务，把轮地接触建模成单向接触约束即可。

补充一个实操细节：mc_rtc 里轮式基座通常建模为 `free flyer` 基座 + 轮地接触约束（wheeled 基座在 mc_rbdyn 中有专门支持），这样 WBC 求解出的基座加速度能自然包含轮子驱动力，比"基座位置 PID + 臂阻抗"的硬拼接效果好。

需要的话，我可以帮你搭一个 mc_rtc 的轮式双臂最小示例（机器人描述 + 力控任务配置）。

---

## 3. ARC-OPT 与 qm_door 开源库适用性分析

本章针对用户点名询问的 ARC-OPT、qm_door 两个库，助手通过 GitHub 在线查证后给出准确结论：ARC-OPT 是 DFKI 的通用 WBC 库（合适），qm_door 是特定机器人的 OCS2 参考项目（不宜直接使用）。

**用户**：
ARC-OPT，qm_door等开源库合适吗？

**助手**：
两个库都确认了，信息如下：

### ARC-OPT（ARC-OPT/wbc）— 合适，是正经的 WBC 力控库

**来源**：德国 DFKI 不来梅机器人创新中心（Robotics Innovation Center）开发的通用 WBC 库，JOSS 论文（Mronga & Kirchner 2024），BSD-3-Clause，C++，持续维护（CI 覆盖 Ubuntu 20.04/22.04/24.04），88 stars。

**定位**：把控制问题写成 QP 的代价/约束，输出 **velocity / acceleration / torque 三档**关节指令；官方明确声明面向"冗余机器人，特别是浮动基座系统（人形、四足）"。

**对轮式双臂的适配点**：

-   浮动基座（free-flyer）是一等支持；DFKI 自己就用它控制 **SherpaTT**（四轮平台 + 机械臂）这类轮式机器人，还有 RH5 人形
-   支持**闭链结构**（ICRA 2022 论文），如果腿/基座带闭链连杆有用
-   模型层可插拔（pinocchio 等），QP 求解器可插拔（qpoases 等），有独立的 ROS 2 接口（wbc_ros）

**注意**：纯 C++、生态比 mc_rtc 小、文档以 tutorials 为主；高层任务（质心、接触力、阻抗等）需要按它的 API 自己组装，上手门槛高于 mc_rtc。适合"要透明、要研究级控制、愿意写 C++"的场景。

### qm_door — 不合适直接使用，但值得当参考

**来源**：`danisotelo/qm_door`，个人项目（2024，93 stars，BSD-3-Clause），基于 **OCS2** 实现的 **Unitree AlienGo（四足）+ Z1 单臂**的 MPC + WBC 控制器，专做开门（door-opening）、爬楼梯，带 YOLOv8 门把手检测。

**为什么不能直接用**：

-   它是针对**特定机器人**（四足 + 单臂）的完整项目，不是通用库；没有轮式基座、没有双臂
-   基本是存档性质（2024-09 创建后几乎没有更新），个人维护，不适合做工程基础

**它的价值**：如果你打算走 OCS2 路线做自己的移动操作机器人，它是"腿/基座 + 臂协同的 MPC 规划 + WBC 力控"最完整的开源参考实现之一——可以直接抄它怎么把操作任务（门）写进 MPC 成本、怎么切 MPC/WBC 模式。

### 结论（结合你的轮式双臂人形）

| 方案 | 评价 |
|---|---|
| **mc_rtc** | 最省事：轮式浮动基座 + 双臂任务库现成、文档最全、有 Python API，做产品原型首选 |
| **ARC-OPT/wbc** | 合适且正规：轮式基座 + 双臂没问题（DFKI SherpaTT 就是先例），要 C++ 集成、自建任务；比 mc_rtc 透明、可控 |
| **qm_door** | 别当库用；若选 OCS2 路线，作为 MPC+WBC 的参考实现学习 |
| OCS2 本身 | 通用框架，配轮式基座可行，但轮式双臂的开箱示例少，需要自己搭 |

一句话：**工程落地选 mc_rtc，研究/自研控制内核选 ARC-OPT，qm_door 只当 OCS2 的参考代码**。ARC-OPT 的 wbc 库里就有 SherpaTT 的机器人模型和示例，你可以先跑它的轮式示例感受一下。

---

## 4. 速度级 WBC 与力矩级 WBC 的实现机制

本章回答"速度级/力矩级 WBC 在 mc_rtc 与 ARC-OPT 中是否都能实现"：结论是都能实现，但机制不同——ARC-OPT 原生支持三层级，mc_rtc 力矩级原生、速度级需间接实现。

**用户**：
轮式双臂人形机械臂的速度级WBC和力矩级WBC，都可以通过mc_rtc，ARC-OPT都可以实现吗？

**助手**：
查证完毕，两个库的输出层级机制确实不同，分别说明：

### 结论：都能实现，但机制不一样

#### ARC-OPT — 原生支持三种层级

这是它的设计核心，README 明示提供 **velocity / acceleration / torque 三个层级的 WBC 实现**（对应 `VelocityLevelWBC`、`AccelerationLevelWBC`、`TorqueLevelWBC`）：

-   **速度级 WBC**：QP 的决策变量是关节速度，输出关节速度命令（需配位置内环执行）
-   **力矩级 WBC**：QP 直接优化广义力，输出关节力矩（需力矩/电流环硬件）
-   还附带一个优势：模块化设计，同一套任务描述可以切换层级和 QP 求解器做对比

#### mc_rtc — 力矩级原生，速度级"间接实现"

mc_rtc 的内核（基于 Tasks 库）是**加速度级 QP + 逆动力学 → 输出力矩**，默认执行器命令就是力矩（effort 模式）：

-   **力矩级**：✅ 原生。`DynamicsConstraint` 把动力学方程纳入 QP，任务库（`EndEffectorTask`、`AdmittanceTask`、`CoMTask` 等）定义在加速度/力层面，最终经逆动力学输出关节力矩，这是 mc_rtc 的标准用法
-   **速度级**：⚠️ 没有独立的 velocity-level QP 求解器，但可以通过以下方式达到速度级控制效果：
    -   任务级：把任务配成速度跟踪（低 stiffness、damping 主导，如 `EndEffectorTask` 的 velocity 项）
    -   约束级：接触约束支持 `contactType: velocity`（我查了官方 JSON 文档，`ContactConstraint` 有 acceleration/velocity/position 三档），还有 `BoundedSpeedConstr` 限制速度
    -   输出级：若机器人只支持速度模式，需要外环把 mc_rtc 的力矩/加速度命令转换成速度指令（或用接口层的速度模式）

### 对轮式双臂的实操建议

| 需求 | 选型 |
|---|---|
| 真机有**力矩控制接口**（电流环） | 两者都行；mc_rtc 力矩输出直接可用，ARC-OPT 用 `TorqueLevelWBC` |
| 真机只有**速度/位置接口**（多数轮式平台的现状） | ARC-OPT `VelocityLevelWBC` 更直接；mc_rtc 需要自己做加速度→速度的转换/内环 |
| 双臂力控任务多、要快速开发 | mc_rtc（任务库最全，力矩级开箱即用） |
| 研究级对比不同 WBC 层级 | ARC-OPT（同任务跨层级对比是它的卖点） |

**一个关键提醒**：轮式平台 + 双臂，实务上通常**不是**把"速度级 vs 力矩级"当成二选一——而是**分层**：双臂末端力控用力矩级（阻抗/导纳），基座轮子用速度级（运动学控制），两者通过基座运动补偿耦合。mc_rtc 和 ARC-OPT 都允许这种混搭（浮动基座关节可以单独配置控制层级），但 mc_rtc 把双臂力控任务写起来更省事。

如果你的轮式平台驱动是**速度/位置模式**（大多数商用 AGV 底盘），建议：**ARC-OPT 的速度级 WBC 做基座 + mc_rtc 的力矩级任务做双臂**这种组合，或者干脆全部用 ARC-OPT 统一在速度级、末端力控用导纳外环实现。

---

## 5. 完整 WBC 力控框架学习选型

本章回答学习选型问题：推荐 mc_rtc，并给出从在线演示到源码对照的完整学习路径。

**用户**：
如果要学习完整 WBC 力控框架，推荐哪个库？

**助手**：
直接给结论：**学习用 mc_rtc，没有悬念**。理由和学习路径如下。

### 为什么是 mc_rtc

| 维度 | mc_rtc | ARC-OPT | TSID |
|---|---|---|---|
| 教程体系 | 官方分三档：introduction → recipes → advanced，覆盖安装、控制器编写、任务定制、FSM | 有 tutorials 和论文，但资料量少一个量级 | 基本只有论文 + Doxygen |
| 上手门槛 | 有 **Python API** + **在线演示站**（mc-rtc-demo.netlify.app，浏览器里直接跑，零安装） | 纯 C++，需要自己编依赖（有一键脚本但问题排查靠自己） | C++ + 源码编译，门槛最高 |
| 概念覆盖度 | 一个框架里能看到 WBC 全部概念：任务（位置/姿态/CoM/接触力/阻抗/导纳）、约束（动力学/接触/碰撞）、FSM、状态观测器 | 三种层级 WBC 实现，但任务种类少 | 只覆盖任务空间逆动力学这一块 |
| 社区/资料 | GitHub issues 活跃，网上问答多，demo 多 | 小众（DFKI 内部驱动） | 学术向，文献多但教程少 |

关键点：mc_rtc 是"**完整框架**"——控制内核（Tasks 库的 QP）+ 仿真接口（mc_mujoco / mc_vrep / mc_rtc_ros）+ 真机接口（mc_udp / mc_franka）+ 在线演示，学它你能看到 WBC 从"任务定义 → QP 求解 → 力矩输出 → 执行器"的全链路，而不只是其中一段。

### 推荐学习路径

1.  **先玩**：打开 mc-rtc-demo.netlify.app，浏览器里拖动 JVRC1/双臂机器人，感受任务切换和 FSM 状态机
2.  **再装**：Ubuntu 上用官方 install 脚本（或 conda），装好后跑 `mc_rtc_ticker` + mc_mujoco 里的官方示例
3.  **系统学**：官方 tutorials 按顺序——
    -   introduction：理解 `MCController`、`MetaTask`、约束（constraints）三个核心概念
    -   recipes：学常见场景写法（双臂搬运、接触力任务、阻抗任务）
    -   advanced：自定义任务、自定义求解器、多机器人协调
4.  **补数学**：mc_rtc 基于 SpaceVecAlg/RBDyn（sva 库），配合 sva_rbdyn_tutorials 补刚体动力学和 SE(3) 表示
5.  **进阶对照**（学到一定程度后）：
    -   读 **ARC-OPT 源码**的三种 WBC 层级实现，对比速度级/加速度级/力矩级 QP 的区别——这是 mc_rtc 不直接展示的部分
    -   读 **TSID 相关论文**（Del Prete 等）理解任务空间逆动力学的理论细节

### 提醒

mc_rtc 的 WBC 是"加速度级 + 逆动力学"这一个范式，它的教程不会教你其他范式。所以把它当**入门和工程框架**学，等理解了 QP 任务公式化的思路后，再通过 ARC-OPT/TSID 横向扩展速度级、力矩级和 MPC 的视野——这样你的知识结构是完整的，而不是只会一个库。

---

## 6. mc_rtc 任务库种类与 WBC 构造方式详解

本章为最后一轮问答：基于官方 JSON 文档与源码查证，给出 mc_rtc 完整任务库清单（按功能分类）与 WBC 构造方式（加权 / 两级硬软 / 层级 / 任务动力学 / 反馈闭环）。

**用户**：
mc_rtc这个软件库，有哪些任务库种类？以及哪些种类的WBC构造（加权WBC，层级WBC等）？

**助手**：
查证完毕。mc_rtc 官方 JSON 文档给出了完整的任务清单，源码确认了求解器构造。分两部分回答：

### 一、任务库种类（官方 MetaTask 清单，按功能分类）

| 类别 | 任务 |
|---|---|
| **基础运动学** | `PositionTask`（位置）、`OrientationTask`（姿态）、`TransformTask`（位姿）、`VectorOrientationTask`（矢量方向）、`EndEffectorTask`（末端综合：位置+姿态+速度+力目标）、`RelativeEndEffectorTask`（相对参考系末端） |
| **质心/动量** | `CoMTask`（质心）、`MomentumTask`（动量） |
| **接触与力** | `CoPTask`（压力中心/ZMP）、`AddContactTask` / `RemoveContactTask`（接触平滑过渡）、`ImpedanceTask`（阻抗）、`AdmittanceTask`（导纳）、`ComplianceTask`（柔顺）、`FirstOrderImpedanceTask`（一阶阻抗） |
| **稳定** | `StabilizerTask`（LIPM 稳定器，CoM+ZMP 联合控制，人形站立标配）、`PostureTask`（关节姿态/回中） |
| **轨迹** | `BSplineTrajectoryTask`、`ExactCubicTrajectoryTask` |
| **注视/视觉** | `GazeTask`、`LookAtTask` / `LookAtFrameTask` / `LookAtTFTask`、`PositionBasedVisServoTask`（视觉伺服） |

配套的**约束集**（ConstraintSet）：`DynamicsConstraint`（动力学/逆动力学）、`KinematicsConstraint`（运动学）、`ContactConstraint`（接触，且 `contactType` 支持 acceleration/velocity/position 三档）、`CollisionsConstraint`（碰撞避免）、`CompoundJointConstraint`（复合关节限位）、`BoundedSpeedConstr`（速度界限）、`CoMIncPlaneConstr`（CoM 平面约束）。

### 二、WBC 构造方式

**1. 加权 WBC（weighted）—— 默认且唯一开箱即用的范式**

mc_rtc 的架构是：所有任务（MetaTask）作为 QP 目标函数的**加权代价项**（每个任务有 `weight`、`dimWeight` 维度权重），所有约束作为**硬等式/不等式**。源码确认了两个后端都是加权求解：

-   Tasks 后端（`TasksQPSolver`，基于 Tasks 库）
-   TVM 后端（`TVMQPSolver`，内部用 `tvm::scheme::WeightedLeastSquares`，源码直接可见）

**2. 两级硬/软结构（constraints > tasks）**

约束（动力学、接触、碰撞、限位）优先级恒高于任务，任务之间才靠权重权衡。这本质上是一种"硬约束 + 软任务"的两级层级。

**3. 层级 WBC（hierarchical）—— 不直接提供，但底层可达**

-   mc_rtc 主框架**没有**现成的 `HierarchicalQP` 类（已验证 `include/mc_solver/` 下该头文件不存在）
-   但底层 TVM 库本身就提供 `tvm::scheme::HierarchicalLeastSquares`（TVM 的 scheme 目录下只有 `WeightedLeastSquares.h` 和 `HierarchicalLeastSquares.h` 两个方案），且 mc_rtc 的 `TVMQPSolver` 把内部 `problem()` 暴露了出来——理论上可以自行装配层级求解，但没有官方封装，需要自己写
-   工程上 mc_rtc 用户更常用**状态机（FSM）+ 任务按阶段切换**的方式来处理优先级，而不是真正的 lexicographic 求解

**4. 任务动力学（task dynamics）**

每个任务带 `stiffness` / `damping`（一阶/二阶），决定任务误差的收敛动力学——这是 mc_rtc 任务库的特色机制，也是加权 WBC 调参的核心旋钮。

**5. 反馈闭环方式（FeedbackType）**

`None/OpenLoop`（开环）、`Joints`（编码器反馈）、`JointsWVelocity`（+数值微分速度）、`ObservedRobots/ClosedLoop`（观测器管道闭环）、`SkipQP`。

### 小结

mc_rtc 的 WBC 构造是**"硬约束 + 加权任务"**这一种范式（内部再分 Tasks/TVM 两个后端，可插拔 QP 求解器）；**层级 WBC 不在开箱范围内**。如果你需要真正的 lexicographic 层级 WBC，mc_rtc 这条路线要用 TVM 库自行装配 `HierarchicalLeastSquares`；或者干脆选择 ARC-OPT——它的层级化（约束优先级/任务优先级）是内置设计。这也是两者在"WBC 构造种类"上的实质区别。
