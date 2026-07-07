# MAESTRO: Orchestrating Robotics Modules with Vision-Language Models for Zero-Shot Generalist Robots

## 基本信息

- 论文: MAESTRO: Orchestrating Robotics Modules with Vision-Language Models for Zero-Shot Generalist Robots
- 作者: Junyao Shi, Rujia Yang, Kaitian Chao, Selina Bingqing Wan, Yifei Shao, Jiahui Lei, Jianing Qian, Long Le, Pratik Chaudhari, Kostas Daniilidis, Chuan Wen, Dinesh Jayaraman
- 机构: University of Pennsylvania
- 时间: PDF 未标注 arXiv 编号；项目页论文版本约 2026 年前后
- 项目页: https://maestro-robot.github.io
- 关键词: VLM coding agent, modular robotics, zero-shot manipulation, Code-as-Policy, tool orchestration, plan-react-replan, generalist robots

## 一句话总结

MAESTRO 走的是一条和大规模 VLA 训练不同的路线：不把所有机器人能力都塞进一个端到端模型，而是让 VLM coding agent 在闭环中动态编排感知、几何、抓取、运动规划、VLA、图像编辑和移动操作模块，从而在没有机器人行为数据训练的情况下，在桌面和移动操作任务上零样本达到甚至超过多个 VLA 和先前 CaP 系统。

## 核心问题

当前通用机器人主流路线是收集大规模 observation-action 数据，训练 VLA。这个路线很强，但也有几个问题：

1. 机器人数据昂贵。  
   遥操作采集成本高，覆盖的物体、任务、环境和 embodiment 都有限。

2. 端到端 VLA 可解释性和可编辑性弱。  
   新硬件、新工具、新任务或少量真实经验通常需要重新收集数据或 fine-tune。

3. 纯 CaP / VLM-as-policy 过去能力不足。  
   早期 Code-as-Policy 通常是 open-loop 或工具太少，能做简单 pick-and-place，但在精细感知、空间推理、复杂抓取、长程失败恢复上明显落后 VLA。

MAESTRO 想回答的是：如果不给 VLM 直接输出低层动作，而是给它一套足够丰富、足够机器人化的模块，并让它在执行中持续观察、写代码、反应和重规划，能否成为一个真正有竞争力的 generalist robot policy？

## 方法总览

MAESTRO 全称是 Managerial Agent for Executing Sensorimotor Tasks in Robotics。它的定位很清楚：VLM 不负责所有底层细节，而是做 manager，负责根据任务和场景动态选择、组合和修复工具调用。

输入包括：

- 系统 prompt。
- 当前场景图像。
- 语言任务指令。
- 机器人状态和工具反馈。

输出是可执行代码。代码调用工具模块完成操作。执行后，系统把图像、stdout、机器人状态和任务进展反馈给 VLM，VLM 再决定继续执行、修复当前子目标，还是重新规划。

这个循环可以概括为：

1. Plan：把任务拆成子步骤，生成第一段代码。
2. React：执行后检查子目标是否完成。
3. Replan：失败则诊断原因并重写代码，成功则进入下一步。

## 模块工具集

MAESTRO 的关键不是“一个更强 prompt”，而是工具集设计。论文明确说，它相比先前 CaP 系统有两个特点：

- 更完整的机器人相关 tool repertoire。
- 更少人为强加的结构约束，让 VLM 有更充分的表达空间。

### 桌面操作平台

桌面实验使用 DROID 风格 setup：

- 7-DoF Franka Emika Panda。
- Robotiq 2F gripper。
- wrist camera。
- third-person camera。

主要工具模块包括：

| 类别      | MAESTRO 工具                                                                                                                     |
| ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 感知      | RGB/proprioception、segmentation centers、pointing、active perception、FoundationStereo depth、VLM-selected task-relevant keypoints |
| 控制      | gripper open/close、move-to、collision-free motion planning                                                                      |
| 学习型策略   | GraspGen grasp model、π0.5 VLA with high-frequency monitor                                                                      |
| 几何与线性代数 | distance、vector construction、vector rotation、relative rotation                                                                 |
| 图像编辑    | draw points、overlay 6D poses                                                                                                   |

### 移动操作模块

移动平台是 Unitree Go2-W wheeled quadruped 加 AgileX PiPER arm 和 wrist camera。

额外工具包括：

- mobile base state estimation。
- active perception：look left/right/ground、view basket、log object location。
- navigation。
- fine-grained nudge。
- put-in-basket 这类 long-horizon transport 工具。
- semantic map，用来缓存已观察到的物体位置。

## 几个关键设计原则

### 1. Coarse-to-fine perception

不同任务需要不同粒度的视觉信息。MAESTRO 给 VLM 多级感知工具：

- 快速粗粒度：原始观测、mask centroid。
- 中等：segmentation / pointing。
- 精细但慢：VLM-selected task-relevant keypoints。
- 主动感知：用 wrist camera zoom / look around 获取更好视角。

这样 VLM 可以自己决定什么时候用快工具，什么时候为精确操作付出更高感知成本。

### 2. 几何工具补足 VLM 空间推理短板

作者发现，只有高级感知工具还不够，VLM 在空间链式推理上仍会出错。于是加入距离、向量、旋转、相对方向等小工具，帮助它把“把紫色面朝上”“让杯把孔对准支架”等任务转成几何约束。

这其实很像给 VLM 配了一套外部 scratchpad：不要让模型在自然语言里想象 3D 关系，而是用可计算的几何函数落地。

### 3. 高频 VLM monitor 让 VLA 成为可调用模块

MAESTRO 可以把 π0.5 这类 VLA 当作工具调用。但 VLA 通常不知道什么时候停，也不擅长在完成或失败后自我中断。

因此系统用本地 Qwen2.5-VL-72B-Instruct 以约 2Hz 判断 yes/no 条件，例如任务是否完成、是否需要停止或重规划。这样 VLA 负责快执行，MAESTRO 负责监控和调度。

### 4. Collision-free motion planning

在 cluttered scenes 里，直接执行目标位姿容易碰撞。MAESTRO 加入基于 point cloud 的 collision-free motion planning，提升物体交互稳定性。

### 5. 运行历史驱动 evolution

系统会记录过去执行：

- 生成代码。
- stdout。
- Gemini 对执行视频的成功/失败分析。

下次同任务运行前，把这些记录作为 in-context examples 提供给 VLM。这样 MAESTRO 可以从少量真实运行中改进代码，不需要重新训练。

## 实验设置

作者评估三个问题：

1. MAESTRO 在不同 embodiment、环境和任务上 zero-shot 表现如何？
2. 哪些设计组件最重要？
3. 能否通过少量真实世界 trials 的 evolution 继续提升？

评测使用 STAR-Gen taxonomy 做系统扰动。每个任务 5 个 trials：

- 1 个初始设置。
- 4 个由 STAR-Gen 生成的变化。

扰动包括：

- task-relevant objects 的视觉变化。
- 物体位置变化。
- action verbs 变化。
- 新的 manipulated objects。

作者不用简单 success/failure，而是给每个任务设计 progress rubric，分阶段打分 0-100。

## 桌面操作结果

7 个桌面任务覆盖多个挑战轴：

- Pick-place：把物体放进碗。
- Deformable object：把毛巾四角折到中心。
- Articulated object：打开柜子。
- Spatial reasoning：把 cube 转到紫色面朝上。
- Tool use：用刀切香蕉。
- Object affordance：把杯子挂到杯架上。
- Memory + long-horizon semantic reasoning：擦掉白板指令，再按指令堆杯子。

对比方法包括：

- Gemini Robotics Agent：类似 CaP 的闭环代码系统，但工具较少。
- π0。
- π0.5。
- MAESTRO。

结果：

| 任务 | Gemini Robotics Agent | π0 | π0.5 | MAESTRO |
|---|---|---|---|---|
| Put item in bowl | 73.3 | 74.0 | 70.0 | 98.0 |
| Fold towel corners | 40.0 | 47.0 | 70.0 | 71.3 |
| Open cabinet | 3.3 | 8.3 | 0.0 | 68.0 |
| Rotate cube purple side up | 23.6 | 29.0 | 10.0 | 60.0 |
| Cut banana with knife | 71.0 | 30.0 | 14.0 | 92.0 |
| Hang mug on mug holder | 46.0 | 59.0 | 80.0 | 69.0 |
| Erase instructions then stack cups | 26.7 | 12.0 | 22.0 | 63.0 |

MAESTRO 在 6/7 个任务上超过所有 baseline。唯一明显落后 π0.5 的是挂杯子任务，主要因为它需要细致理解杯把孔和支架方向之间的 affordance / spatial relation。

论文对结果的解释很清楚：

- VLA 在训练分布接近的 pick-place 上可以不错，但遇到语义扰动、长程记忆、开柜、转 cube 等 OOD 任务会掉。
- Gemini Robotics Agent 会做高层计划，但工具太少，无法定位毛巾角、柜门把手或实现精确几何旋转。
- MAESTRO 的优势来自 VLM 语义推理 + 机器人模块精确执行的组合。

## 移动操作结果

4 个移动操作任务：

- 收集桌上所有玩具。
- 把绿色球扔进垃圾桶。
- 搜索物体并放回桌上。
- 按按钮开门。

结果：

| 任务 | MAESTRO 平均进度 |
|---|---:|
| Collect all toys on table | 85.0 |
| Throw green ball into garbage can | 76.7 |
| Search item and put on table | 96.0 |
| Press button to open door | 93.3 |

移动场景中 semantic map 很重要，因为机器人需要记住物体位置，避免重复搜索。失败主要来自低层执行，例如垃圾桶深度估计不准导致 grasp pose 不可达，或 collision-free path 找不到后 replanning 进入震荡。

## 消融实验

作者重点消融两类模块：

- advanced perception：task-relevant keypoints + active perception。
- geometry modules：距离、向量、旋转等几何工具。

在 Fold Towel 和 Rotate Cube 上：

| 方法                      | Fold Towel | Rotate Cube |
| ----------------------- | ---------: | ----------: |
| MAESTRO                 |       71.3 |        60.0 |
| w/o advanced perception |       40.0 |        25.0 |
| w/o geometry modules    |       67.5 |        42.5 |
|                         |            |             |

结论：

- 毛巾折叠强依赖精细交互点，所以 advanced perception 特别关键。
- cube 旋转强依赖空间几何推理，所以 geometry modules 明显提升。

这组消融说明 MAESTRO 的强不是单靠 VLM，而是 VLM + 合适工具接口的组合。

## Evolution from previous runs

论文展示 open-cabinet 任务的少量真实经验改进：

1. 初始失败 run：MAESTRO 找到把手，但尝试 top-down grasp，只达到约 35% progress。
2. 一次 evolution 后：它开始围绕把手扫描并调用 grasp model，能抓住但沿错误方向拉，约 70.0 progress。
3. 第三次 evolution：它用 handle 和 hinge keypoints 构造向量，计算正确旋转方向，达到约 85.0 progress。

这说明代码策略有一个很实用的优点：失败经验可以通过局部代码和提示上下文更新进入下一次执行，而不是必须重训端到端模型。

## 论文贡献

1. 提出 MAESTRO，一个 VLM-driven modular robot policy。  
   它通过写代码调度感知、几何、控制、抓取、VLA 和移动操作模块。

2. 证明模块化 VLM 代理可以在 zero-shot 操作上竞争甚至超过 VLA。  
   尤其在长程、语义、工具使用、空间推理和 articulated object 任务上优势明显。

3. 系统分析哪些模块重要。  
   advanced perception 和 geometry tools 对复杂任务有明确贡献。

4. 展示跨 embodiment 扩展。  
   同一理念可用于 Franka 桌面机械臂，也可用于四足轮式移动操作平台。

5. 展示从少量真实运行中 evolution。  
   通过记录和复用过去代码、执行输出和失败分析，提升后续任务表现。

## 局限

1. 延迟和计算开销高。  
   每次 react / replan 都需要 VLM API 调用，任务总耗时明显长于端到端 VLA。

2. 精细连续控制仍不足。  
   对非常 delicate 的接触、插入、柔顺控制等任务，现有模块和闭环频率可能不够。

3. 工具集质量决定上限。  
   如果没有合适的 perception、keypoint、grasp 或 geometry tool，VLM 的高层推理很难落地。

4. 某些 affordance / spatial relation 仍会失败。  
   挂杯子任务就是例子，需要理解杯把孔、支架枝干、姿态对齐和执行轨迹。

5. evolution 仍是 in-context / local code 改进。  
   它能用少量经验修补任务策略，但还不是系统性长期学习。

## 和具身智能研究的关系

MAESTRO 明确挑战了“通用机器人只能靠更大 VLA 数据训练”的单一路线。它说明，预训练 VLM 的语义和代码能力，加上机器人社区已有的感知、几何、控制和学习模块，也可以组成强 generalist robot policy。

它和 VLA 的关系不是对立，而是互补：

- VLM agent 负责规划、工具选择、错误诊断、长程记忆和语义推理。
- VLA / grasp model / planner 负责低层连续执行。
- fast monitor 负责把 VLA 变成可中断、可调度的工具。

从具身智能角度看，MAESTRO 强调的是“模块可组合性”和“运行时智能”。它不靠一次训练覆盖所有情况，而是在每个任务现场动态决定用什么工具、如何看、如何动、如何修。

## 和 CaP-X 的关系

CaP-X 更像 benchmark 和训练平台，系统研究 coding agents 在机器人操作中的能力边界；MAESTRO 更像一个实际机器人系统，强调在真实机器人上用丰富模块拿到强 zero-shot 表现。

两者共同指向同一个判断：Code-as-Policy 的关键不只是让 LLM 写代码，而是要给它合适的工具、反馈和闭环接口。

差别在于：

- CaP-X 研究抽象层级、多轮反馈、VDM、技能库和 RL。
- MAESTRO 研究更完整的机器人模块库、active perception、geometry tools、VLA-as-tool 和跨 embodiment 部署。

## 我的理解

MAESTRO 最有意思的地方是它让“传统模块化机器人系统”重新变得可扩展。过去模块化系统强在可解释、可调试、可替换，但弱在需要人类工程师为每个任务手写 glue code。MAESTRO 用 VLM coding agent 替代这个 glue-code 工程师。

我觉得这是一条很现实的路线：机器人社区已经有大量 perception、grasping、planning、control 模块，VLA 也可以作为模块。真正缺的是一个能在任务现场把它们灵活组合起来的 manager。MAESTRO 就是在尝试把 VLM 放到这个位置。

## 可继续追的问题

- MAESTRO 的工具选择能否通过长期经验自动优化，而不是完全靠 VLM prompt？
- fast monitor 能否替换成更专门的 learned verifier，降低成本和延迟？
- 对接触丰富任务，应该给 MAESTRO 什么样的 low-level skill / VLA 接口？
- evolution 能否变成持久技能库或策略库，而不是只把历史 run 作为上下文？
- 如何评估模块化系统和端到端 VLA 在真实开放环境中的长期可靠性和维护成本？

## 记忆钩子

MAESTRO 可以记成三句话：

1. VLM 不直接控制电机，而是写代码调度机器人模块。
2. 工具集要足够丰富：感知、几何、抓取、规划、VLA、active perception 都要能被调用。
3. 通过 plan-react-replan 闭环，它在 zero-shot 长程操作上可以超过很多数据训练型 VLA。
