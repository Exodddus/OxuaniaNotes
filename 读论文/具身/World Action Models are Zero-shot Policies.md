# World Action Models are Zero-shot Policies

## 基本信息

- 论文: World Action Models are Zero-shot Policies
- 机构: NVIDIA
- 时间: arXiv v1, 2026-02-17; PDF 首页标注 2026-02-19
- 项目页: https://dreamzero0.github.io
- 代码/模型: https://github.com/dreamzero0/dreamzero
- 关键词: World Action Model, WAM, DreamZero, video diffusion policy, zero-shot robotics, cross-embodiment transfer, embodied AI

## 一句话总结

这篇文章提出 DreamZero: 基于预训练视频扩散模型的 World Action Model, 让模型同时预测未来视频和动作。核心主张是: 机器人策略不只应从语言和动作模仿中学 "what to do", 还应利用视频世界模型学 "how the world changes", 从而在新环境、新任务、甚至新机器人形态上获得更强的零样本和少样本泛化。

## 论文要解决的问题

现有 VLA 模型继承了 VLM 的语义先验，所以能理解对象、语言和常识，但在物理动作泛化上仍弱。比如模型可能知道 "untie the shoelace" 是什么，却没有足够的运动和动力学先验来执行这个动作。

作者认为问题在于 VLA 通常直接学习 observation -> action 映射，动作监督稀疏且强依赖示教分布。相比之下，视频包含密集的世界状态变化信号；如果策略能预测未来视觉状态，再把视觉未来对齐到动作，就可以把视频生成模型中的时空/物理先验转化为机器人控制能力。

## 核心概念: World Action Model

WAM 指同时建模未来世界状态和动作的模型。本文中 "world" 主要由未来视频表示，但作者强调未来也可以是触觉、力反馈、3D 表征或潜变量。

DreamZero 建模的是:

```text
p(o_t:t+H, a_t:t+H | o_0:t, c, q_t)
```

其中:

- `o`: 视觉观测/未来视频
- `a`: 动作序列
- `c`: 语言指令
- `q`: 本体状态
- `H`: action horizon

直观上，DreamZero 把策略拆成两件事但放在一个模型里端到端训练:

1. 预测接下来会发生什么视觉变化。
2. 从这个视觉未来中反推出应该执行的动作。

这类似一个隐式 inverse dynamics model, 但不是先生成视频再用另一个 IDM 解动作，而是同一个 DiT 同时 denoise 视频 latent 和动作。

## 方法

### 1. 模型架构

DreamZero 基于 Wan2.1-I2V-14B-480P 视频扩散模型。为了尽量保留视频模型的泛化能力，作者只加入少量机器人相关模块:

- VAE 编码视觉上下文。
- text encoder 编码语言指令。
- state encoder 编码本体状态。
- action encoder/action decoder 处理动作。
- 共享 autoregressive DiT backbone, 同时预测未来视频帧和 action chunk。

如果机器人数据有多视角，作者没有改 backbone, 而是把多视角拼接成单帧输入。

### 2. 为什么用 autoregressive 而不是 bidirectional

作者认为 AR 架构更适合闭环机器人控制:

- 可以用 KV cache 加速推理。
- 可以保留原始视频 FPS, 避免为了对齐语言 caption 而 subsample 视频。
- 可以把历史视觉观测作为上下文，帮助语言-视频-动作三模态对齐。
- 推理时每执行完一个 action chunk, 用真实观测替换预测视频写回 KV cache, 降低自回归视频生成的误差累积。

Bidirectional WAM 在长任务中会遇到 caption 和采样片段不匹配的问题: 语言描述的是完整任务，但固定长度视频片段可能只覆盖任务的一小段；如果通过 subsampling 对齐，又会破坏原生帧率和动作对齐。

### 3. 训练目标

训练使用 flow matching。每个 chunk 内视频 latent 和动作共享 denoising timestep, 模型学习预测 joint velocity。

训练时采用 teacher forcing: 当前 noisy chunk 可以 attend 到前面干净的 chunks, 让模型学习 chunk-wise 的视频和动作生成。

关键点不是单独训练视频预测和动作预测，而是在同一个模型里联合 denoise:

- 视频预测提供密集世界建模监督。
- 动作预测被迫和预测出来的视觉未来对齐。
- 失败时经常表现为 "视频计划错了, 动作忠实执行了错误计划", 说明动作侧对齐较强，瓶颈更多在视觉规划/视频预测。

### 4. 闭环实时控制

原始 14B 视频扩散模型太慢，naive 版本每个 action chunk 约 5.7 秒，无法做响应式控制。DreamZero 用异步闭环执行和多层优化把速度推到约 7Hz。

核心策略:

- 异步执行: 控制器持续执行最近生成的 action chunk, 同时模型基于最新观测生成下一段动作。
- Action horizon: AgiBot 上 48 steps at 30Hz, 每个 chunk 1.6 秒。
- CFG parallelism: conditional/unconditional 两个 forward 分到两张 GPU。
- DiT caching: velocity 方向相近时复用缓存，把有效 DiT steps 从 16 降到约 4。
- torch.compile + CUDA Graphs。
- cuDNN attention, scheduler GPU 化。
- Blackwell 上 NVFP4 quantization, 敏感部分保留 FP8/FP16。
- DreamZero-Flash: 解耦 video/action denoising timestep, 让模型适应 action 单步 denoise。

累计推理加速:

| 优化阶段 | H100 | GB200 |
|---|---:|---:|
| Baseline | 1x | 1.1x |
| + CFG Parallelism | 1.9x | 1.8x |
| + DiT Caching | 5.5x | 5.4x |
| + Torch Compile + CUDA Graphs | 8.9x | 10.9x |
| + Kernel & Scheduler Opts. | 9.6x | 14.8x |
| + Quantization | - | 16.6x |
| + DreamZero-Flash | - | 38x |

## 数据

### AgiBot G1

作者采集了约 500 小时 AgiBot G1 双臂移动操作数据:

- 22 个真实环境: 家、餐厅、超市、咖啡店、办公室、仓库、实验室、酒店等。
- 7,193 episodes。
- 平均 episode 时长约 4.4 分钟。
- 平均每个 episode 约 42.4 个 subtasks。
- 强调任务多样性和真实效用，而不是每个任务大量重复示教。

数据采集策略很有意思: 每天给 teleoperator 一个任务表，每个 episode 选 3 个较粗粒度任务连续执行；某个任务达到 50 个 episodes 后从任务表移除，迫使采集分布不断扩展成长尾。

### DROID-Franka

为了验证公开数据可复现性，作者也在 DROID 上训练 Franka 单臂版本，并开放 checkpoint 和推理代码用于 DROID-sim/PolaRiS 评测。

### Post-training 数据

三个下游任务:

- Shirt folding: 33 小时。
- Fruit packing: 12 小时。
- Table bussing: 40 小时。

## 实验设置

对比基线:

- GR00T N1.6
- pi0.5

每个基线有两种初始化:

- scratch: 只用预训练 VLM 权重，没有额外 robot pretraining。
- pretrained: 使用官方在大量跨 embodiment 机器人数据上预训练的 checkpoint, 再用和 DreamZero 相同数据继续训练。

DreamZero 训练:

- Backbone: Wan2.1-I2V-14B-480P。
- AgiBot 和 DROID 都训练 100K steps, global batch size 128。
- 更新 DiT blocks, state encoder, action encoder, action decoder。
- 冻结 text encoder, image encoder, VAE。
- 默认 action 表示为 relative joint positions。

评测默认是 OOD: 训练数据和评测环境来自不同地理位置，所以 seen task 也不是简单插值，而是在新环境、新物体下执行。

## 主要结果

### Q1: WAM 能否从多样且非重复数据中学到更好策略

AgiBot seen-task 评测中，DreamZero 平均 task progress 达到 62.2%, 最好 pretrained VLA baseline 为 27.4%。作者强调这不是因为 DreamZero 见过更多机器人数据，因为 pretrained VLA 基线反而有数千小时跨 embodiment robot pretraining。

DROID-Franka 上也观察到类似趋势: 只在 DROID 上训练的 DreamZero 超过已经在多机器人数据上预训练过的 VLA 基线。

解释: VLA 需要从状态-动作对中隐式学动力学，而 WAM 通过视频生成目标继承了世界动态先验，所以更能利用异质、长尾、非重复数据。

### Q2: 能否泛化到训练中没有出现过的任务

AgiBot unseen-task:

- from-scratch VLA: 接近 0。
- best pretrained VLA: 16.3% average task progress。
- DreamZero: 39.5% average task progress。

代表任务包括:

- Untie shoelaces
- Remove hat from mannequin
- Draw with pen
- Take out straw
- Cube stacking
- Painting with brush
- Ironing
- Shake hands
- Fold map
- Pulling cart

DROID-Franka unseen-task:

- DreamZero: 49% task progress, 22.5% success rate。
- GR00T N1.6 pretrained: 31% progress, 12.5% success。
- pi0.5 pretrained: 33% progress, 7.5% success。

作者的质性观察: VLA 往往不管指令是什么都倾向于 reach/grasp, 像是在复用 pick-and-place 主导行为；DreamZero 更像先做视觉规划，再执行与视觉计划一致的动作。

### Q3: Post-training 后能否保留泛化

三个任务上 DreamZero 与 pretrained VLA 持平或更好:

| 任务 | DreamZero 表现 | 结论 |
|---|---:|---|
| Shirt folding | 92.5% | 与强基线接近 |
| Fruit packing | 96% | 明显优于 VLA |
| Table bussing | 83% | 与强基线接近或更好 |
| 平均 | 90.5% | 保留新环境泛化 |

这里重要的是评测仍在 unseen environments 中进行，所以 post-training 并没有破坏 DreamZero 的环境泛化。

### Q4: 只用其他 embodiment 的视频能否迁移

作者测试两种 video-only transfer:

- Robot2Robot: YAM -> AgiBot。
- Human2Robot: egocentric human videos -> AgiBot。

数据量很小:

- 9 个 unseen tasks。
- 每个任务 8 条示范, 共 72 条。
- YAM 约 20 分钟。
- Human 约 12 分钟。
- 不使用动作标签，只用视频预测目标。

结果:

| 方法 | Unseen-task task progress |
|---|---:|
| DreamZero | 38.3% +/- 7.6% |
| + Human2Robot video-only transfer | 54.3% +/- 10.4% |
| + Robot2Robot video-only transfer | 55.4% +/- 9.5% |

这说明 WAM 确实可以把视觉经验当作额外世界模型训练信号，即使没有目标机器人的动作标注。

### Q5: 新 embodiment 少样本适配

作者把 AgiBot 上训练的 DreamZero 迁移到新双臂机器人 YAM:

- 只用 55 条 trajectories。
- 11 个任务。
- 约 30 分钟 play data。

迁移后模型还能对新物体和自然语言组合保持较强语言跟随，例如 pumpkin, teddy bear, cup noodles, paper bag 等。作者没有给完整量化表，但强调视频-动作对齐仍然紧密。

### Q6: DreamZero-Flash 是否能降低 denoising steps

Table bussing 上:

| 方法 | Denoising steps | Task progress | Latency |
|---|---:|---:|---:|
| DreamZero | 4 | 83% +/- 6.1% | 350ms |
| DreamZero | 1 | 52% +/- 10.2% | 150ms |
| DreamZero-Flash | 1 | 74% +/- 10.1% | 150ms |

结论: 直接减少 denoising steps 会明显掉性能；通过训练时解耦 video/action timestep, 可以让单步 action denoise 接近 4-step baseline。

## 消融实验

消融在 PnP Easy 上做，所有模型训练 50K steps, batch size 32。

| 问题 | 设置 | Task progress |
|---|---|---:|
| 数据多样性 | DreamZero AR, 14B, repetitive data | 33% +/- 4.2% |
| 数据多样性 | DreamZero AR, 14B, diverse data | 50% +/- 6.3% |
| 模型规模 | DreamZero AR, 5B, diverse data | 21% +/- 4.2% |
| 模型规模 | DreamZero AR, 14B, diverse data | 50% +/- 6.3% |
| VLA 规模 | VLA, 5B, diverse data | 0% |
| VLA 规模 | VLA, 14B, diverse data | 0% |
| 架构 | DreamZero BD, 14B, diverse data | 50% +/- 14.4% |
| 架构 | DreamZero AR, 14B, diverse data | 50% +/- 6.3% |

解读:

- 多样数据优于重复数据。
- WAM 对模型规模更敏感，14B 明显优于 5B。
- 只放大 VLA 不能解决从多样非重复数据中学动作的问题。
- AR 和 BD task progress 接近，但 AR 动作更平滑，且因为 KV cache 推理快 3-4x。

## 作者讨论的局限

1. 计算成本仍高  
   DreamZero 需要 2 张 GB200 才做到约 7Hz, 相比可在消费级 GPU 上 20Hz 以上运行的 VLA 仍重很多。

2. 长程推理不足  
   当前视觉上下文约 6.6 秒，DreamZero 更像 System 1 反应式模型。复杂长程任务可能还需要 System 2 planner 或更长上下文的视频世界模型。

3. 高精度操作仍难  
   钥匙插入、精密装配等亚厘米级任务对行为克隆和数据密度要求高。当前多样化预训练偏向广度，可能牺牲精细任务密度。

4. Cross-embodiment 仍是早期结果  
   人类视频只有 12 分钟，机器人视频 20 分钟；结果有启发性，但还不能证明大规模开放人类视频一定能稳定转化为机器人能力。

5. 失败瓶颈常在视频生成  
   这是优点也是风险: 策略动作忠实跟随视觉计划，但如果视觉计划 hallucinate 或语言跟随失败，机器人也会执行错误动作。

## 我对这篇文章的理解

这篇文章最重要的贡献不是又做了一个更大的机器人策略，而是把 robot policy 的学习对象从 "直接模仿动作" 改成了 "预测世界如何变化并把动作绑定到这个变化上"。这改变了数据利用方式: 不再要求每个任务有大量重复 demonstrations, 而是把长尾、多样、非重复的真实操作片段当作世界动态和 inverse dynamics 的训练材料。

它与 VLA 的差别也很清楚:

- VLA 的优势是语义和语言泛化。
- WAM 的优势是时空和物理动态泛化。
- DreamZero 试图把视频模型的物理先验转化为 action policy, 因此在 unseen motions 上有明显优势。

不过论文里的 "zero-shot policy" 也需要谨慎理解。DreamZero 并不是没有机器人动作数据；它有 500 小时 AgiBot 数据或 DROID 数据。这里的 zero-shot 主要指对新任务、新物体、新环境的零样本泛化，而不是从纯视频直接变成机器人策略。

## 对具身智能研究的启发

1. 数据采集策略可能要从 "每个任务重复很多次" 转向 "真实场景中持续扩展长尾任务分布"。  
   如果模型有强世界模型先验，重复 demonstration 的边际价值可能低于状态-动作对应的多样性。

2. 视频生成质量可能成为机器人策略上限。  
   作者多次观察到动作会忠实跟随生成视频，因此提升视频规划、语言跟随和物理一致性可能直接提升 policy。

3. Cross-embodiment 的突破口可能是 video-only data。  
   传统跨机器人迁移常卡在 action space 和 morphology mismatch。WAM 先吸收视觉行为，再用目标 embodiment 的少量 play data 学 implicit IDM, 这条路很适合利用大规模人类视频。

4. "显式规划" 和 "隐式视频规划" 的边界正在变模糊。  
   DreamZero 不做搜索式 MPC, 但它生成未来视频，本质上已经在内部做了视觉计划。未来可能需要将这种 System 1 视频计划与 System 2 任务规划结合。

5. Real-time world model policy 会成为工程瓶颈。  
   论文花很大篇幅讲 38x 加速，说明这不是可有可无的优化，而是 WAM 能不能成为闭环控制策略的核心条件。

## 可以追问或复现的点

- DreamZero 在更大规模 human egocentric video 上是否还能稳定提升，还是会受视角、手形、交互物体分布影响？
- 如果把视频未来换成 3D scene flow、tactile prediction 或 force prediction, WAM 是否更适合高精度接触任务？
- AR WAM 的 6.6 秒上下文如何扩展到分钟级任务？
- 视频生成错误如何被在线检测和纠正？是否能加入 uncertainty 或 verifier？
- 少样本新 embodiment 适配是否只适用于 morphology 接近的双臂平行夹爪机器人？
- WAM 与 VLA 是否应该融合: VLA 做语义/任务分解，WAM 做短程物理执行？

## 记忆钩子

DreamZero 的核心可以记成三句话:

1. 让机器人先想象世界怎么变，再输出动作。
2. 视频预测把 web-scale 时空先验带进机器人策略。
3. 用真实观测刷新自回归缓存，让视频世界模型变成闭环 policy。

