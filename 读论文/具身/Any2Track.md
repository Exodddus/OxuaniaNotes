# Track Any Motions under Any Disturbances

## 基本信息

- 论文: Track Any Motions under Any Disturbances
- 方法名: Any2Track / AnyTracker / AnyAdapter
- 作者: Zhikai Zhang, Jun Guo, Chao Chen, Jilong Wang, Chenghuai Lin, Yunrui Lian, Han Xue, Zhenrong Wang, Maoqi Liu, Jiangran Lyu, Huaping Liu, He Wang, Li Yi
- 机构: Tsinghua University, Peking University, Galbot, Shanghai Qi Zhi Institute
- 时间: arXiv v3, 2025-09-30
- 项目页: https://zzk273.github.io/Any2Track/
- 关键词: humanoid motion tracking, online dynamics adaptation, sim-to-real, disturbance robustness, Unitree G1, AMASS, LAFAN1

## 一句话总结

Any2Track 的目标是做一个“任意动作 + 任意扰动”的人形机器人基础 motion tracker：先用 AnyTracker 通过 canonicalized action space 和 specialist-to-generalist distillation 学会多样、高动态、接触丰富的人类动作，再用 AnyAdapter 在不破坏基础动作表达力的前提下，从历史交互中估计环境动力学并在线适应地形、外力和物理参数变化。

## 核心问题

论文认为，一个 foundational humanoid motion tracker 需要同时满足两件事：

1. Track Any Motions：能跟踪多样、高动态、接触丰富的人类动作，而不是只会短 clip 或准静态动作。
2. Adapt to Any Disturbances：真实环境里面对地形、外力、载荷、摩擦、质心变化等扰动仍然稳定。

现有方法通常只能做到其中一边：

- 有些方法动作表达力强，但不够鲁棒。
- 有些方法通过 domain randomization 变鲁棒，但动作会变保守，失去人类式动态质量。

Any2Track 的核心思路是把“基础动作执行能力”和“动力学适应能力”解耦，而不是让一个网络同时学习所有东西。

## 方法总览

Any2Track 是两阶段 RL 框架：

| 模块 | 作用 | 关键设计 |
|---|---|---|
| AnyTracker | 学一个通用基础 motion tracker | canonicalized action space, motion clustering, specialist-to-generalist distillation |
| AnyAdapter | 在扰动下做在线动力学适应 | history encoder, dynamics-aware world model prediction, adapter fine-tuning |

两阶段的关系很重要：AnyTracker 训练时不加 dynamics randomization，专注学动作质量；AnyAdapter 再引入 dynamics variance，把适应能力作为额外模块加到基础 tracker 上。

## AnyTracker：Track Any Motions

### 训练数据

作者使用 AMASS 和 LAFAN1 的组合数据，并保留高动态和接触丰富的动作。和 GMT 不同，Any2Track 没有为了训练容易而删掉大量高动态/接触动作。

### 输入输出

策略输入包括：

- 当前机器人本体状态：角速度、投影重力、各关节位置/速度、上一帧 action。
- tracking goal：下一帧目标关节位置/速度，以及局部坐标下的 rigid body 信息。

输出是 per-joint action，再送入 PD 控制器。

### Canonicalized Action Spaces

高自由度人形机器人的各关节 action 分布差异很大，如果让策略直接在一个统一 action range 里学习，会增加优化难度。

AnyTracker 的做法是：

- 用 `tanh` 把 policy 输出压到 `[-1, 1]`。
- 为不同关节设置经验 action scale `alpha`。
- 不直接预测绝对 PD target，而是预测相对 reference motion 的 residual PD offset。

公式直观上是：

$q_d = q_{\mathrm{ref,next}} + \alpha \tanh(\pi(s, g))$

这个设计把复杂、多模态的关节动作分布规范化，让策略只需要学习更紧凑的 residual 控制。

### Specialist-to-Generalist

动作类别差异也会导致一个策略难以直接学所有动作。

作者先按动作类别训练 specialist：

- LAFAN1 直接使用已有动作类别。
- AMASS 使用 HumanML3D 的 VERB 标签，经 CLIP 文本 embedding 后 K-means 聚成 6 类。

然后用 DAgger 把 specialists 蒸馏成一个 generalist policy。这样既降低训练难度，又保留部署时单策略的便利。

## AnyAdapter：Adapt to Any Disturbances

AnyAdapter 的问题设定是：不要重新训练或破坏已经学好的 motion tracker，而是在它上面加一个轻量适配能力。

### 扰动类型

训练时加入三类 dynamics variance：

- Terrains：地面摩擦、Perlin noise 地形高度等。
- External Forces：随机施加外力。
- Physical Property Changes：DoF friction、armature、torso CoM、torso mass、默认关节位置 jitter 等。

### Dynamics-aware world model prediction

AnyAdapter 使用历史窗口 `H=79` 的 state-action 交互，编码成 dynamics embedding `e_t`。然后 world model 用这个 embedding 自回归预测未来 `N=20` 步状态。

训练目标是：

$L_{\mathrm{wm}} = \sum_i \lVert s_{t+i} - \hat{s}_{t+i} \rVert_1$

这相当于用“预测动力学是否准确”作为 proxy task，逼迫 history encoder 提取真正和环境动力学有关的信息，而不是把历史信息全部粗暴拼给策略。

### Adapter fine-tuning

作者冻结 AnyTracker，在每一层加 zero-initialized adapter。初始时 adapter 不影响基础 tracker；训练推进后，adapter 逐层注入动力学适应能力。

这个设计很像 LoRA/adapter 思想在人形控制里的应用：基础网络保留运动表达力，适配模块负责应对地形、外力和载荷变化。

## 实验设置

平台：29-DoF Unitree G1。  
仿真：MuJoCo，PPO，8 GPU 并行训练。  
指标：

- SR：tracking 成功率。
- MPJPE：平均每关节位置误差，单位 mm。
- MPJVE：平均每关节速度误差，单位 mm/frame。

## 主要结果

### 1. AnyTracker 提升通用动作跟踪质量

在 curated AMASS + LAFAN1 测试上：

| 方法 | SR ↑ | MPJPE ↓ | MPJVE ↓ |
|---|---:|---:|---:|
| OmniH2O | 75.64 | 36.12 | 12.24 |
| Exbody2 | 79.68 | 34.70 | 13.69 |
| Ours w/o CAS | 84.92 | 30.15 | 9.87 |
| Ours w/o distillation | 83.32 | 31.29 | 9.61 |
| AnyTracker | 89.23 | 27.96 | 6.43 |

结论很清楚：canonicalized action space 和 specialist-to-generalist distillation 都有效，组合后最好。

### 2. AnyAdapter 在扰动下优于基线

在 LAFAN1 上加入不同扰动后，Any2Track 与 PPO、RMA、DWL、Dual History 等方法比较。

关键结果：

| 场景     | Any2Track SR | Any2Track MPJPE | 说明                        |
| ------ | -----------: | --------------: | ------------------------- |
| 无扰动    |         89.8 |           16.46 | 成功率最高，误差最低                |
| 地形扰动   |         83.2 |           20.68 | 比 RMA/DWL/Dual History 更稳 |
| 外力扰动   |         59.0 |           28.97 | 外力是最难场景，但仍最好              |
| 物理参数变化 |         80.6 |           27.75 | 载荷/质心/摩擦变化下保持优势           |
|        |              |                 |                           |

消融也说明：

- 去掉 world model 后，在外力和物理参数变化下性能明显掉，甚至比 vanilla PPO 差。
- 去掉 adapter、直接混合学习基础动作和适应能力，也会损失性能。

这支持作者的核心观点：动力学适应要靠好的 dynamics representation，也要和基础动作能力解耦。

### 3. 真实 Unitree G1 部署

真实测试包括：

- complex terrain：木板、纸板、泡沫、布料。
- external constraint：背部通过定长绳连接到吊具，形成外部约束/拉力。
- weight carrying：背部加 5kg payload。

与带 domain randomization 的 PPO 比：

| 场景 | PPO MPJPE | Any2Track MPJPE | 改善 |
|---|---:|---:|---:|
| 无扰动 | 29.38 | 17.38 | -12.00 |
| Complex Terrain | 37.21 | 18.34 | -18.87 |
| External Constraint | 39.84 | 19.17 | -20.67 |
| Weight Carrying | 37.52 | 23.24 | -14.28 |

Any2Track 在真实扰动越强时优势越明显，这正好验证了 adapter 的价值。

## 论文贡献

1. 提出 Any2Track，把通用 motion tracking 和在线动力学适应拆成两阶段。  
   这比直接用一个策略加大 domain randomization 更清晰，也更能保持动作表达力。

2. 提出 AnyTracker 的两个实用设计。  
   canonicalized action space 解决高自由度关节 action 分布问题；specialist-to-generalist 解决动作类别多样性问题。

3. 提出 AnyAdapter，用 world model proxy task 学 dynamics embedding。  
   它不是简单把历史拼给策略，而是让历史编码必须能解释未来动力学。

4. 在真实 Unitree G1 上展示多扰动鲁棒跟踪。  
   包括复杂地形、外部约束和 5kg 载荷。

## 和相关工作的差异

与 Exbody2、ASAP、GMT 等 motion tracking 工作相比，Any2Track 的区别在于：

- 既保留动作多样性，又处理高动态和接触丰富动作。
- 不只在干净仿真里跟踪，还专门处理真实世界扰动。
- 它不是靠一个大而全的 domain-randomized policy，而是把适应能力模块化。

与 RMA、DWL 等在线适应工作相比，Any2Track 更关注 humanoid motion tracking 的表达力。腿式机器人适应方法如果直接套过来，可能会让动作变得保守，不适合需要人类式自然性和高动态的 tracking。

## 局限与疑问

1. 任务仍是 motion tracking，不是语义任务执行。  
   它能让机器人稳定复现动作，但如何接到语言指令、视觉目标或 manipulation planner 还需要下游系统。

2. 外力扰动下成功率仍明显下降。  
   外力场景 SR 为 59.0，说明强外部扰动仍是主要瓶颈。

3. 依赖动作数据和 retargeting。  
   AMASS/LAFAN1 的动作到 Unitree G1 的可跟踪性和 retargeting 质量会影响最终能力。

4. Adapter 的泛化边界还不清楚。  
   当前覆盖地形、外力、物理参数变化，但对更复杂接触、碰撞、手持工具或移动操作是否足够，还需要验证。

## 对具身智能研究的启发

1. “基础动作能力”和“环境适应能力”可以解耦。  
   先学高质量动作，再加适配模块，可能比从一开始把所有随机化混进去更适合 humanoid。

2. history encoder 需要有明确 proxy task。  
   让 embedding 能预测未来动力学，比直接用历史观测更可靠。

3. Adapter 思想很适合机器人控制。  
   基础 policy 作为 motion prior，adapter 负责地形/载荷/动力学变化，既稳又可扩展。

4. Motion tracker 可以成为上层 VLA/WAM 的执行底座。  
   如果上层模型输出目标姿态、关键帧或动作片段，Any2Track 这类 tracker 可以负责真实硬件上的鲁棒执行。

## 我的理解

Any2Track 最有价值的地方，是它没有把鲁棒性和动作质量当成同一个 RL 问题硬解。人形机器人如果为了抗扰动而过度随机化，动作很容易变钝、变保守；如果只追求好看，又很难落地。Any2Track 的两阶段结构给了一个工程上很可用的折中：基础 tracker 负责“像人一样动”，adapter 负责“在真实世界还能动”。

我会把它看作 humanoid foundation stack 里的底层模块。它本身不是完整具身智能模型，但如果和上层语言/视觉/世界模型结合，能成为非常重要的 action execution layer。

## 可复用笔记

- 核心框架：Any2Track = AnyTracker + AnyAdapter。
- 核心动作设计：`qd = q_ref_next + alpha * tanh(policy(s, g))`。
- 核心训练策略：先无扰动学高质量 tracking，再冻结基础 tracker 加 adapter 学扰动适应。
- 核心 proxy task：dynamics-aware world model prediction from history。
- 核心结论：鲁棒性不是简单 domain randomization 越多越好，适应能力最好模块化注入。

