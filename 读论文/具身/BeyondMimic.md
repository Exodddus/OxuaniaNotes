# BeyondMimic: From Motion Tracking to Versatile Humanoid Control via Guided Diffusion

## 基本信息

- 论文: BeyondMimic: From Motion Tracking to Versatile Humanoid Control via Guided Diffusion
- 作者: Qiayuan Liao, Takara E. Truong, Xiaoyu Huang, Yuman Gao, Guy Tevet, Koushil Sreenath, C. Karen Liu
- 机构: UC Berkeley, Stanford University
- 时间: arXiv v4, 2025-11-13
- 项目页: https://beyondmimic.github.io
- 关键词: humanoid control, motion tracking, guided diffusion, classifier guidance, human motion imitation, zero-shot task composition

## 一句话总结

BeyondMimic 的核心是先用一个非常克制的 RL motion tracking 配方，让人形机器人学会大量自然、敏捷的人类动作；再训练一个 state-action latent diffusion model，把这些动作当作可组合的运动先验，在推理时用 classifier guidance 直接朝新任务目标优化，从而实现 joystick 控制、motion inpainting、避障等未见任务，并零样本转移到真实硬件。

## 核心问题

人形机器人控制里有两个长期矛盾：

1. 如果只做 motion tracking，可以复现人类动作，但通常是 motion-specific 的，换一个动作就需要重新调 reward、domain randomization 或控制参数。
2. 如果做通用 goal-conditioned policy，任务适配更强，但动作自然性、敏捷性和真实硬件可转移性往往下降。

BeyondMimic 想回答的是：能否从无标签人类动作中学到一个可扩展、自然、敏捷的技能库，并且在部署时不重新训练，就把这些技能组合成新任务策略？

## 方法总览

论文整体分成两阶段：

| 阶段 | 目标 | 核心做法 |
|---|---|---|
| Stage I: scalable motion tracking | 学会大量人类动作，并稳定转移到真实人形机器人 | 简洁的 RL tracking formulation + 适度 domain randomization + 精细系统实现 |
| Stage II: guided diffusion control | 在推理时组合动作、适配未见任务 | VAE 学运动 latent，latent state-action diffusion 预测短期未来，用 classifier guidance 加任务 cost |

这篇文章有意思的地方在于，它没有把“会模仿”和“会做任务”混在一个训练目标里，而是先把人类动作学扎实，再用扩散模型在推理时把动作先验变成可优化的控制空间。

## 第一阶段：可扩展的人类动作跟踪

BeyondMimic 对 motion tracking 的判断比较反直觉：真实硬件转移不一定需要很重的随机化和大量手工正则，关键是把动力学实现、延迟、执行器建模和 reward 做干净。

### Tracking objective

作者使用 anchor-centered body tracking。也就是说，不强迫所有 body link 在全局坐标里死板追踪 reference，而是用 torso/root 之类的 anchor 表达身体各部位的相对位姿。这样既能保留动作风格，又允许外部扰动或 sim-to-real gap 带来一定全局漂移。

### Reward 设计

相比很多 humanoid RL 工作堆很多正则项，BeyondMimic 只保留：

- task-space body tracking reward：位置、朝向、线速度、角速度。
- joint limit penalty：避免伤硬件。
- action rate penalty：减少高频抖动。
- self-contact penalty：抑制自碰撞。

这种 reward 的价值在于“一套配方走到底”：同一 MDP、同一 reward、同一超参数可以训练大量动作，而不是每个动作单独调。

### Observation 与 action

- Observation 不做复杂 temporal stacking，主要包含 reference phase、anchor error、IMU twist、joint state、上一帧 action。
- Action 是归一化的 joint position setpoints，再交给底层 PD 控制器。
- 作者特别强调不要把高阻抗 PD 当万能解。高增益虽然让仿真里更像 kinematic playback，但真实硬件上会放大噪声、降低被动顺应性，对冲击和接触不友好。

### Domain randomization

作者只随机化少量真实不确定的物理量：地面摩擦/恢复系数、默认关节位置、躯干质心位置，并加入随机速度扰动。论文的态度是：domain randomization 要服务于真实不确定性，而不是把策略训练成保守风格。

## 第二阶段：Guided Diffusion 做通用控制

BeyondMimic 的第二阶段把 motion tracking policies 产生的 state-action 轨迹当作人类式运动分布来建模。

模型由两部分组成：

1. VAE：把 motion intent 编码成平滑 latent，并用 decoder 根据当前本体状态输出动作。
2. Latent state-action diffusion：在包含过去、当前、未来的短期轨迹上建模 state + latent action 分布。

关键设计是 joint modeling of states and actions。普通 action-only diffusion 很难用 task cost 引导，因为任务目标通常定义在未来状态空间，比如到某个 waypoint、避开障碍物、插入关键帧。BeyondMimic 让扩散模型同时预测未来状态和动作，于是推理时可以直接对未来状态加 differentiable cost，denoising 过程会把 state 和 action 一起拉向目标。

## Classifier guidance 的控制含义

推理时，不需要为新任务再训练策略。只要写出一个可微 cost `G(tau)`，扩散采样就可以变成：

$\mathrm{score} = \mathrm{data\_prior\_score} - \nabla G(\tau)$

直观理解：

- diffusion prior 保证动作仍像训练分布里的人类式运动。
- task cost 负责把未来轨迹推向目标。
- receding-horizon 执行时只取当前动作，再根据最新观测重新生成。

这让它可以用简单 cost 做 joystick tracking、waypoint navigation、motion inpainting 和 obstacle avoidance。

## 实验结果

### 1. 大量动作可在同一配方下训练并转移

论文在约 2.5 小时多样人类动作上训练，并在高保真仿真中验证所有动作。真实机器人上部署了 30 个代表性 clips，总计约 15 分钟。

动作覆盖：

- 静态和平衡动作：单腿站立、不同姿态站起。
- 高动态动作：单腿跳、转身踢、180/360 度旋转跳、cartwheel。
- 风格化动作：老人式走路、舞蹈、运动动作。

作者强调很多困难片段和其他动作联合训练，reference motion 超过 3 分钟，但策略仍能保留敏捷性和风格细节。

### 2. 人类偏好评测显示动作更自然

与 Unitree 原生控制器相比，参与者在 20 对 5 秒 walking/running clips 中选择更自然的一方：

| 比较 | BeyondMimic 偏好率 | Unitree 偏好率 |
|---|---:|---:|
| overall | 70.8% | 29.2% |
| walking | 57.0% | 43.0% |
| running | 84.7% | 15.3% |

running 的差距尤其大，说明它不仅能“走稳”，还确实学到了更接近人类运动的动态协调。

### 3. 未见任务的 zero-shot 组合

BeyondMimic 展示了几类推理时组合能力：

- velocity/waypoint command：仿真中 walking 和 running 平均速度跟踪误差分别约 12.14% 和 13.65%。
- joystick locomotion：可以做全向行走、受踢扰动后继续追踪命令，并在跑道上连续跑 50m 以上。
- motion inpainting：从 joystick walking 切换到 cartwheel keyframes，再平滑回到 walking。
- task composition：把 waypoint cost 和 SDF obstacle avoidance cost 相加，实现简单场景避障。

这里最重要的不是某个指标，而是同一个模型可以在任务形式之间切换：速度跟踪、关键帧补全、避障导航，都通过推理时 guidance 完成。

## 论文贡献

1. 提出一个可扩展的 humanoid motion tracking recipe。  
   单套 reward 和超参数支持大量敏捷、自然的人类动作，并能零样本转移到真实硬件。

2. 把 motion imitation 推进到 task versatility。  
   不是只复现已有动作，而是用 latent state-action diffusion 在推理时组合动作，解决训练中没出现过的任务。

3. 展示了 diffusion guidance 在机器人控制中的实际价值。  
   它不是生成好看的轨迹，而是在 state-action 轨迹空间里做在线优化。

4. 给 humanoid foundation model 提供了一个中间路线。  
   不直接从语言到动作，也不靠硬编码 planner，而是先学运动分布，再用 cost 引导这个分布。

## 局限

1. 预测 horizon 短。  
   当前模型预测约 0.64 秒，足够 reactive control 和局部避障，但不适合远距离目标和长程任务规划。

2. 依赖状态估计质量。  
   proprioception 或 state estimation 错误会传到生成轨迹里，未来需要更好的 sensor fusion 或 learned estimator。

3. guidance 对 history 有副作用。  
   history 有助于稳定预测，但也可能让模型陷入重复 gait orbit。为了跳出模式需要加大 guidance weight，而这又可能让 mode switching 不稳定。

4. 细粒度控制仍弱。  
   guidance 对粗粒度目标有效，但对高精度动作目标还需要调权重，未来可能要结合 adapter、fine-tuning 或更结构化的控制层。

## 和具身智能研究的关系

BeyondMimic 说明，人形机器人的“基础能力”未必首先来自语言或任务数据，而可能来自大规模人类运动数据。它把人类动作看作一个可复用的 motor prior，然后用推理时优化把这个 prior 变成任务执行能力。

这和 DreamZero / DreamDojo 这类视觉世界模型路线有一个互补关系：

- DreamZero 强调视频未来和动作的联合建模。
- BeyondMimic 强调身体运动轨迹和动作的联合建模。
- 两者都在做一件事：先学一个强行为/世界先验，再用目标条件把先验拉到当前任务。

## 我的理解

这篇文章的关键不是“扩散模型也能做人形机器人控制”，而是它给出了一种很清楚的分工：RL 负责把真实硬件可执行的人类动作打磨出来，diffusion 负责把动作空间变成可组合、可引导、可推理时优化的生成模型。

我觉得它最值得记的是：task generalization 不一定要全部塞进训练数据。只要生成模型学到的运动先验足够好，一些新任务可以通过推理时 cost composition 解决。这对 humanoid 特别有价值，因为人形动作的自然性和稳定性太难手写，最好先作为 prior 学出来。

## 可继续追的问题

- 0.64 秒预测 horizon 如何扩展到分钟级长程任务？
- guidance weight 能否自动调节，而不是人工设置？
- state-action diffusion 能否接入视觉和物体状态，支持真正的 loco-manipulation？
- 能否把 BeyondMimic 的 motor prior 和 VLA / WAM 的语义规划结合起来？
- 对高精度接触任务，是否需要加入力觉、足底接触或全身 contact prediction？

## 记忆钩子

BeyondMimic 可以记成三句话：

1. 先用简洁 RL 配方学会人类式动作。
2. 再用 latent state-action diffusion 把动作变成可组合先验。
3. 推理时写一个 cost，就能让机器人用已有动作去解新任务。

