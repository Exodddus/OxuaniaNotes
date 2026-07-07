# ARM4R: Pre-training Auto-regressive Robotic Models with 4D Representations

## 基本信息

- 论文: Pre-training Auto-regressive Robotic Models with 4D Representations
- 作者: Dantong Niu, Yuvan Sharma, Haoru Xue, Giscard Biamby, Junyi Zhang, Ziteng Ji, Trevor Darrell, Roei Herzig
- 机构: Berkeley AI Research, UC Berkeley
- 时间: arXiv v2, 2025-05-17；ICML 2025
- 项目页: https://arm4r.github.io/
- 关键词: robot pre-training, 4D representations, 3D point tracking, human video, auto-regressive model, low-level control
- 本地文件说明: `LITERATURE/ARM4R.pdf` 存在，但 PDF trailer / xref 损坏，无法由 `pdftotext` 抽取；本笔记正文参考 arXiv 官方 v2 PDF。

## 一句话总结

ARM4R 提出一种面向机器人控制的自回归预训练模型：先在大规模人类第一视角视频上学习 3D point tracks 这种低层 4D 表示，再用少量机器人视频适配相机和 embodiment，最后把同一个自回归结构微调成机器人 proprioceptive state/action 预测器；实验显示这种 4D 表示比单纯 2D 轨迹或 VLA 高层语言预训练更适合低层机器人控制。

## 核心问题

机器人领域也想复用 foundation model 的预训练思路，但一直有两个难点：

1. 机器人数据太少。  
   OpenX 等数据集已经很大，但和互联网文本、图像、视频相比仍小得多，而且机器人硬件、相机、动作空间差异很大。

2. 通用 VLM/VLA 的预训练目标不够低层。  
   很多 VLA 继承的是图文问答、caption 或语言解码能力，这有助于理解语义指令，但不直接对应机器人控制需要的几何、轨迹、接触和精确空间推理。

ARM4R 的核心判断是：如果要从人类视频迁移到机器人，不应该只学高层语义，而应该学一种更接近物理世界的低层表示。论文选择的表示是 4D representation，也就是视频中一组 3D 点随时间运动的轨迹。

## 方法总览

ARM4R 全称是 Auto-regressive Robotic Model with 4D Representations。它分三阶段训练：

| 阶段      | 数据                        | 任务                      | 作用                     |
| ------- | ------------------------- | ----------------------- | ---------------------- |
| Stage 1 | 人类第一视角视频 Epic-Kitchens100 | 预测 3D point tracks      | 学低层 4D 物理/几何动态         |
| Stage 2 | 少量目标机器人场景视频               | 继续预测 3D point tracks    | 适配机器人相机、场景和 embodiment |
| Stage 3 | 机器人示范数据                   | 预测未来 robot state/action | 转成低层机器人控制策略            |

最关键的是，三个阶段都保持相似的自回归 next-token prediction 形式：前两阶段预测未来 3D 点轨迹，第三阶段把点轨迹 token 换成机器人 proprioceptive state token，预测未来机器人状态。

## 为什么 4D 表示适合机器人

论文用一个简单的机器人运动学观察来支撑这个选择：对于开链机械臂，机械臂上任意点的位置变化可以由关节状态对应的一系列 SE(3) 变换描述。也就是说，机器人 proprioceptive states 和机器人身体/附着物上的 3D points 之间存在共享几何结构，至少可以通过线性/刚体变换联系起来。

这让 3D point tracks 成为人类视频和机器人控制之间的桥：

- 人类视频里没有机器人关节状态，但可以通过 2D tracking + monocular depth 得到 3D 点轨迹。
- 机器人控制里有 proprioception，它同样描述一个身体在 3D 空间中的运动。
- 如果模型先学会预测 3D 点如何运动，再迁移到预测机器人 state，就比直接从图像语义跳到 action 更自然。

## 模型架构

ARM4R 的输入在时间步 `t` 包含三类信息：

- 语言指令 `l`。
- 当前图像 `i_t`。
- 当前 3D point coordinates `p_t`，或控制阶段的当前 robot state `s_t`。

输出是下一时间步的 3D point coordinates `p_{t+1}`，或控制阶段的未来 robot state `s_{t+1}`。

主要模块包括：

| 模块 | 设计 |
|---|---|
| Language Encoder | frozen CLIP text encoder + learnable projection |
| Image Encoder | ViT-Base，使用 CrossMAE 在 ImageNet + OpenX 上预训练 |
| Point / State Encoder | 2 层 MLP 编码 3D points 或 robot states |
| Attention Pooling | 融合图像 token 和 point/state token |
| Causal Transformer | 随机初始化的 ViT-Base 风格 Transformer，自回归预测 |
| Decoder | 2 层 MLP，把预测 token 解码成点坐标或机器人状态 |

控制微调时，模型预测未来 16 个动作/状态，但评估时只执行第一个预测。这是一种 receding horizon 的用法。

## 三阶段训练

### Stage 1：人类视频 3D point tracking 预训练

Stage 1 使用 Epic-Kitchens100 的 76K 人类第一视角视频。这个数据集包含大量人-物交互，覆盖 97 个动词和 300 个名词类别。

因为人类视频没有真实 3D 点轨迹标签，作者用 off-the-shelf 3D point tracker 生成伪标签。具体做法是在第一帧初始化一个 `g x g` 的点网格，然后在整段视频中跟踪这些点的 3D 坐标。论文实现中使用 SpatialTracker 作为 3D point tracker。

这一阶段的目标不是学机器人，而是学“物理世界里点如何随时间运动”。尤其是第一视角视频包含大量手、物体、相机共同运动，比静态图像更接近机器人操作所需的动态理解。

### Stage 2：机器人视频 3D point tracking 微调

人类视频和机器人视频有明显差异：

- 人类第一视角相机常常运动，机器人相机可能固定。
- 人手和机器人夹爪/机械臂的形态不同。
- 人类视频里的交互模式和机器人操作轨迹不同。

所以 Stage 2 继续做同一个 3D point tracking 任务，但数据换成目标机器人 setup 的视频。论文强调这一步只需要对每个机器人 setup 做一次，不需要为每个任务单独做，数据量约为 Stage 1 的 5-10%。

它的作用是把 Stage 1 学到的通用 4D 动态表示适配到机器人相机和场景分布。

### Stage 3：机器人控制微调

最后，模型被微调为机器人控制策略。此时当前点轨迹 `p_t` 被替换为当前机器人 state `s_t`，输出点轨迹 `p_{t+1}` 被替换为未来机器人 state/action。

真实和仿真实验中，ARM4R 使用 end-effector control，预测：

- 末端 Cartesian position。
- 末端 rotation。
- gripper binary value。

训练目标仍是 L1 预测损失，架构基本保持一致。

## 实验设置

论文在仿真和真实机器人上评估：

1. RLBench 仿真。  
   12 个任务，每个任务 25 个 validation episodes，5 个随机种子。

2. 真实 Kinova Gen3。  
   7-DoF Kinova Gen3 + Robotiq 2F-85 gripper，13 个真实任务，分为 pick、destack、stack、pick-and-place、push 五类。

3. 真实 Franka Emika Panda。  
   用于验证跨机器人 generalization。

对比方法包括 Image-BC、C2FARM-BC、ManiGaussian、PerAct、ATM、LLARVA、pi0-FAST、OpenVLA、MVP、RPT、Octo 等。

## 主要结果

### 1. RLBench 仿真

在 RLBench 12 个任务上，ARM4R 平均成功率为 59.47%，高于 PerAct 的 55.33%、LLARVA 的 48.33%、ManiGaussian 的 48.00%。

有意思的是，PerAct 直接使用仿真环境里的 voxel 信息，这在真实世界很贵；ARM4R 则通过 3D point tracking 预训练学习 3D 结构，仍然取得更高平均表现。

论文也指出两个失败类型：

- Unnatural rotation：例如 `put knife` 这类由仿真 motion planner 生成的不自然全臂旋转，ARM4R 不容易学。
- Lack of precision：例如 `screw bulb` 需要非常精确插入，模型仍困难。

### 2. 真实 Kinova 多任务

在 13 个真实任务上，ARM4R 平均成功率 83.1%，显著高于：

- OpenVLA：37.2%
- pi0-FAST：21.2%
- LLARVA：18.3%
- ATM：6.4%

任务包括：

- pick cube up，不同颜色方块。
- destack cubes。
- stack cubes。
- pick toys then place to target。
- play basketball。
- push red button / push red then blue。

这里最重要的结论是：ARM4R 只用人类视频做 Stage 1 预训练，却能超过许多用机器人数据或 VLA 组件预训练的方法。作者认为关键来自低层 4D 表示，而不是高层语言解码器。

### 3. 消融：Stage 1 比 Stage 2 更关键

消融比较了四种训练组合：

- Stage 1 + Stage 2 + Stage 3。
- Stage 1 + Stage 3。
- Stage 2 + Stage 3。
- Stage 3 only。

结果显示：

- Stage 1 人类视频预训练对所有任务都有明显帮助。
- Stage 2 机器人视频微调也有帮助，但增益小于 Stage 1。
- 最好结果来自三阶段都使用。

这说明只要表示选得对，人类视频确实可以成为机器人预训练的主要资源，而不只是辅助数据。

### 4. 与其它预训练方式比较

在真实 Kinova 的 pick、stack、destack 三个任务上，ARM4R 与 MVP、RPT、Octo、ATM、OpenVLA、LLARVA 比较。ARM4R 在平均意义上最好，例如 pick cube 达到约 96%，stack cubes 约 61.3%，destack cubes 约 94.7%。

这组实验说明：

- 只预训练视觉 encoder 不够。
- 只用 2D point trajectory 不够。
- 只用 OpenX VLA 风格预训练也不一定够。
- 3D point tracks 这种 4D 低层表示更贴近机器人控制。

### 5. 跨机器人泛化

作者测试了从 Kinova setup 迁移到 Franka setup。两者机器人不同，安装方式不同，相机/工作台配置也不同。结果显示，加入 Epic 人类视频预训练和 Kinova 视频微调后，在 Franka 控制微调上的平均表现提升约 19.6%。

这说明 ARM4R 学到的不是某个单一机器人硬件的技巧，而是有一定跨 embodiment 迁移能力的 4D 动态表示。

### 6. 鲁棒性分析

论文还在真实 Kinova 上测试了动态扰动和噪声：

- 机械臂下降到 1/3 或 2/3 时移动目标方块，成功率只小幅下降。
- 光照降低到 50%，背景有人走动/窗帘移动，表现仍较稳。
- 桌面附近加入 distractor objects 时下降更明显。

作者认为这可能是因为 attention pooling 会让模型更关注目标区域，而桌面 distractor 离目标近，更容易干扰点轨迹和动作预测。

## 论文贡献

1. 提出 ARM4R。  
   一个使用 4D representations 做机器人预训练的自回归模型。

2. 证明人类视频可用于低层机器人预训练。  
   关键不是只学语义，而是学 3D points over time。

3. 设计三阶段迁移路线。  
   人类视频点轨迹预训练、机器人视频点轨迹适配、机器人控制微调，三者目标形式连续。

4. 在仿真和真实机器人上验证效果。  
   包括 RLBench、Kinova 和 Franka，超过多种 2D、3D、VLA 和预训练 baseline。

5. 展示 4D representation 的跨机器人和鲁棒性潜力。  
   在动态扰动、光照变化、背景干扰和跨机器人设置中都有一定稳定性。

## 和相关工作的差异

与 VLA 相比：ARM4R 不主要依赖语言 decoder 的高层视觉语言预训练，而是从低层 3D point tracking 任务学物理动态。

与 ATM / Track2Act 相比：ARM4R 使用 3D point tracks，而不是 2D point trajectories，因此有更强空间感和遮挡处理能力。

与 PerAct / RVT 类 3D 方法相比：ARM4R 不要求真实部署时直接获得昂贵 voxel / point cloud 输入，而是通过预训练学会从视频中建模 3D 动态。

与 OpenX 机器人数据预训练相比：ARM4R 的一个激进点是，Stage 1 只用人类视频，也能在真实机器人任务上超过用机器人数据预训练的若干 baseline。

## 局限

1. 3D tracks 在 camera coordinates 中预测。  
   这会把物体运动和相机运动耦合在一起，模型不容易 disentangle camera motion、object motion 和 camera intrinsics。

2. 单视角仍有遮挡问题。  
   未来可能需要 multi-view fusion，减少单视角遮挡和视角变化带来的不稳定。

3. 均匀网格点不一定最有效。  
   固定 grid 会浪费很多背景点，也可能给小物体不够分辨率。未来可以只跟踪 relevant / moving points。

4. 精细接触任务仍困难。  
   `screw bulb` 这类需要高精度插入的任务暴露出 ARM4R 对接触和精确几何控制的不足。

5. 依赖 off-the-shelf 3D tracker 的伪标签质量。  
   如果 tracker 在人类视频中出错，预训练信号也会受影响。

## 和具身智能研究的关系

ARM4R 是一篇非常直接回应“机器人预训练到底该学什么”的论文。它的答案是：学语言和图像语义当然有用，但低层控制更需要 4D 几何动态。

这和很多近期工作可以放在一条线看：

- EgoScale / DreamDojo 强调从人类第一视角视频中学习操作知识或世界模型。
- ManipTrans 把人类双手 MoCap 转成机器人灵巧手轨迹。
- ARM4R 把人类视频中的 3D 点轨迹转成机器人控制预训练。
- D4RT 则进一步推进了通用视频 4D 几何重建和跟踪能力。

共同趋势是：人类视频不再只是语义预训练材料，而是被转化为机器人可用的几何、动作、动态和物理先验。

## 我的理解

ARM4R 最有价值的地方是把“从人类视频学机器人”这件事落到了一个很具体的中间表示上。它没有直接说人类视频能给机器人 action，也没有只拿 VLM feature 来 fine-tune，而是选择了 3D point tracks 这个低层桥梁。

我觉得这个选择很聪明：点轨迹比语言低层，比 raw pixels 结构化，比机器人 proprioception 更容易从人类视频里获得，又和机器人运动学有天然联系。所以它既能吃到互联网人类视频规模，又不会离控制目标太远。

当然，ARM4R 目前还没有解决所有机器人控制问题。尤其是接触丰富、精密装配、软物体和工具使用，还需要更强的触觉、力控和闭环几何推理。但它提供了一个很清晰的方向：机器人 foundation model 的预训练目标应该更接近“世界如何在 3D 中运动”。

## 可继续追的问题

- 如果用 D4RT 这类更强的 4D tracker 替代 SpatialTracker，ARM4R 会提升多少？
- Stage 2 的机器人视频适配能否跨更多 embodiment 复用，而不是每个 setup 都做一次？
- 能否把 4D point tracks 和 tactile / force 数据结合，处理更精细接触任务？
- 对移动操作、双臂操作、灵巧手操作，当前 robot state token 是否足够表达？
- 是否可以从 uniform grid tracking 变成 object-centric / affordance-centric tracking？

## 记忆钩子

1. ARM4R = 用 4D point tracks 预训练机器人自回归模型。
2. 三阶段：人类视频点轨迹 -> 机器人视频点轨迹 -> 机器人控制。
3. 核心判断：3D points over time 比高层 VLM 语义更贴近低层控制。
4. 实验亮点：真实 Kinova 13 任务平均 83.1%，显著超过 OpenVLA、pi0-FAST、LLARVA 和 ATM。
5. 主要局限：camera-coordinate tracks 纠缠相机/物体运动，精细接触和遮挡仍难。
