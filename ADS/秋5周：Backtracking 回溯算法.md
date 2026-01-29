# N-Queen Problem

$N >= 4$ 时始终有解

# the Turnpike Reconstruction Problem

在数轴上给定$N$个点的坐标，计算两两之间的距离，可以在 $N^2$ 的时间复杂度内做到；
那么如果给定两两间距（ $N$ 个点共 $N(N-1)/2$ 个间距），假设$x_0$位于原点，能否计算恢复所有点的原始坐标？是否有多组解？

> 找到问题的关键性质，从而限制可能解的规模。

按当前剩下的最大距离确定点的可能位置，同时检验其他距离能否满足，如果出现冲突情况就回溯。

pseudo-polynomial 时间复杂度，看起来是多项式形式的，但是和最长距离有关系。有没有多项式算法至今仍是未解决的问题。
有 $O(2^N N log(N))$ 的确定算法。 

# Tic-tae-toe 井字棋

![[Tic-tae-toe.png]]
采取哪种策略获胜可能性更大？
**极小化极大值策略** 
（与机器学习国际化课的博弈收益 [[Lec1 Game Theory and Randomized Algorithms]] 类似）

![[MinimaxStrategy.png]]
$W_k$代表在当前棋局下$k$方获胜可能的数目，如上图中Human有两行两列四种情况。
人类希望评估函数取尽可能小，计算机希望评估函数尽可能大。从后往前推导可以得出第一步时计算机希望采取的策略。
![[GameInitialStrategy.png]]
## $\alpha$ 剪枝与 $\beta$ 剪枝

![[alphaPruning.png]]

