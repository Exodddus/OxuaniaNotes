[InternVLA-N1: An Open Dual-System Vision-Language Navigation Foundation Model with Learned Latent Plans](https://internrobotics.github.io/internvla-n1.github.io/)
[Ground Slow, Move Fast: A Dual-System Foundation Model for Generalizable Vision-and-Language Navigation | alphaXiv](https://www.alphaxiv.org/abs/2512.08186)

待学习概念：
- [ ] ViT
- [ ] Diffusion Transformer
- [ ] Q-Former
- [ ] 多模态大模型基本原理

# Dual-VLN
DualVLN 将 VLN 流程解耦为两个互补系统:

| 系统       | 功能      | 速度       | 核心任务                  |
| -------- | ------- | -------- | --------------------- |
| System 2 | 高级推理与规划 | 慢速(2Hz)  | 基于VLM的全局规划,预测中期航点像素目标 |
| System 1 | 低级动作执行  | 快速(30Hz) | 轻量级扩散策略,生成平滑连续轨迹      |

## System 2: VLM-Based Pixel-Goal Grounding

基于Qwen-VL-2.5-7B。
实现功能：
1. 最远像素目标定位(Farthest Pixel Goal Grounding)
    输入自车视角RGB图像序列 + 语言指令，输出图像中下一个导航航点的2D坐标
2. 自主视角调整(Self-Directed View Adjustment)
	模仿人类导航行为(环顾四周、低头看地面)，自主决策何时调整相机角度(Turn Left/Right 15°, Look Up/Down 15°)，确保像素目标预测基于信息丰富的视角

Sys2 输入：
- **当前视觉观测 ：** 机器人当前摄像头拍摄的图像。
- **历史视觉序列：** 过去几帧的图像（通常以视频片段或关键帧的形式输入），帮助模型理解运动趋势。
- **导航指令：** 用户给出的自然语言命令（例如：“穿过客厅，去走廊尽头的浴室”）。
- **潜查询向量 ：** 在训练和提取潜目标时，专门插入到序列末尾的 4 个特殊 token 的 Embedding。这四个向量通过 prompt-tuning 在训练过程中进行调优。
- **（可选）视角调整历史：** 记录之前是否进行过“抬头/低头”等操作，以决定是否需要进一步调整视野。
Sys2 输出：
分为三种类型，模型会根据当前情况在一次前向传播中决定输出哪一种：
- **类型 A：像素目标+潜目标**
    - 输出一个 2D 坐标 $(x, y)$，标注在当前图像上。这代表了为了完成长程指令，机器人近期应该到达的一个“中转点”。
    - 输出 4 个从模型最后一层隐藏状态提取的高维向量（对应插入的 4 个潜查询位置），指导 System 1 生成具体轨迹。
- **类型 C：视角调整动作** 
    - 输出离散的动作标签（如：`LOOK_UP`, `LOOK_DOWN`, `TURN_LEFT`, `TURN_RIGHT`）。当大模型认为当前视野看不清目标或被遮挡时，它会主动要求机器人“转转头”再重新规划。
- **类型 D：停止信号**
    - 当判定已经到达最终目的地时，输出 `STOP` 字符结束任务。

## System 1: Diffusion Transformer Policy
核心架构: **扩散Transformer (DiT)**
输入条件:
1. 低频轨迹潜在特征 (来自System 2的潜在目标表示)
2. 高频RGB输入 (当前观测 + System 2最后一帧)
    
技术细节:
- 生成32个密集航点的平滑轨迹
- 采用 **ViT编码器**提取视觉特征
- 使用自注意力机制融合不同时间步的特征
- 通过**Q-Former**压缩为32个token作为DiT的条件输入


# 训练过程
## 阶段1：Sys2 导航基础训练

改变Qwen-VL-7B全量参数，将其从通用模型转变为导航专用模型
用到的数据

# Benchmark

## Social-VLN
新增了这个Benchmark似乎是Ground Slow, Move Fast 和InternVLA-N1原始文章最大的区别。
现有VLN基准测试(如VLN-CE)仅关注静态环境,缺乏对**动态障碍物处理能力**的评估，因此基于R2R-CE，在Habitat 3.0中引入多个动态人形智能体，并将人形智能体放VLN轨迹沿线


# 其他
SPL：路径长度加权成功率。
计算方式如下：
$$  
SPL = \frac{1}{N} \sum_{i=1}^{N} S_i \cdot \frac{L_i}{\max(P_i, L_i)}  
$$
其中：
- $N$：测试的总次数。
- $S_i$：第 $i$ 次是否成功（成功为 1，失败为 0）。
- $L_i$：从起点到终点的最短路径长度（理想情况）。
- $P_i$：机器人实际走过的路径长度。