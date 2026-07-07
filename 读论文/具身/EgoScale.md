# EgoScale: Scaling Dexterous Manipulation with Diverse Egocentric Human Data

## 论文信息

- 标题：**EgoScale: Scaling Dexterous Manipulation with Diverse Egocentric Human Data**
- 作者：Ruijie Zheng, Dantong Niu, Yuqi Xie, Jing Wang, Mengda Xu, Yunfan Jiang, Fernando Castañeda, Fengyuan Hu, You Liang Tan, Letian Fu, Trevor Darrell, Furong Huang, Yuke Zhu, Danfei Xu, Linxi Fan
- 机构：NVIDIA, UC Berkeley, University of Maryland
- 时间：2026
- 主题：第一视角人类视频、灵巧操作、VLA、human-to-robot transfer、scaling law、少样本机器人学习
- 项目页：https://research.nvidia.com/labs/gear/egoscale/

## 一句话总结

EgoScale 证明了大规模第一视角人类操作视频可以作为灵巧机器人操作的可扩展监督信号：先用 20,854 小时人类视频预训练 VLA，再用少量人类-机器人对齐数据做 mid-training，能够显著提升 22-DoF 灵巧手的真实机器人任务表现，并出现 one-shot 任务适应和跨机器人形态迁移能力。

## 核心问题

论文想回答的是：人类数据能否真正成为灵巧机器人操作的主要训练来源，而不只是视觉表征预训练或低自由度夹爪控制的辅助数据？

过去 human-to-robot transfer 的主要限制有两点：

1. 人类数据规模通常只有几十到几百小时，无法观察 scaling 行为。
2. 多数工作面向夹爪或低 DoF 手，无法验证细粒度手指关节运动对高自由度灵巧操作是否有用。

EgoScale 的核心判断是：灵巧操作的人机迁移不是单纯的对齐技巧问题，而是一个 scaling phenomenon。人类第一视角数据越大，动作预测损失越低，真实机器人操作性能越好。

## 方法总览

EgoScale 使用三阶段训练流程：

| 阶段 | 数据 | 目的 |
|---|---:|---|
| Stage I: Human Pretraining | 20,854 小时第一视角人类视频 | 学习通用操作结构、手腕运动和手指关节先验 |
| Stage II: Aligned Mid-training | 约 50 小时人类数据 + 4 小时机器人数据 | 把人类运动表征锚定到机器人传感和控制空间 |
| Stage III: Post-training | 下游任务机器人示范 | 面向具体灵巧任务微调执行策略 |

整体思路可以概括为：用大规模人类数据解决多样性和操作先验，用小规模对齐数据解决 embodiment gap，再用少量机器人数据完成任务适配。

## 人类动作表示

论文没有直接拿视频做普通视觉预训练，而是显式监督人类动作：

1. 手腕级手臂运动  
   使用相邻时间步之间的相对 wrist motion，也就是相对末端执行器位姿变化。这样可以去掉绝对相机运动和全局坐标的影响，更接近机器人控制中的相对 end-effector command。

2. 手指关节运动  
   将 21 个人手关键点通过优化式 retargeting 映射到 Sharpa 22-DoF 灵巧手关节空间。目标不是只看 fingertips，而是保留完整手指关节协同、捏合、握拳等细节。

3. 跨形态适配  
   手腕动作在不同机器人间共享；手部动作通过 embodiment-conditioned MLP adapters 映射到具体机器人手的动作空间。

这点是论文的关键：它让人类视频不仅提供视觉语义，还直接提供“可执行动作结构”的监督。

## 数据构成

### Stage I: 大规模人类预训练数据

- 总量：20,854 小时第一视角视频。
- 场景：9,869 个。
- 任务：6,015 个。
- 对象：43,237 个。
- 来源：大量 in-the-wild egocentric recordings，覆盖家庭、工业、零售、教育等真实环境。
- 额外加入：829 小时 EgoDex 数据，由 Apple Vision Pro 采集，手腕和手部 tracking 更精确，覆盖 194 个 tabletop manipulation tasks。

论文强调，这些大规模数据本身是 noisy、unconstrained、not task-aligned 的，但规模和多样性可以弥补噪声。

### Stage II: 人类-机器人对齐 mid-training 数据

- 任务：344 个 tabletop manipulation tasks。
- 每个任务约 30 条人类轨迹和 5 条机器人轨迹。
- 总量：约 50 小时人类数据 + 4 小时机器人数据。
- 采集设置：人类演示使用与机器人相同或相近的相机配置，包括头戴相机和腕部相机；手腕用 Vive tracker，手部用 Manus gloves。

这批数据规模很小，但价值在于精确对齐：它把大规模人类预训练得到的抽象操作先验连接到机器人可执行的视觉、状态和动作接口。

## 模型结构

模型是 flow-based Vision-Language-Action policy，结构类似 GR00T N1：

- 输入：图像、语言指令、机器人 proprioceptive state。
- 人类数据没有机器人 proprioception，因此用 learnable placeholder token 替代。
- Backbone：VLM/视觉语言编码器。
- Action module：DiT action expert，用 flow matching 预测未来动作 chunk。
- 共享部分：视觉语言骨干、相对 wrist motion prediction、DiT action expert。
- 形态特定部分：轻量 MLP adapters，用于不同机器人 proprioception 和手部动作空间的输入输出映射。

训练细节：

- Stage I：20K 小时人类数据，100K steps，256 张 GB200 GPU，global batch size 8,192，学习率约 5e-5，全参数解冻。
- Stage II：对齐人类-机器人 play data，50K steps，batch size 2,048，学习率约 3e-5，冻结视觉语言 backbone，只更新 vision encoder 和 DiT action expert。
- Stage III：下游任务机器人示范，10K steps，batch size 512，学习率约 3e-5。

## 机器人平台与任务

主实验平台是 Galaxea R1 Pro：

- 双 7-DoF 机械臂。
- 22-DoF Sharpa Wave 灵巧手。
- 三个 RGB 相机：头部第一视角相机 + 两个腕部相机。
- 底盘和躯干固定，重点测试双臂桌面灵巧操作。

下游评测任务共 5 个：

1. Shirt Rolling：双手折叠并卷起 T-shirt，放入篮子。
2. Card Sorting：从一叠卡片中摩擦分离单张卡片，并按颜色插入对应 holder。
3. Tong Fruit Transfer：拿起夹子，用夹子夹取水果并放到目标位置。
4. Bottle Cap Unscrewing：抓住并持续旋转瓶盖，将其拧下。
5. Syringe Liquid Transfer：拿起注射器，从管 A 抽液体，注入管 B，再丢弃注射器。

除 Shirt Rolling 使用 20 条示范外，其余每个任务使用 100 条 teleoperated robot demonstrations。指标包括 binary success rate 和细粒度 task completion score。

## 主要实验结论

### 1. 大规模人类预训练显著提升灵巧操作

主实验比较了四种模型：

- No Pretrain：从零训练。
- Midtrain Only：只用对齐人类-机器人数据。
- Human Pretrain：只用大规模人类预训练。
- Human Pretrain + Midtrain：大规模人类预训练后再做对齐 mid-training。

平均 task completion score：

| 方法 | 平均 completion score |
|---|---:|
| No Pretrain | 0.24 |
| Midtrain Only | 0.53 |
| Human Pretrain | 0.71 |
| Human Pretrain + Midtrain | 0.83 |

平均 success rate：

| 方法 | 平均 success rate |
|---|---:|
| No Pretrain | 0.02 |
| Midtrain Only | 0.28 |
| Human Pretrain | 0.38 |
| Human Pretrain + Midtrain | 0.56 |

结论：大规模人类预训练即使 noisy、非任务对齐、非机器人传感对齐，也能超过只用小规模对齐数据的 baseline；而 human pretraining 与 aligned mid-training 结合效果最好。

### 2. 人类数据规模和机器人性能存在明确 scaling 行为

作者分别用 1k、2k、4k、10k、20k 小时人类数据预训练，然后直接 post-train 到下游机器人任务。

平均 task completion score 随数据规模单调上升：

| 人类预训练数据量 | 平均 completion score |
|---:|---:|
| 1k hours | 0.30 |
| 2k hours | 0.45 |
| 4k hours | 0.48 |
| 10k hours | 0.57 |
| 20k hours | 0.71 |

更重要的是，人类动作预测验证损失与数据规模之间呈近似 log-linear scaling law：

```text
L = 0.024 - 0.003 * ln(D)
R^2 = 0.9983
```

其中 D 是人类预训练小时数。论文进一步指出，offline human validation loss 与真实机器人 task completion 强相关，因此这个 loss 不是纯离线指标，而能预测 embodied control capability。

### 3. aligned mid-training 带来 one-shot transfer

作者在两个新任务上测试 one-shot 适应：

- Fold Shirt：1 条机器人示范 + 100 条 aligned human demonstrations。
- Unscrewing Water Bottles：每种瓶子 1 条机器人示范 + 100 条 aligned human demonstrations，共 3 种瓶子。

结果：

| 方法 | Fold Shirt success | Water Bottle success |
|---|---:|---:|
| Midtrain Only | 0.15 | 0.00 |
| Human Pretrain | 0.35 | 0.00 |
| Human Pretrain + Midtrain | 0.88 | 0.556 |

单独人类预训练或单独 mid-training 都不足以 one-shot 成功；两者结合后才出现明显少样本泛化。这说明 mid-training 不只是加数据，而是在学习可复用 motion primitives 与机器人执行接口之间的映射。

### 4. 人类预训练可以跨机器人形态迁移

作者把策略迁移到 Unitree G1：

- G1 机械臂工作空间更短。
- 手是 7-DoF tri-finger hand，而不是 22-DoF Sharpa hand。
- 任务包括 Pen in Bin 和 Dish Handover in Rack。

结果显示 Human Pretrain + Midtrain 最好：

| 任务 | completion score | success rate |
|---|---:|---:|
| Pen in Bin | 0.83 | 0.67 |
| Dish in Rack | 0.88 | 0.50 |

相比 Midtrain Only 或 Human Pretrain Only，加入二者组合后有明显提升。论文的解释是：人类预训练学到的是可迁移的操作结构，而 mid-training 把该结构适配到 G1 的 sensing 和 actuation interface。

### 5. 手部动作表示很关键

作者比较了三种人类预训练动作表示：

- Wrist Only：只预测手腕轨迹。
- Fingertip：预测手腕和指尖 SE(3) 轨迹，再通过 MLP 映射到机器人关节。
- Full Retargeted Joints：将人手关键点 retarget 到 22-DoF 机器人手关节空间。

在 Card、Tong、Bottle 三个任务上的 completion score：

| 表示 | Card | Tong | Bottle |
|---|---:|---:|---:|
| Wrist Only | 0.56 | 0.24 | 0.26 |
| Fingertip | 0.17 | 0.76 | 0.55 |
| Full Retargeted Joints | 0.74 | 0.79 | 0.61 |

结论：只用 wrist motion 不足以支撑精细接触和手指时序；fingertip 表示几何信息更多，但映射到关节后容易产生不合理姿态；retargeted joint-space hand actions 最稳定。

## 论文贡献

1. 提出并验证大规模 egocentric human data 对灵巧操作的 scaling law。  
   20K 小时规模下，人类动作验证损失与数据量呈 log-linear 关系，且该损失能预测真实机器人性能。

2. 提出简单有效的人机迁移 recipe。  
   先大规模 human pretraining，再用少量 aligned human-robot mid-training 进行 embodiment grounding，最后下游 post-training。

3. 展示 one-shot transfer 和跨 embodiment transfer。  
   在 22-DoF 灵巧手上可 one-shot 适应新任务，也能迁移到 7-DoF tri-finger hand 的 Unitree G1。

4. 强调手部动作监督的重要性。  
   对灵巧操作而言，显式 hand articulation supervision 比只学视觉表征或 wrist motion 更关键。

## 和相关工作的差异

与 EgoMimic、EgoVLA、DexWild 等工作相比，EgoScale 的区别在于：

- 更关注大规模纯人类预训练，而不是主要依赖 paired/co-training robot demonstrations。
- 数据规模显著更大，达到 20,854 小时。
- 任务聚焦高 DoF 灵巧手，而不只是夹爪或低自由度末端执行器。
- 系统性展示 scaling behavior，并把 offline validation loss 与 downstream real-robot performance 关联起来。

与机器人数据 scaling 工作相比，EgoScale 的重点不是继续扩大昂贵机器人数据，而是把人类第一视角视频视作一种可扩展 embodiment，用人类行为规模弥补机器人数据采集成本。

## 局限与疑问

1. 对感知 pipeline 的依赖很强。  
   大规模人类数据需要 SLAM、hand pose estimation、retargeting 等步骤。如果这些估计存在系统性误差，可能影响动作监督质量。

2. 训练成本极高。  
   Stage I 使用 256 张 GB200 GPU 训练 100K steps，这说明方法虽然数据可扩展，但算力门槛很高。

3. mid-training 仍然需要精心对齐采集。  
   50 小时人类 + 4 小时机器人数据不算大，但需要相机视角、运动捕捉、机器人 teleoperation 等系统级配置。

4. zero-shot 仍未真正解决。  
   论文展示了 one-shot 和 few-shot 能力，但新任务仍然需要至少 1 条机器人示范和 aligned human demonstrations。

5. 实验任务主要是 tabletop manipulation。  
   尽管 G1 实验带有 humanoid 平台迁移，但核心评估仍集中在桌面操作，开放环境长时程任务还有距离。

6. scaling law 的外推边界未知。  
   论文在 1k 到 20k 小时范围内没有观察到饱和，但没有证明继续扩大数据和模型后仍保持同样趋势。

## 对具身智能研究的启发

1. 人类第一视角视频的价值不只是视觉语义，而是动作监督。  
   如果能提取 wrist motion、hand pose、object interaction，人类视频可以直接变成机器人学习的行动数据。

2. *scale 和 alignment 可以解耦*。  
   大规模数据不必精确对齐，负责提供多样性和操作先验；少量高质量对齐数据负责把表征接到机器人控制接口。

3. 灵巧操作需要 hand-level representation。  
   高自由度手的关键不只是末端轨迹，而是 finger articulation、contact timing、grasp stability。

4. offline metric 如果设计得好，可以预测真实机器人表现。  
   EgoScale 的 human validation loss 能追踪下游 completion score，这对减少真实机器人试错成本很重要。

5. 未来可行方向是弱标注或自监督扩展。  
   论文已经依赖大规模动作标注/估计，如果未来能用未标注 egocentric video 做 self-supervised 或 semi-supervised pretraining，数据规模会更容易继续扩大。

## 我的理解

这篇文章最有价值的地方，不是单个任务成功率多高，而是把“人类视频能不能训练灵巧机器人”从经验判断推进到了 scaling law 的层面。它说明只要动作表征足够贴近机器人控制，第一视角人类数据就不只是 imitation learning 的辅助材料，而可能成为具身智能的基础数据来源。

不过 EgoScale 也没有绕开 embodiment gap。它的 recipe 仍然需要一个小而精的 mid-training 阶段。换句话说，scale 解决“学到什么操作结构”，alignment 解决“如何让机器人执行”。这两个部分缺一不可。

对后续研究来说，我会重点关注三个问题：

1. 能否降低 hand pose extraction 和 retargeting 的成本与误差？
2. 能否把 one-shot 推到 zero-shot，或者只用语言/视频指定新任务？
3. 能否在非桌面、移动操作、长时程组合任务中继续观察到类似 scaling law？

## 可复用笔记

- 核心公式：`L = 0.024 - 0.003 * ln(D)`，其中 D 是人类预训练小时数，`R^2 = 0.9983`。
- 核心 recipe：large-scale human pretraining + lightweight aligned human-robot mid-training + task-specific robot post-training。
- 核心经验：noisy large-scale human data > small aligned data alone；但 best performance 需要二者结合。
- 核心设计选择：retargeted joint-space hand actions 比 wrist-only 和 fingertip representation 更稳。
- 核心结论：human data 可以被视作一种 scalable embodiment，而不是机器人数据的弱替代品。
