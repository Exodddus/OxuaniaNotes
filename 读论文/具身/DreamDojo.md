# DreamDojo: A Generalist Robot World Model from Large-Scale Human Videos

## 基本信息

- 论文：DreamDojo: A Generalist Robot World Model from Large-Scale Human Videos
- 作者/机构：NVIDIA, HKUST, UC Berkeley, UW, Stanford, KAIST, UT Austin 等
- 时间：arXiv v1, 2026-02-06；PDF 首页日期 2026-02-09
- 项目页：dreamdojo-world.github.io
- 主题：机器人世界模型、人类视频预训练、latent action、具身智能、视频扩散模型

## 一句话总结

DreamDojo 试图把大规模第一视角人类视频变成机器人世界模型的预训练来源：先从 4.4 万小时人类交互视频中学习通用物理和交互知识，用连续 latent actions 作为统一的伪动作标签，再通过少量目标机器人数据后训练和自回归蒸馏，得到可以实时预测、用于遥操作、策略评估和模型规划的机器人视频世界模型。

## 核心问题

机器人世界模型的目标是：给定当前状态和动作，预测未来状态。视频世界模型里，未来状态通常表现为后续视频帧。

这篇文章关注的痛点有三个：

- 真实机器人数据贵、覆盖窄，很难覆盖开放世界里的物体、场景和接触交互。
- 大量互联网/人类视频没有动作标签，直接做无动作视频预测会学到一些物理外观，但很难学到“动作 -> 后果”的因果关系。
- 机器人控制是连续、高维、接触丰富的动作空间，不能像游戏或驾驶那样只靠离散控制信号。

作者的判断是：虽然人手和机器人有 embodiment gap，但许多物体交互背后的物理规律相通。因此，大规模第一视角人类视频可以给机器人世界模型提供广覆盖的物理和交互先验。

## 方法主线

论文整体流程可以分为四步：

1. Human Video Pretraining：用 In-lab、EgoDex、DreamDojo-HV 三类人类第一视角视频预训练。
2. Robot Post-Training：在目标机器人数据上后训练，使模型适配具体 embodiment 和动作空间。
3. Autoregressive Distillation：把较慢的 teacher 世界模型蒸馏成少步数、自回归、实时的 student。
4. Applications：用于 unseen environments 的世界预测、live teleoperation、policy evaluation 和 model-based planning。

## 数据集：DreamDojo-HV

作者构建了一个非常大的第一视角人类视频集合 DreamDojo-HV，并和其他数据源组成混合预训练数据。

关键数字：

- In-lab：55 小时，13.9k trajectories，35 skills，1 scene。
- EgoDex：829 小时，30k trajectories，194 skills，5 scenes。
- DreamDojo-HV：43,827 小时，1,135k trajectories，约 6,015 skills，约 1,135k scenes。
- 总混合数据：44,711 小时，1,179k trajectories，超过 6,015 skills 和 1,135k scenes。

论文强调它相比已有机器人世界模型常用数据有明显规模优势：更长的视频时长、更多技能、更多场景。DreamDojo-HV 覆盖家居、工业、零售、教育、行政等多种环境，也包含长时序、多子任务的日常活动。这个数据多样性是模型 OOD 泛化的主要来源。

## 关键技术 1：连续 Latent Actions 作为统一伪动作

最大难点是人类视频没有精细动作标签。作者没有用手部姿态估计作为主要动作标签，而是训练一个 latent action model，从相邻帧中自监督提取动作表示。

具体做法：

- latent action model 是一个 VAE 风格的时空 Transformer。
- encoder 输入相邻两帧，输出低维连续动作嵌入。
- decoder 接收前一帧和动作嵌入，重建后一帧。
- 由于存在信息瓶颈，嵌入被迫压缩“从前一帧到后一帧发生了什么动作”。
- latent action 维度为 32；模型约 700M，训练 400k steps。

这套设计的意义在于：

- 不依赖人手、关节、机器人动作格式，适合不同 embodiment。
- 比 action-free 视频预测更能保留动作因果关系。
- 比 MANO/retargeted action 更容易扩展到互联网级或众包视频。

我认为这是本文最核心的技术选择：DreamDojo 并不是简单“拿大量视频做预训练”，而是用 latent action 把无标签视频转成 action-conditioned world model 可用的数据。

## 关键技术 2：动作注入与训练目标

DreamDojo 基于 Cosmos-Predict2.5，一个 latent video diffusion model。为了让它适配机器人世界模型，作者做了几处改造：

- 使用相对动作：不直接用绝对关节姿态，而是相对于每个 latent frame 开始时的姿态进行 rebase，降低建模复杂度。
- chunked action injection：WAN2.2 tokenizer 的时间压缩比是 4，所以把连续 4 个动作拼成一个 chunk，注入对应 latent frame，避免把未来动作作为全局条件带来的因果混乱。
- 动作 MLP 零初始化：投影到 timestep embedding 维度后加入 AdaLN 条件，最后一层零初始化，避免一开始破坏预训练模型。
- temporal consistency loss：除了标准 flow matching loss，还加入时间差分一致性损失，使预测速度场的相邻帧差分更接近真值差分。论文设置权重 lambda=0.1。

这几项设计都服务于同一个目标：提高连续动作条件下的视频预测可控性，尤其是接触交互和反事实动作。

## 后训练与蒸馏

后训练阶段用于适配目标机器人：

- 将目标机器人的真实动作 flatten 成序列，通过 action MLP 投影。
- 为匹配新动作空间，重新初始化 action MLP 第一层，并全量 finetune。
- 默认在 GR-1 等目标机器人上用 128 张 H100，50k steps，batch size 512。

蒸馏阶段用于实时交互：

- teacher 是后训练后的 DreamDojo。
- student 初始化自 teacher，但把双向注意力替换为 causal attention，并把 diffusion steps 缩短到少步数，实验中 teacher 35 steps，student 4 steps。
- 训练分 warmup 和 distillation 两阶段，借鉴 Self Forcing，让 student 在训练时使用自己生成的历史上下文，减少长时序 rollout 的误差累积。
- student 使用 12 帧上下文窗口，能比 teacher 更好处理遮挡和相机抖动后的上下文一致性。

结果是 student 可以 640x480 分辨率下达到 10.81 FPS，支持 1 分钟以上实时交互。

## 实验设置

模型规模：

- DreamDojo-2B
- DreamDojo-14B

预训练：

- 初始化自 Cosmos-Predict2.5。
- 人类视频采样比例 In-lab:EgoDex:DreamDojo-HV = 1:2:10。
- 分辨率 640x480，序列长度 13。
- 2B 和 14B 都预训练 140k steps，有效 batch size 1024，使用 256 张 H100。

评测集：

- In-lab Eval
- EgoDex Eval
- DreamDojo-HV Eval
- Counterfactual Eval
- EgoDex-novel Eval
- DreamDojo-HV-novel Eval

指标：

- 有真值视频时使用 PSNR、SSIM、LPIPS。
- novel 场景无真值时使用人类偏好评测，比较 physics correctness 和 action following。

## 主要实验结论

### 1. Latent action 比 action-free 预训练更有效

在 In-lab Eval 和 EgoDex Eval 上，latent action conditioning 明显优于无预训练或 action-free 预训练，并接近使用真实动作捕捉标签的理想设置。论文结论是：大规模人类视频要用于机器人世界模型，关键不是只学视频外观，而是要保留动作因果。

### 2. 数据多样性持续提升泛化

数据消融显示，从 In-lab 到 In-lab+EgoDex，再到加入 DreamDojo-HV，模型在 OOD 场景和 counterfactual action 上整体持续提升。DreamDojo-14B 在多个自动指标上表现最好，说明更大模型容量也能吃下更大的数据多样性。

### 3. 人类偏好评测显示 DreamDojo 明显优于 Cosmos-Predict2.5

在 EgoDex-novel 和 DreamDojo-HV-novel 上：

- DreamDojo-2B 相比 Cosmos-Predict2.5：physics correctness 胜率 62.50%，action following 胜率 63.45%。
- DreamDojo-14B 相比 Cosmos-Predict2.5：physics correctness 胜率 73.50%，action following 胜率 72.55%。
- DreamDojo-14B 相比 DreamDojo-2B：physics correctness 胜率 72.50%，action following 胜率 65.53%。

这说明预训练和模型规模都对复杂 OOD 场景有明显帮助。

### 4. 结构设计消融有效

从基础 Cosmos-Predict2.5 改造开始，逐步加入相对动作、chunked action injection、temporal consistency loss，GR-1 Val 和 Counterfactual Eval 的指标逐步改善。尤其 chunked action injection 的提升很明显，说明“按时间对应地注入动作”对因果控制很关键。

### 5. 蒸馏后速度大幅提升，泛化仍保留

GR-1 Long Eval 上：

- teacher：2.72 FPS，predict length 12，context length 1。
- student：10.81 FPS，predict length 4，context length 12。

student 画质指标略降，但速度接近 4 倍提升，并获得实时流式交互能力。蒸馏后的 w/ pretrain student 仍然优于 w/o pretrain student，说明人类视频预训练带来的泛化能力没有在蒸馏中丢掉。

## 下游应用

### Policy Evaluation

作者在 AgiBot fruit packing 上验证 DreamDojo 是否能作为策略评估模拟器。做法是：

- 训练多个不同 checkpoint 的机器人策略。
- 在真实世界收集闭环 rollout。
- 用相同初始帧在 DreamDojo 中模拟完整 rollout。
- 比较真实成功率和 DreamDojo 成功率。

结果：

- Pearson r = 0.995
- MMRV = 0.003

这说明 DreamDojo 对不同策略 checkpoint 的排序和真实世界高度一致，适合做大规模策略筛选。

### Model-Based Planning

作者 ensemble 多个策略 checkpoint 产生候选动作，用 DreamDojo 批量预测未来视频，再由外部 value model 选择最优提案执行。实验中，DreamDojo 帮助成功率超过最佳单个 checkpoint；相比均匀采样候选动作，成功率接近 2 倍提升。

### Live Teleoperation

作者把 PICO VR 控制器接入 DreamDojo，在 RTX 5090 本地机器上实时遥操作虚拟 G1 机器人。这个应用展示了世界模型可以成为低成本、可交互的机器人数据收集或预演环境。

## 局限

论文自己承认的局限包括：

- 对少见动作仍不够好，比如 slapping、fast waving。
- 在 policy evaluation 中，DreamDojo 的绝对成功率常高于真实世界，说明它还不擅长生成细微失败模式。
- 推理速度仍有工程优化空间。
- 当前不自然支持 multi-view simulation，而多视角对 SOTA 机器人策略很重要。
- 如何在后训练时最大限度保留预训练知识还没有深入研究。

我的补充理解：

- DreamDojo 的评估很强，但仍以视频预测质量和人类偏好为主，距离直接作为真实物理模拟器还有差距。
- latent action 是统一接口，但它的可解释性和可控性边界还需要进一步研究。
- 人类视频到机器人动作之间的 gap 被后训练缓解，但并没有从根本上消除；不同机器人 morphology 下的迁移效果可能差异很大。
- DreamDojo 更像是“视觉世界模型 + 动作条件模拟器”，不是传统意义上可精确计算接触力、状态量的物理引擎。

## 和具身智能研究的关系

这篇论文的重要性在于，它把三个方向接起来了：

- 大规模视频生成模型：用 Cosmos-Predict2.5/WAN2.2 这类强视频先验作为基础。
- 人类视频预训练：用低成本、大覆盖的日常第一视角视频补足机器人数据稀缺。
- 机器人策略闭环应用：不仅做视频生成，而是尝试服务 policy evaluation、planning、teleoperation。

对具身智能来说，DreamDojo 提供了一个清晰路线：机器人世界模型不一定只能从机器人数据学，可以先从人类行为视频学通用物理和交互，再用少量机器人数据对齐 embodiment。

## 我认为最值得记的点

- 大规模人类视频的价值不只是外观多样性，而是接触交互、物体运动、日常任务结构的覆盖。
- action-free video prediction 对世界模型不够，因为它缺少动作因果；latent action 是让无标签视频进入 action-conditioned world model 的关键桥梁。
- chunked action injection 是很实用的设计：动作条件要和视频 latent 的时间粒度对齐，否则未来动作会污染当前预测。
- 蒸馏不是简单加速，还改变了交互形态：从固定 horizon 的 diffusion teacher 变成可流式、自回归、有多帧上下文的 student。
- 真正有应用价值的世界模型需要能排序策略、支持规划、支持实时交互，而不仅是生成“看起来合理”的视频。

## 可继续追的问题

- latent action model 学到的动作空间是否可以被显式操控，还是只能作为中间伪标签？
- 如果目标机器人和人类形态差异更大，例如轮式移动机械臂或四足机器人，DreamDojo 的人类视频先验还能迁移多少？
- DreamDojo 模拟的失败模式偏少，是否会让策略选择过于乐观？
- 多视角世界模型如何与单视角第一人称人类视频预训练结合？
- 能否把 DreamDojo 的视频预测转成更结构化的状态预测，用于更可靠的 MPC 或强化学习？

## 个人评价

DreamDojo 的贡献不只是提出一个更大的机器人视频世界模型，而是给出了“人类视频 -> latent action 伪标签 -> 机器人后训练 -> 实时交互应用”的完整链路。它的亮点在于规模、动作条件设计和应用验证都比较完整。缺点也很明显：视频世界模型仍然容易高估成功率，真实物理可置信度和失败模式覆盖还不足。但作为具身智能里“用大规模人类经验预训练机器人世界模型”的代表工作，它非常值得重点关注。
