# ManipTrans: Efficient Dexterous Bimanual Manipulation Transfer via Residual Learning

## 基本信息

- 论文: ManipTrans: Efficient Dexterous Bimanual Manipulation Transfer via Residual Learning
- 作者: Kailin Li, Puhao Li, Tengyu Liu, Yuyang Li, Siyuan Huang
- 机构: BIGAI, Tsinghua University, Peking University
- 时间: arXiv v1, 2025-03-27；CVPR 2025
- 项目页: https://maniptrans.github.io
- 关键词: dexterous manipulation, bimanual manipulation, residual learning, human-to-robot transfer, MoCap, DexManipNet, Isaac Gym

## 一句话总结

ManipTrans 提出一个高效的双阶段迁移框架：先预训练一个通用手部轨迹模仿器，让机器人手学会人手关节/指尖运动；再在物体交互约束下训练 residual policy，修正接触、物体跟踪和双手协同，从而把人类 MoCap 双手操作数据转成物理可行的灵巧机器人手轨迹，并构建 3.3K episodes 的 DexManipNet 数据集。

## 核心问题

灵巧双手操作需要大量精确、自然、物理可行的数据，但现有数据获取方式都有明显问题：

1. 纯 RL 需要 task-specific reward，探索成本高，很难覆盖复杂双手任务。
2. 真实遥操作成本高，且 often embodiment-specific，缺少触觉反馈时动作容易僵硬。
3. 直接 retarget 人手姿态到机器人手会受到 morphology gap、MoCap 噪声和物理接触误差影响，尤其在 pen capping、bottle unscrewing 这类高精度双手任务中容易失败。

ManipTrans 的目标是：离线利用已有手-物交互 MoCap 数据，快速生成机器人灵巧手可以执行的高质量操作轨迹。

## 方法总览

ManipTrans 把迁移分成两个阶段：

| 阶段 | 目标 | 作用 |
|---|---|---|
| Hand Trajectory Imitating | 只模仿人手运动 | 缓解 morphology gap，学通用 finger motion prior |
| Residual Learning for Interaction | 修正物体交互和接触 | 保证稳定接触、物体轨迹跟踪、双手协同 |

这个分解很关键：第一阶段不碰复杂物体约束，只把“手怎么动”学好；第二阶段在第一阶段输出的粗动作上加 residual correction，只负责物理交互细节。

## 问题建模

论文以双手双物体交互为一般形式。人手轨迹包括：

- wrist 6-DoF pose。
- wrist linear/angular velocity。
- MANO hand keypoints / finger joint positions。
- 对应速度。

物体轨迹包括：

- object 6-DoF pose。
- object linear/angular velocity。

动作空间包括每只 dexterous hand 的：

- 关节 PD target positions。
- 施加在 wrist 上的 6-DoF force。

训练使用 PPO，两个阶段分别有不同 state 和 reward。

## 第一阶段：Hand Trajectory Imitating

第一阶段训练一个 general hand trajectory imitation model `I`，目标是准确模仿人手细粒度运动。

输入包括：

- reference human hand trajectory。
- 当前机器人手 proprioception：关节位置/速度、腕部位姿/速度等。

Reward 主要包含：

1. wrist tracking reward：追踪手腕位姿和速度。
2. finger imitation reward：追踪人手关键点和机器人手关键点的位置。
3. smoothness reward：减少抖动和不自然控制。

训练数据来自多个 hand-only / hand-object 数据集和插值生成数据，并通过镜像增强平衡左右手。

我的理解是，这一阶段相当于学一个 embodiment-level 的“手部运动翻译器”。它不要求物体一定被正确操作，只要求机器人手能自然、稳定地跟随人手运动结构。

## 第二阶段：Residual Learning for Interaction

第二阶段在第一阶段动作 `a_I` 的基础上，加一个 residual action：

$a = a_I + \Delta a_R$

Residual module 的任务是让动作满足物理交互约束。

### State expansion

为了处理接触和物体操作，第二阶段加入：

- object pose / velocity。
- hand-object distance features。
- fingertip contact force / tactile-like information。
- expanded proprioception。

### Reward

除了第一阶段的 hand imitation reward，还加入：

1. object following reward：让仿真物体轨迹跟 reference object trajectory 接近，包括位置、姿态和速度。
2. contact force reward：当 MoCap 中人手指尖接近物体表面时，鼓励机器人手产生合适接触力。

总目标可以理解为：

$r_R = r_I + w_{\mathrm{object}} r_{\mathrm{object}} + w_{\mathrm{contact}} r_{\mathrm{contact}}$

### Training strategy

为了避免复杂接触任务一开始就陷入局部最优，作者使用 curriculum / relaxation：

- 初期放松 gravity。
- 提高 friction，让接触更容易稳定。
- 放松 finger/object tracking thresholds。
- 随训练推进逐步恢复真实物理约束。

这和 QuasiSim 之类 quasi-physical relaxation 思想相似，但 ManipTrans 直接在 Isaac Gym 中调物理约束，训练效率更高。

## DexManipNet 数据集

基于 ManipTrans，作者把 FAVOR 和 OakInk-V2 等手-物交互数据迁移到机器人手，构建 DexManipNet。

关键规模：

- 61 个任务。
- 3.3K episodes。
- 1.2K objects。
- 1.34M frames。
- 约 600 条复杂 bimanual sequences。

任务包括很多过去机器人灵巧手数据里少见的双手精细操作，例如：

- pen capping。
- bottle cap unscrewing。
- chemical experimentation。
- object rearrangement。

主平台是 Inspire Hand 的模拟 12-DoF 配置，后续也验证了 Shadow Hand、articulated MANO hand、Allegro Hand 等跨 embodiment 可扩展性。

## 实验设置

定量评估使用 OakInk-V2 validation set，手动筛选满足任务完整性和语义相关性的 MoCap 序列，长度 4-20 秒，降采样到 60 fps，排除 deformable 或过大物体，最终约 80 episodes。

指标包括：

- `Er`：object rotation error，单位 degree。
- `Et`：object translation error，单位 cm。
- `Ej`：mean per-joint position error，单位 cm。
- `Eft`：mean fingertip position error，单位 cm。
- `SR`：success rate。成功要求 `Er < 30°`、`Et < 3cm`、`Ej < 8cm`、`Eft < 6cm`；双手任务中任一只手失败就算失败。

实现细节：

- Isaac Gym。
- 4096 parallel environments。
- timestep 1/60s。
- 单台 PC：RTX 4090 + i9-13900KF。
- PPO，batch size 1024，learning rate 5e-4。

## 主要结果

### 1. 优于 RL-combined baselines

| 方法 | Er ↓ | Et ↓ | Ej ↓ | Eft ↓ | SR 单手/双手 ↑ |
|---|---:|---:|---:|---:|---:|
| Retarget-Only | N/A | N/A | N/A | N/A | 4.6 / 0.0 |
| RL-Only | 9.72 | 1.23 | 2.96 | 2.38 | 34.3 / 12.1 |
| Retarget + Residual | 11.58 | 0.79 | 2.54 | 1.74 | 47.8 / 13.9 |
| ManipTrans | 8.60 | 0.49 | 2.15 | 1.36 | 58.1 / 39.5 |

结论：直接 retarget 基本不可用；RL-only 探索慢且精度差；retarget + residual 有提升，但 ManipTrans 因为先学了通用手部模仿器，在单手和双手任务上都明显更好。双手成功率从 13.9 提到 39.5，是最关键的提升。

### 2. 效率远高于 optimization-based 方法

与 QuasiSim 的直接定量比较不完全可行，因为验证集和完整 pipeline 不公开。但作者给了一个代表性例子：对 60 帧的 unseen 单手任务 “rotating a mouse”，ManipTrans 约 15 分钟训练出稳定结果，而 QuasiSim 需要约 40 小时优化。

这说明 ManipTrans 的价值不仅是精度，还在于它能成为数据生成 pipeline，而不是只能做少量离线优化案例。

### 3. 跨手型迁移

作者在不同机器人手上验证：

- Shadow Hand：22 DoF。
- articulated MANO hand：22 DoF。
- Inspire Hand：12 DoF。
- Allegro Hand：16 DoF。

不改网络超参数和 reward weight，也能在单手和双手任务中得到一致、流畅、精确的动作。这说明方法依赖的是人手 keypoints 和机器人关节/指尖对应关系，而不是某一个特定硬件的手工设计。

### 4. 真实硬件 replay

真实部署使用两台 7-DoF Realman arms 和一对升级 Inspire Hands。由于真实手是 6-DoF，而仿真用 12-DoF，作者用 fitting-based 方法把仿真手指轨迹拟合到真实手关节。

展示任务包括 opening toothpaste 等细粒度双手操作：左手稳定持物，右手拇指和食指打开小盖子。这类动作很难通过普通遥操作稳定采集，说明 ManipTrans 生成的数据有潜力服务真实机器人策略学习。

### 5. 消融实验

Tactile/contact information 有三种用途：

- 作为 observation。
- 作为 reward。
- 作为 early termination 条件。

消融显示：

- contact force reward 提升成功率。
- contact observation 加速收敛。
- contact termination 对最终稳定任务完成很重要。

Training strategy 消融显示：

- 放松 gravity、高 friction 和放松 thresholds 有助于复杂双手任务早期收敛。
- 如果不放松阈值，网络可能完全无法收敛。

### 6. DexManipNet 用于 policy learning 仍很难

作者用 DexManipNet 的 bottle rearrangement 任务测试 imitation learning：

| 方法 | SR |
|---|---:|
| IBC | 4.69% |
| BET | 9.69% |
| DP-UNet | 18.44% |
| DP-Trans | 14.69% |

所有方法表现都不算高，说明 DexManipNet 的任务确实难：高自由度灵巧手需要精准 finger control 和稳定 object manipulation，普通 behavior cloning 容易误差累积。

## 论文贡献

1. 提出 ManipTrans 双阶段迁移框架。  
   先学手部运动，再用 residual learning 修正物理交互，避免一开始就在高维接触任务里硬探索。

2. 构建 DexManipNet。  
   规模达到 3.3K episodes / 1.34M frames，覆盖很多过去少见的双手灵巧操作任务。

3. 展示跨 embodiment 可扩展性。  
   可迁移到 Inspire、Shadow、MANO、Allegro 等不同手型。

4. 展示真实硬件 replay 可行性。  
   尽管还不是闭环策略部署，但说明生成轨迹有实际硬件落地潜力。

## 和相关工作的差异

与直接 retarget 相比：ManipTrans 不相信几何姿态映射本身足够，而是在仿真中用 residual policy 修正接触和物体运动。

与 RL-only 相比：它避免从零探索复杂双手任务，第一阶段给了强动作先验。

与 QuasiSim 相比：它不是每条 trajectory 做漫长优化，而是用预训练 + residual policy 提供更高吞吐的数据生成方式。

与真实 teleoperation 相比：它不需要人工在线控制，可以复用已有 MoCap / hand-object 数据集，适合规模化。

## 局限

1. 依赖 MoCap 和物体模型质量。  
   如果交互 pose 噪声过大，或者 articulated object 模型不准确，迁移可能失败。

2. 仍主要在仿真中生成轨迹。  
   真实硬件部分是 replay 展示，不等于已经训练出闭环真实策略。

3. 接触建模仍是弱点。  
   contact reward 和 tactile-like features 很有用，但真实触觉、摩擦、软物体和微小装配误差仍可能带来 gap。

4. Policy learning benchmark 表现偏低。  
   DexManipNet 虽然提供了数据，但如何有效训练高成功率闭环策略仍是开放问题。

## 对具身智能研究的启发

1. 数据生成 pipeline 和 policy learning 同样重要。  
   灵巧操作不是只有模型架构问题，先要有物理可行、规模足够、任务多样的数据。

2. “先模仿手，再修正物体交互”是一个很实用的分解。  
   它降低了双手高维接触任务的探索难度，也保留了人类动作自然性。

3. Residual learning 很适合 human-to-robot transfer。  
   人类数据给出粗动作结构，residual 模块处理 morphology gap 和物理约束。

4. 双手灵巧操作会成为检验具身智能数据质量的硬任务。  
   单手抓取和物体搬运已经不够，pen capping、unscrewing、toothpaste opening 这类任务更能暴露接触、协同和精度问题。

## 我的理解

ManipTrans 的贡献不在于提出一个很复杂的模型，而在于把“把人类双手 MoCap 变成机器人手数据”这件事做成了一个效率很高的工程 recipe。它承认直接 retarget 不够，也承认从零 RL 太贵，于是把问题切成 hand motion prior 和 interaction residual 两块。

我觉得它和 EgoScale、DreamDojo 这类工作有一个共同趋势：人类数据越来越不只是视觉预训练材料，而是可以被加工成机器人动作、世界模型或操作先验。ManipTrans 做的是其中最底层、最物理的一环：把人手操作轨迹变成机器人手能执行的数据。

## 可继续追的问题

- DexManipNet 能否训练出强闭环真实机器人 policy，而不只是仿真 replay？
- 如果加入真实 tactile sensor 数据，contact reward 是否能更可靠？
- 对 deformable objects、工具使用、液体等任务，当前 residual framework 是否还够？
- 能否把 ManipTrans 生成的数据和 VLA / diffusion policy / world model 预训练结合？
- 对低 DoF 手或非人形末端执行器，human hand motion prior 的价值会下降多少？

## 可复用笔记

- 核心 recipe：hand trajectory imitation pretraining + residual interaction fine-tuning。
- 核心数据集：DexManipNet，61 tasks，3.3K episodes，1.34M frames，约 600 bimanual sequences。
- 核心结果：双手任务 SR 从 Retarget + Residual 的 13.9 提到 ManipTrans 的 39.5。
- 核心工程点：4096 Isaac Gym 并行环境，RTX 4090 单机可跑，适合做数据生成 pipeline。
- 核心判断：human MoCap 到 dexterous robot 的关键不是几何 retarget，而是 residual physics correction。

