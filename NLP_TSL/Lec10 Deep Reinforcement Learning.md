# 强化学习简介

机器学习从监督信号的角度可以分成几大类
1. supervised learning
2. unsupervised learning
3. reinforcement leaerning
![[Pasted image 20260512103607.png]]


强化学习的分类：
1. Model-free approach
	1. value-based
	2. policy-based
2. Model-based approach

# Value-based RL
## Q-Learning
价值函数Q是对将来回报的估计，即根据策略 $\pi$  在状态 $s_t$ 下采取行动 $a_t$ 的价值。
定义：$Q^{\pi}(s, a) = E\left[r_{t+1} + \gamma r_{t+2} + \gamma^2 r_{t+3} + \cdots \mid s, a\right]$
Bellman 方程：$Q^{\pi}(s, a) = E_{s',a'}\left[r + \gamma Q^{\pi}(s', a') \mid s, a\right]$

![[Pasted image 20260512111555.png]]

## DQN
DQN的基本思想是用深度神经网络来计算Q，采用的是基于价值(value-based)的方法
在 Nature 上发表的论文中，DQN被用来玩48个不同的 Atari Game
$$loss = \left( r + \gamma \max_{a'} Q(s', a', \mathbf{w}) - Q(s, a, \mathbf{w}) \right)^2$$
**$Q(s, a, \mathbf{w})$**：当前神经网络预测的“在状态 $s$ 下采取动作 $a$ 能获得的长期奖励”。
括号左半部分 $r + \gamma \max Q(...)$，利用了**贝尔曼方程**，认为当下的奖励 $r$ 加上未来最好的预期回报，才是更接近真相的评分。
神经网络的训练目标是通过不断缩小“预测值”和“目标值”之间的均方误差，来学习如何更准确地评估每一个动作。

### 1. 奖励裁剪与归一化 (Reward Clipping / Normalization)

- **方法**：将奖励限制在一个感知的范围内（例如 -1, 0, 1）。
    
- **目的**：解决尺度未知问题，确保梯度平稳，让神经网络训练更健壮。
    

### 2. 经验回放 (Experience Replay)

- **出处**：最早在 NIPS 版本的 DQN 中提出。
    
- **原理**：AI 将学到的经验 $(s, a, r, s')$ 存入一个“池子”，训练时随机抽取一批（Batch）数据来学习。
    
- **作用**：打乱了数据的顺序，**打破了连续帧之间的相关性**，让数据更接近独立同分布。
    

### 3. 冻结目标网络 (Freeze Target Q-network)

- **出处**：这是 **Nature 版本**的核心贡献。
    
- **公式解读**：注意公式里的 $Q(s', a', \mathbf{w^-})$。
    
    - **$\mathbf{w}$**：是当前正在更新的神经网络参数。
        
    - **$\mathbf{w^-}$**：是一个“冻结”的参数副本，每隔几千步才同步一次。
        
- **作用**：在计算“目标值（Target）”时使用一套不动的参数。这就像射击时**把靶子固定住**，而不是让靶子随着你的瞄准准心到处乱动，从而极大地提高了训练的稳定性。


