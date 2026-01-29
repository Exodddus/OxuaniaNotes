
# Computational Problems

## Decision Problem
可以确定为Yes/No的问题
- 判定一个正整数是否为质数
	- 枚举：从2枚举到$\sqrt N$。该方法时间复杂度为$O(\sqrt N)$，最多只能做出$1/log(N)$量级的优化。
	- 可证明可以采用$O(log(N)^k)$的方法

## Optimization Problems

计算最优解
- 着色问题
- 背包问题
- 如果目标值是一个整数值，可以变换为查询一系列的Decision Problem

### Solution Problems
输出的是一个具体的解（如路径、组合）

### Value Problems
输出的是解对应的最优值（如最短距离、最大利润）


论证你提供的方法和最优解差距20%以内的思路：
1. 证明提供的解与最优解的差距不超过20%；
2. 找到与最优解差距达到20%一个例子。


# Algorithms

## Exact Algorithms
1. 分治
2. Backtracking
3. 动态规划

## Heuristic Algorithms
启发式算法——小聪明的算法

### Greedy Algorithms
并不一定能做出与最优解差距的分析
如果分析不出来，就叫Greedy Algorithms
从不可信解开始，保留最优性逐步优化

### Local Search
找一个可信解，然后采用迭代法，在小范围内逐次改进
微积分中找极值的最速下降法

## Approximation Algorithms
近似算法，如分析NP完备问题

## Parrallel Algorithms
并行算法，压缩计算时间，但同时可能占用更多的计算资源



# AVL Tree

![[AVL定义.png]]
假设$n_h$为高度为$h$的AVL树的最小节点数，则可以通过递归定义 $n_h = n_{h-1} + n_{h-2}$
初始条件为：$n_0 = 1$，$n_1=2$。

![[AVL树高度计算.png]]
