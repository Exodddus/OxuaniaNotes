# Game Theory
## 定义
zero-sum game: payoff matrix 各项的和为0
假设一个射门-守门的情境，射入球门则攻方得一分，守方减一分；未射入球门则攻方减一分，守方得一分。这样就形成了一个零和博弈。
![[Payoff_Matrix.png]]
## Pure Strategy & Mixed Strategy

Pure Strategy: 确定的策略
Mixed Strategy: 随机决定行动策略
问题来了：如何决定混合策略的收益？
![[Strategies.png]]
收益的定义公式如下：
假设R，C对应的两位玩家做出的选择是独立的
![[RewardFunction.png]]

对于玩家p, 最大化收益的函数 $lb = \max_{p} \min_{q} V_R(p, q)$ 。