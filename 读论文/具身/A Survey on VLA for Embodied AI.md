![[Pasted image 20260210114401.png]]

![[Pasted image 20260210122052.png]]

![[Pasted image 20260210122116.png]]

## Components of VLA

### RL
**DQN** 证明了直接从高维像素输入学习策略的可能性 。
RL 轨迹（状态、动作、奖励序列）与**序列建模**问题高度契合，适用于 Transformer 架构，如 Decision Transformer (DT) 和 Trajectory Transformer (TT) 。
**RLHF**通过人类反馈强化学习，使模型符合人类偏好并提高安全性，如 SEED。
Reflexion 利用语言反馈代替传统的权重更新，适用于具身决策。
LLM 可以为 Agent 设计出优于专家人工编写的奖励函数

### 预训练视觉表示 PVR
视觉编码器提供了物体类别、位置和可操作性（Affordance）等关键状态信息。
- **CLIP**：通过 4 亿对图文数据训练，建立视觉与文本信息的丰富关联，被广泛用作视觉编码器 。
- **R3M & VIP**：利用视频的时序关系进行预训练，通过时间对比学习捕捉时序关联。
- **MVP & RPT**：采用掩码自编码器（MAE）技术，通过重建被遮掩的图像块进行自监督学习。
- **DINOv2**：通过自监督学习提取非常清晰的特征图，不需要任何标签。是目前 OpenVLA 等最前沿模型首选的视觉骨架，对物体的几何边缘和语义分割处理得非常好。
- **I-JEPA**：非生成式方法，通过预测表征空间中的块嵌入来捕获底层图像特征。
- 


### Video Representations

视频本质上是随时间变化的图像序列，其具备的多视图特性（同一物体在不同角度的快照）为提取丰富的 3D 空间信息提供了可能。
- **NeRF (Neural Radiance Fields)**：通过视频流提取 3D 神经辐射场。代表作如 **F3RM** 和 **3D-LLM**，它们能够从视频中提炼出物体的 3D 几何特征，这对于机器人在复杂空间中避障和精准抓取非常有帮助。
- **3D Gaussian Splatting (3D-GS)**：比 NeRF 更先进的技术，在视觉质量和渲染速度上均有显著提升。

### Dynamics Learning

旨在赋予模型理解“因果关系”和“物理演变”的能力，理解动作（Action）与状态变化（State Change）之间的关联。
#### 前向动力学
预测执行某个动作后，环境会变成什么样？即 $\hat{s}_{t+1} \leftarrow f_{fwd}(s_t, a_t)$。这有助于模型进行“预判”。通常作为预训练任务或辅助损失函数。
#### 逆动力学
为了从状态 $A$ 变成状态 $B$，需要采取什么动作？即 $\hat{a}_t \leftarrow f_{inv}(s_t, s_{t+1})$。这常用于从海量无标注视频中推断出对应的机器人动作。
常用于为只有图像而无动作标注的原始视频生成动作标签。

### 世界模型
#### LLM-induced World Models

#### Visual World Models

### Reasoning



