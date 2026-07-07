# Articraft: An Agentic System for Scalable Articulated 3D Asset Generation

## 基本信息

- 论文: Articraft: An Agentic System for Scalable Articulated 3D Asset Generation
- 作者: Matt Zhou, Ruining Li, Xiaoyang Lyu, Zhaomou Song, Zhening Huang, Chuanxia Zheng, Christian Rupprecht, Andrea Vedaldi, Shangzhe Wu
- 机构: University of Cambridge, University of Oxford, Nanyang Technological University
- 时间: arXiv v1, 2026-05-14
- 项目页: https://articraft3d.github.io
- 关键词: articulated 3D assets, agentic generation, LLM coding agent, URDF, simulation-ready assets, synthetic data

## 一句话总结

Articraft 把“生成一个可动 3D 物体”转化成“让 LLM 写一个构造该物体的程序”：通过一个面向关节化资产的 SDK 和受限执行 harness，让模型迭代编写、编译、测试和修复 `model.py`，批量生成带语义部件、关节、运动范围和 URDF 表示的仿真可用资产，并进一步构建了覆盖 245 类、超过 10K 个对象的 Articraft-10K 数据集。

## 核心问题

具身智能和机器人仿真需要大量可交互、可操作的 3D 物体，但现有 articulated object 数据集有几个明显瓶颈：

1. 类别覆盖窄。  
   PartNet-Mobility、AKB-48、GAPartNet 等数据集规模和类别数量有限，很多真实世界常见的铰链、滑轨、折叠、旋转机构没有充分覆盖。

2. 只建模形状不够。  
   对柜子、抽屉、折叠椅、台灯、工具这类对象来说，功能很大程度来自部件之间如何运动。只生成 mesh，不能支持交互、仿真和机器人操作。

3. 手工建模太贵。  
   高质量 articulated assets 需要建模几何、部件层级、关节轴、运动范围、碰撞结构和物理属性，人工成本高，不适合扩展到大规模数据。

Articraft 想回答的问题是：能不能利用 LLM 的代码生成能力，把 articulated object 的结构先验和程序化建模结合起来，低成本、大规模地产生仿真可用的可动 3D 资产？

## 方法总览

Articraft 的核心判断是：关节化物体天然像程序。生成一个对象可以分解成：

- 识别部件。
- 决定部件之间的连接关系。
- 指定 revolute / prismatic / continuous / fixed joints。
- 生成几何和材料。
- 验证部件是否脱离、重叠、运动是否合理。
- 根据错误反馈修复。

因此，论文没有训练一个端到端 3D 生成模型，而是搭了一个 agentic coding environment。LLM 在这个环境里只做一件事：编辑一个 `model.py`，让它构造出完整 articulated object。

整体由两部分组成：

| 模块 | 作用 |
|---|---|
| Articraft SDK | 提供部件、几何、关节、测试等面向 articulated assets 的程序接口 |
| Articraft harness | 提供受限工作区、编译、QC、结构化反馈和 probe 工具，让 LLM 迭代修复 |

我的理解是，这篇文章真正的贡献不是“LLM 可以写 3D 代码”，而是它把 LLM 的自由度大幅收窄到一个为 articulated asset 设计好的环境里：只写一个文件，只用少数工具，只看结构化失败信号。这让生成过程足够稳定，也足够便宜。

## Articraft SDK

每个资产由一个 Python 程序定义，主要入口是：

- `build_object_model()`：构造 articulated object。
- `run_tests()`：写对象特定的几何和运动测试。

输出资产包含：

- URDF。
- 3D meshes。
- semantic parts。
- articulated joints。
- joint axes。
- motion ranges。

### 几何与部件

SDK 提供从低到高的建模能力：

- 基础几何：box、cylinder、sphere 等。
- CAD-like 工具：通过 CadQuery 表达更复杂形状。
- 高级结构：hinges、wheels、panels、supports、grilles、swept profiles 等。

这种混合接口很重要：低级 primitives 保留表达力，高级 abstractions 降低 token 成本和出错率。对于 LLM 来说，直接写原始 URDF 或复杂 CAD 脚本很容易崩；用面向物体结构的 SDK，会把“写对”的概率提高很多。

### 关节定义

每个 articulated joint 显式记录：

- parent part。
- child part。
- joint type。
- origin。
- axis。
- motion limits。

支持的关节类型包括 revolute、prismatic、continuous、fixed。这样生成结果不只是“看起来像物体”，而是有明确可执行的运动结构。例如抽屉需要滑轨几何和 prismatic joint 对齐；柜门需要 hinge leaf、轴线和运动范围一致。

### 自验证

默认 harness 会检查：

- 代码运行错误。
- disconnected parts。
- unintended overlaps。
- 基础结构问题。

但很多对象还需要类别特定约束，例如：

- 抽屉拉出时仍要在滑轨中。
- 铰链盖打开时不能穿过底座。
- 旋钮的 stem 要保持在 socket 里。

因此 Articraft 允许 LLM 在 `run_tests()` 中写测试，用 `TestContext` 检查 contact、gap、overlap、containment、pose-dependent relations，也可以显式声明某些 overlap 是设计上允许的。

## Articraft Harness

harness 的设计是这篇文章最值得注意的地方。它不让 LLM 像普通 coding agent 那样进入一个复杂 repo，而是给一个受限工作区：

- 一个可写文件：`model.py`。
- 只读 SDK 文档。
- curated examples。
- 少量工具。

工具包括：

| 工具 | 作用 |
|---|---|
| `read_file` | 读取 `model.py` 或 SDK 文档 |
| `apply_patch` / `replace` / `write_file` | 修改绑定程序 |
| `find_examples` | 检索相关几何、机构、测试模式 |
| `compile_model` | 执行程序、导出 URDF、运行 QC 和测试 |
| `probe_model` | 只读地测量当前 object_model 的几何关系 |

反馈不是原始日志，而是 failure、warning、note 这类结构化信号。LLM 根据这些信号继续修复，直到没有新的编辑动作且验证通过。

这里的思路和 SWE-agent / Code-as-Policies 有相通之处：关键不是给模型无限自由，而是设计一个适合任务的 agent-computer interface。Articraft 把 3D 资产生成变成“编辑-执行-修复”的闭环。

## 图像条件生成

Articraft 也可以接收 reference image。此时图像作为几何比例、材料、部件和关节的主要依据，优先级高于通用类别先验。

流程大致是：

1. 从图像出发生成 articulated URDF。
2. 初步设置颜色和材料。
3. 借助 LiteReality 的 material painting / PBR refinement 方法改进材料。
4. 生成接近参考图像的可动 3D 资产。

论文还把这一能力接到 LiteReality 的室内场景重建流程中：从 iPhone RoomPlan / RGB-D scan 中取每个物体的 crop、位置、朝向和尺度，用 Articraft 生成对应 articulated object，再组装回完整 room-level scene。

## Articraft-10K 数据集

作者用 Articraft 构建了 Articraft-10K：

- 超过 10K 个 articulated assets。
- 245 个 object categories。
- 映射到 15 个 super-categories。
- 每个对象包含 URDF、`model.py`、agent trace、语义部件和关节信息。

和已有数据集相比：

| 数据集 | 类别数 | 资产数 | 来源 |
|---|---:|---:|---|
| PartNet-Mobility | 46 | 2.3K | PartNet |
| AKB-48 | 48 | 2.0K | real scanning |
| GAPartNet | 27 | 1.2K | PartNet-Mobility / AKB-48 |
| RoboCasa365 articulated portion | 12 | 0.5K | human artist design |
| Articraft-10K | 245 | >10K | agentic generation |

数据构建过程不是纯自动放飞，而是先人工探索 agent 能生成好的类别，再让 LLM 扩展类别和 prompts，最后人工评分过滤。

评分标准包括：

- 几何整体和组件是否真实。
- 是否有预期的 articulated motions。
- 关节是否满足基本物理约束。

低于 4 分的对象被排除。统计上：

- GPT-5.4 生成 6,601 个，保留 5,903 个，保留率 89.4%。
- GPT-5.5 生成 4,010 个，保留 3,828 个，保留率 95.5%。
- Gemini 3.1 Pro 生成 298 个，保留 287 个，保留率 96.3%。
- 总计 10,018 / 10,909 通过，整体保留率 91.8%。

成本上，论文报告约 10,880 次带日志的生成尝试总 API 成本约 12.39K 美元，平均每次约 1.14 美元；保留对象平均约 1.13 美元。由于本地 harness 不需要 GPU，主要瓶颈是 LLM API 延迟和 agent turns。

## 实验结果

### 1. 生成质量优于已有方法

作者比较了：

- Articulate-Anything。
- PhysX-Anything。
- URDF-Anything+。
- 直接让 Codex / GPT-5.5 生成 URDF。
- Articraft with GPT-5.4。
- Articraft with GPT-5.5。

用户研究使用 PartNet-Mobility 的 46 类，每类 5 个 prompts，总共 125 名学生参与，约 5000 个比较。参与者从 6 个方法结果中选出并排序前三名。

结果中 Articraft 明显占优：

- Articraft (GPT-5.5) Top-1 约 42%。
- Articraft (GPT-5.4) Top-1 约 26%。
- Articulate-Anything 和 PhysX-Anything 约 10% 左右。
- 直接 Codex 生成 URDF 约 15%，明显低于 Articraft。

这里最关键的对照是“同样底座模型 + 有没有 Articraft SDK/harness”。直接让 coding agent 写 URDF 排名靠后，而 Articraft 排名前列，说明提升主要来自任务接口和闭环验证，而不只是模型本身更强。

### 2. 不同 LLM 与 reasoning effort

论文用一个 folding quadcopter drone prompt 做定性消融：

- GPT-5.5 high effort 生成更复杂、更完整的几何细节。
- Gemini 3.1 Pro 和 Claude Opus 4.7 也能恢复主关节结构，但几何更简单。
- GPT-5.5 从 low 到 high effort，visual elements 从 39 增加到 78，说明更多推理预算主要提升几何和表面细节，而不只是能不能生成主结构。

这部分不是严格模型排行榜，更像说明：Articraft 是一个可插拔 LLM backend 的系统，模型越强、推理预算越高，资产细节越好。

### 3. 提升 feed-forward articulation model

作者用 Articraft-10K 增强 Particulate 的训练数据，并在 Lightwheel benchmark 上评估。

| 模型 | Rest gIoU ↑ | Rest PC ↓ | Rest mIoU ↑ | Art. gIoU ↑ | Art. PC ↓ | Art. OC ↓ |
|---|---:|---:|---:|---:|---:|---:|
| Particulate | 0.332 | 0.168 | 0.576 | 0.305 | 0.208 | 0.009 |
| Particulate-Articraft | 0.394 | 0.144 | 0.607 | 0.361 | 0.179 | 0.008 |

结论是，Articraft-10K 不只是能看，还能作为训练数据改善下游 3D articulation estimation，尤其对原训练集中没有或少见的类别帮助更明显。

### 4. 可用于机器人仿真和 VR

Articraft 输出的是 URDF 和可动关节结构，因此可以直接导入 NVIDIA Isaac Sim。论文展示了用 Franka arm 通过 IK 控制末端执行器拉开抽屉等交互。

VR 场景中，用户手部和物体发生碰撞后，可以通过脚本控制 URDF joint motion，形成自然的交互。这说明资产具备“可交互性”，不只是静态 3D 展示。

## 论文贡献

1. 提出 Articraft，一个面向 articulated 3D asset generation 的 agentic coding system。  
   它用 SDK + harness 把生成任务变成受控的程序编辑和验证过程。

2. 设计了 LLM-friendly 的 articulated object SDK。  
   它同时覆盖几何、语义部件、关节、运动范围、测试和 URDF 导出。

3. 构建 Articraft-10K。  
   覆盖 245 类、超过 10K 个高质量 articulated assets，并附带源程序和 agent traces。

4. 证明合成 articulated assets 能改善下游模型。  
   用 Articraft-10K 增强 Particulate 训练后，多个 articulation estimation 指标提升。

5. 展示在机器人仿真、VR 和图像条件场景重建中的应用。  
   这把 3D 资产生成和具身交互数据生成连接起来。

## 局限

1. 验证仍偏结构性，难以完全判断全局视觉质量。  
   例如一个 screwcap bottle 可能结构上通过测试，但外形仍明显畸形。

2. 当前 SDK 对某些复杂机构表达不够自然。  
   比如 trigger spray bottle 这类小型复杂机构，需要更丰富的 mechanism-specific abstractions。

3. 没有穷举所有 articulated poses。  
   为了保持低成本，系统鼓励写少量 targeted tests，而不是完整 pose sampling，因此某些运动过程中的碰撞可能漏检。

4. 物理属性仍需要进一步验证。  
   论文中质量、阻尼等可由 LLM 自动赋值，但真实机器人或安全关键仿真中仍需目标域检查。

5. 数据质量依赖人工过滤。  
   Articraft-10K 的高保留质量来自人工 rating 流程，完全自动化数据闭环还没解决。

## 和具身智能研究的关系

Articraft 解决的是具身智能里一个很基础但经常被低估的问题：交互世界从哪里来？

机器人世界模型、VLA、仿真训练和强化学习都需要大量可交互对象。如果只有静态 mesh，机器人无法学习打开、拉出、旋转、折叠、按压这些真实世界动作。Articraft 的价值在于，它用 LLM 生成可关节化、可仿真、可交互的物体数据，把“世界资产生成”推进到可以服务机器人仿真的层面。

它和 CaP-X / MAESTRO 的关系也很有意思：

- Articraft 用代码代理生成机器人可用的环境资产。
- CaP-X 用代码代理控制机器人完成操作任务。
- MAESTRO 用 VLM 代码代理调度机器人模块。

三者共同说明：代码代理正在从软件工程迁移到具身智能，只是作用点不同。Articraft 写的是对象生成程序；CaP-X / MAESTRO 写的是机器人策略程序。

## 我的理解

这篇文章最值得记的点是：可扩展的 3D 资产生成不一定要直接训练一个巨大 3D diffusion / transformer，也可以把问题重写成程序合成。对于 articulated object，程序表达甚至更自然，因为部件、层级、关节和测试本来就是结构化的。

我觉得 Articraft 的真正启发在于 interface design。LLM 不是单独变强就能做好 3D 资产；它需要一个让错误可见、让修复可局部化、让输出可验证的环境。Articraft 把“生成好物体”拆成很多可编译、可探测、可测试的小步骤，所以能扩展到 10K 规模。

## 可继续追的问题

- 能否用 Articraft traces 训练一个开源小模型，降低对闭源大模型 API 的依赖？
- 能否加入更强的自动视觉审查，减少人工 rating？
- 对机器人学习来说，Articraft 生成物体的物理属性是否足够可靠？
- 能否把 Articraft 和任务生成结合，自动生成“对象 + 环境 + 可操作任务”？
- 对复杂机构，是否需要学习或发现新的 SDK primitives？

## 记忆钩子

Articraft 可以记成三句话：

1. 让 LLM 不直接生成 3D，而是写一个生成 articulated object 的程序。
2. 用 SDK 和 harness 把代码生成变成可编译、可测试、可修复的闭环。
3. 最终产出 Articraft-10K，为机器人仿真、VR 和 articulation model 训练提供可交互资产。
