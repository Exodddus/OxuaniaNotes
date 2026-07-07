# CaP-X: A Framework for Benchmarking and Improving Coding Agents for Robot Manipulation

## 基本信息

- 论文: CaP-X: A Framework for Benchmarking and Improving Coding Agents for Robot Manipulation
- 作者: Max Fu, Justin Yu, Karim El-Refai, Ethan Kou, Haoru Xue, Huang Huang, Wenli Xiao, Guanzhi Wang, Fei-Fei Li, Guanya Shi, Jiajun Wu, Shankar Sastry, Yuke Zhu, Ken Goldberg, Linxi "Jim" Fan
- 机构: NVIDIA, UC Berkeley, Stanford University, Carnegie Mellon University
- 时间: arXiv v1, 2026-03-23
- 项目页: https://capgym.github.io
- 关键词: Code-as-Policy, coding agents, robot manipulation, CaP-Gym, CaP-Bench, CaP-Agent0, CaP-RL, test-time computation

## 一句话总结

CaP-X 是一个系统化研究机器人代码代理的框架：它先用 CaP-Gym 把机器人操作暴露成可执行代码环境，再用 CaP-Bench 从抽象层级、交互轮次和视觉 grounding 三个维度评测 12 个前沿模型，发现模型严重依赖人工高层 API；随后提出 CaP-Agent0 和 CaP-RL，证明多轮反馈、视觉差分、自动技能库、并行推理和 RLVR 可以显著提升低层 primitives 上的机器人控制代码生成能力。

## 核心问题

Code-as-Policy 的直觉很吸引人：让 LLM / coding agent 写程序，把感知、几何、规划和控制 primitives 组合起来，像人类工程师一样控制机器人。但之前很多工作有一个问题：它们给了模型很强的人造高层接口，比如 `stack_objs_in_order()`，因此很难判断成功到底来自模型推理，还是来自人类已经把任务结构写进 API 里。

CaP-X 想回答三个问题：

1. 当前 coding agents 在机器人操作中到底有多强？
2. 当去掉人类设计的高层 abstractions，只给低层 perception/control primitives 时，性能会掉多少？
3. 多轮交互、执行反馈、视觉 grounding、自动技能库和 RL 能否弥补低层 primitives 带来的困难？

这篇文章的目标不是提出一个单一机器人策略，而是给 embodied coding agents 建立一套评测和改进的实验平台。

## 方法总览

CaP-X 包含四个部分：

| 组件 | 作用 |
|---|---|
| CaP-Gym | 机器人 coding agent 的交互式环境，把仿真或真实机器人接到代码执行器 |
| CaP-Bench | 评测模型在不同抽象层级、交互模式、视觉 grounding 下的能力 |
| CaP-Agent0 | 训练-free 的增强型 coding agent 框架 |
| CaP-RL | 直接对 coding agent 做带可验证奖励的强化学习后训练 |

CaP-X 的核心思想是：把机器人控制看成“生成、执行、观察、修复程序”的过程。机器人不再只是执行神经网络动作，而是执行模型写出的程序。程序里可以调用 segmentation、pointing、grasp planning、IK、trajectory optimization、gripper control 等工具。

## CaP-Gym

CaP-Gym 是一个基于 Gymnasium 思路的层级控制环境。它把两层循环绑定在一起：

- Low-Level Environment loop：物理仿真或真实机器人环境。
- Stateful Code Executor loop：执行 coding agent 生成的 Python 程序。

CaP-Gym 集成了 187 个机器人操作任务，来源包括：

- RoboSuite。
- LIBERO-PRO。
- BEHAVIOR。

它提供一套既能在仿真里跑，也能接真实机器人的 shared primitive design。常见低层接口包括：

- `get_observation`。
- SAM3 文本或点提示分割。
- Molmo 2 open-vocabulary pointing。
- grasp planning。
- oriented bounding box。
- IK / trajectory optimization。
- 单臂和双臂 joint control。
- gripper open / close。

这一层的价值在于把机器人操作变成一个可验证的代码执行问题：程序能不能编译、能不能调用对 API、机器人有没有完成任务，都可以被环境记录和反馈。

## CaP-Bench

CaP-Bench 评测 12 个 open/closed-source 语言或视觉语言模型，包括 Gemini-3-Pro、GPT 系列、Claude、Qwen、Kimi、DeepSeek 等。评测围绕三个轴：

1. Abstraction Level：高层人工 API vs 低层 primitives。
2. Temporal Interaction：single-turn 程序生成 vs multi-turn 交互修复。
3. Perceptual Grounding：无视觉、原始图像、多模态、视觉差分文本。

### Single-turn tiers

| Tier | 特点 |
|---|---|
| S1 | high-level primitives + privileged state |
| S2 | high-level primitives + noisy perception |
| S3 | low-level primitives + API usage examples |
| S4 | low-level primitives + only signatures/docstrings |

S1/S2 更像早期 Code-as-Policies：人类把复杂操作封装成高层 API。S3/S4 则更接近真实机器人程序员面对的接口，需要模型自己组合 segmentation、IK、grasp、motion 等步骤。

### Multi-turn tiers

| Tier | 特点 |
|---|---|
| M1 | text-only execution feedback，返回 stdout / stderr |
| M2 | 多轮中直接给当前 RGB 图像 |
| M3 | Visual Differencing Module，把图像变化转成结构化文本 |
| M4 | low-level primitives + VDM + 多轮反馈 |

VDM 是一个关键设计。它不把原始图像直接塞回 coding agent，而是用 VLM 将当前场景、任务相关属性和前后图像差异转成自然语言反馈。论文发现，这比直接多模态图像输入更有效。

## CaP-Bench 的主要发现

### 1. 前沿模型离人类 expert code 仍有明显差距

在 single-turn、zero-shot 设置下，即使闭源模型强于开源模型，也没有模型能在低层 primitives 条件下稳定达到人类专家程序的成功率。

这说明软件 benchmark 上强的 coding agent，直接迁移到机器人控制仍然困难。困难不只是代码语法，而是 embodiment-aware reasoning、空间几何、感知不确定性、动作可达性和失败恢复。

### 2. 高层 abstraction 会显著提高性能，但也掩盖能力不足

从 S4 到 S1，随着 primitives 越来越高级，成功率单调上升。原因很直接：高层 API 缩小了搜索空间，模型只需要做任务排序，而不用自己写低层几何和控制组合。

但这也带来一个评测风险：如果 API 过度工程化，模型看起来很强，实际上只是调用了人类预先设计好的技能。CaP-X 因此主张要重点看低层 primitives 上的表现，才能测出真正的 embodied coding 能力。

### 3. 多轮反馈和视觉差分能显著提升

多轮交互允许模型：

- 根据 stdout / stderr 修代码。
- 打印中间状态。
- 检查分割和 grasp 是否成功。
- 根据执行失败重新规划。

论文发现 M1 通常优于 single-turn，而 VDM 的 M3 进一步优于直接图像输入的 M2。原因是机器人任务中的视觉信息往往需要被转成结构化任务状态，例如“物体已经移动到哪里”“目标是否完成”“是否发生碰撞/掉落”，直接塞图像反而可能分散 coding agent 的注意力。

### 4. 低层 primitives + 多轮反馈可以接近或超过高层 single-turn

M4 在低层 primitives 下加入多轮和 VDM，显著超过 S3，并能达到或超过高层 single-turn S2。这是一个很重要的结论：低层接口本身不一定不可用，关键是给 agent 足够的 test-time computation 和可解释反馈。

## CaP-Agent0

CaP-Agent0 是基于 CaP-Bench 发现设计的 training-free agentic framework，核心组件有三个：

### 1. Multi-turn Visual Differencing

把每轮执行前后的图像变化转成文本描述，告诉 coding agent：

- 初始场景里有什么。
- 当前场景和上轮相比发生了什么。
- 任务是否完成。
- 哪些对象位置或关系发生改变。

这让代码模型更像在读 structured execution log，而不是重新做视觉理解。

### 2. 自动合成持久技能库

CaP-Agent0 从成功 rollouts 中抽取可复用函数，再让 LLM 分析和整理成 task-agnostic skill library。它和人工高层 API 的区别在于：

- skills 是从低层成功代码中发现的。
- 它们保留低层接口的表达力。
- 它们减少重复写脆弱低层逻辑。

可以理解为 agent 自己从经验里长出一些 utility functions。

### 3. 并行推理

在每轮生成时，系统并行采样多个候选方案：

- 单模型版本：同一模型 9 次查询。
- 多模型版本：GPT-5.2、Claude Opus 4.5、Gemini-3-Pro 各 3 次。

中央 agent 再合成最终代码。这个机制把 test-time search 用在机器人控制代码生成上，减少偶然失败。

## CaP-Agent0 实验结果

CaP-Agent0 在 CaP-Bench 上以 100 trials per task 评测。消融结果显示：

- S3 baseline 平均成功率约 24%。
- 加入 M4 / VDM 后约 55%。
- 加 skill library 后约 59%。
- 单模型并行推理后约 66%。
- 多模型并行推理后约 68%。

即使只操作低层 primitives，CaP-Agent0 在 7 个任务中有 4 个达到或超过 human expert code 的 single-turn 表现。

### LIBERO-PRO

作者还在 LIBERO-PRO 上比较 OpenVLA、π0、π0.5 和 CaP-Agent0。平均结果中：

| 方法 | libero-object Pos | libero-object Task | libero-goal Pos | libero-goal Task | libero-spatial Pos | libero-spatial Task |
|---|---:|---:|---:|---:|---:|---:|
| OpenVLA | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 |
| π0 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 |
| π0.5 | 0.17 | 0.01 | 0.38 | 0.00 | 0.20 | 0.01 |
| CaP-Agent0 | 0.22 | 0.18 | 0.26 | 0.17 | 0.12 | 0.14 |

VLA 在 instruction perturbation 下几乎失效，而 CaP-Agent0 对指令变化更稳。它在 position perturbation 上接近 π0.5，在 task perturbation 上明显更强。

### BEHAVIOR mobile manipulation

在 R1Pro 轮式 humanoid 上执行两个长程 mobile manipulation 任务：

| 任务               | Human Nav | S3 Nav | CaP-Agent0 Nav | Human Task | S3 Task | CaP-Agent0 Task |
| ---------------- | --------: | -----: | -------------: | ---------: | ------: | --------------: |
| Pick up Radio    |       88% |    72% |            80% |        36% |     24% |             56% |
| Pick up Soda Can |       80% |    52% |            84% |        72% |     32% |             72% |

CaP-Agent0 的优势主要来自闭环修正：丢失视野时重新定位，抓取失败后重新采样 grasp pose，小物体定位不准时调整搜索策略。

### 真实机器人

论文还展示了 Franka Panda 和 AgiBot G1 上的真实机器人任务，包括：

- 找出杯子下面的目标物。
- 根据真实场景中的数学题选择正确 block。
- 按常识堆叠不同形状物体。
- 接收人类反馈后修正抓取方式。

这里的重点不是某个 benchmark 数字，而是 CaP-Gym 的代码执行循环能接真实机器人，并且 off-the-shelf VLM 可以在无后训练情况下做长程 reasoning + manipulation。

## CaP-RL

CaP-X 还提出 CaP-RL：直接对 coding agent 做带可验证奖励的强化学习。作者用 GRPO 对 Qwen2.5-Coder-7B-Instruct 后训练，任务包括：

- Cube Lift。
- Cube Stack。
- Spill Wipe。

训练时使用 S1 privileged state APIs 来稳定奖励，避免 noisy perception 带来的 credit assignment 问题。评估时再看 noisy perception 和真实机器人迁移。

结果：

| 方法 | Sim Cube Lift | Sim Cube Stack | Sim Spill Wipe | Real Cube Lift | Real Cube Stack |
|---|---:|---:|---:|---:|---:|
| Human Expert | 93% | 73% | 100% | 92% | 84% |
| Qwen 2.5 Coder 7B | 25% | 4% | 30% | 24% | 12% |
| Qwen w/ CaP-RL | 80% | 44% | 93% | 84% | 76% |

CaP-RL 的一个亮点是 sim-to-real gap 小。因为它优化的是在抽象 API 上写出更合理程序，而不是直接拟合像素动作，所以策略迁移到真实 Franka 时仍有较高成功率。

## 论文贡献

1. 提出 CaP-Gym。  
   把机器人控制环境和代码执行器绑定起来，让 coding agents 可以在仿真和真实机器人上执行并获得反馈。

2. 提出 CaP-Bench。  
   系统评测抽象层级、多轮交互和视觉 grounding 对机器人代码生成的影响。

3. 揭示 coding agents 对人工高层 abstractions 的依赖。  
   高层 API 提高成功率，但会掩盖低层 embodied reasoning 能力不足。

4. 提出 CaP-Agent0。  
   用 VDM、技能库、并行推理等 training-free test-time computation 显著提升低层 primitives 上的表现。

5. 提出 CaP-RL。  
   用可验证环境奖励后训练代码模型，显著改善机器人操作程序生成，并展示真实机器人迁移。

## 局限

1. Programmatic control 仍不擅长高频连续反馈任务。  
   插入、倒液、柔顺接触等需要紧密视觉伺服和连续控制的任务，纯代码组合仍容易脆。

2. 感知和 grasp API 仍是瓶颈。  
   论文在 LIBERO-PRO 中提到，SAM3 可能把 alphabet soup can 分割成 tomato sauce can，遮挡场景中 grasp 和 control 也会失败。

3. 当前 RL 训练主要在 privileged state / S1 上稳定收敛。  
   noisy perception 下直接训练仍有 credit assignment 问题。

4. 评测仍依赖一组给定 primitives。  
   不同 robot tool stack 的设计会显著影响结果，CaP-Bench 测的是“模型 + API + feedback loop”的整体能力。

5. 长程任务中需要更强的任务级约束规划。  
   论文也建议加入 optimization-based control primitives，让 agent 指定约束和碰撞规避，而不是只靠 IK 后插值。

## 和具身智能研究的关系

CaP-X 代表的是具身智能里一条和 VLA 不同的路线：不是把所有东西塞进一个 action model，而是让通用 coding agent 动态编排感知、几何、规划和控制模块。

它和 VLA 的关系不是简单替代，而更像分工：

- VLA 擅长快速、连续、接触丰富的低层执行。
- Coding agent 擅长长程逻辑、任务分解、工具调用、异常恢复和可解释控制。
- 未来可能是 hybrid CaP-VLA：代码代理管任务逻辑和恢复，VLA 处理低层动作。

这篇文章给领域一个很重要的提醒：机器人“通用性”不只来自数据规模，也来自 agentic test-time computation。一个会写代码、会观察执行结果、会修复错误的系统，在长程操作中可以弥补很多端到端模型的弱点。

## 我的理解

CaP-X 最有价值的地方是把 Code-as-Policy 从 demo 拉到 benchmark 和训练范式上。它不是只展示“LLM 会写机器人代码”，而是拆开问：高层 API 帮了多少？多轮反馈帮了多少？视觉用原图好还是文本差分好？技能库和并行推理能不能把低层 primitives 救回来？RL 能不能直接训练写代码的模型？

我觉得它对之后做机器人 agent 很有启发：与其只追求更大的 VLA，不如认真设计 agent-environment interface。代码代理的能力很大一部分来自它能否看到正确的反馈、能否试错、能否把成功经验固化为技能。

## 可继续追的问题

- CaP-Agent0 的自动技能库能否持续在线增长，而不是一次性合成？
- CaP-RL 能否直接在 noisy perception 或真实机器人反馈上稳定训练？
- VDM 的文本反馈是否会遗漏关键视觉细节？能否和结构化 scene graph 结合？
- 对高频接触任务，什么样的 VLA / controller 最适合被 CaP agent 调用？
- CaP-Gym 能否扩展到更多 classical robotics 问题，比如主动感知、双臂协作、装配和导航操作一体化？

## 记忆钩子

CaP-X 可以记成三句话：

1. 先用 CaP-Bench 测清楚机器人 coding agent 到底依赖多少人工 API。
2. 再用 VDM、多轮反馈、技能库和并行推理增强低层 primitives 上的表现。
3. 最后用 CaP-RL 直接训练写机器人程序的模型，让代码策略具备可学习性。
