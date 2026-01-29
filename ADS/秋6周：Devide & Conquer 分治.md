# Closest Point Problem

给定平面上的N个点，找到其中最近的两个点。
### 穷举法

计算全部$N(N-1)/2$对的距离

### 分治法

分成三部分：左半边，右半边，跨越两个区域的中间地带。

![[分治法时间复杂度证明.png]]
可以通过$O(N)$的时间复杂度找到中间情况的最小项。
时间复杂度$O(Nlog(N))$。

## 分治法时间复杂度证明

### Substitution 代换法

先猜后证
![[代换法证明时间复杂度.png]]


> 主定理！！主定理！！

### $T(N)=aT(N/b)+f(N)$

三种解法：
1. Substitution Method
2. Recursion-tree Method
3. Master Theorem

#### Substitution Method
先猜后证

e.g. ![[时间复杂度递推代换法.png]]
最后一定要推出系数为$c$，如果是$c+1$之类的就是不对的


#### Recursion-tree Method

$T(N)=3T(N/4)+\Theta(N^2)$

![[解时间复杂度的递归树法.png]]

$$
\begin{aligned}T\left(N\right) & =\sum_{i=0}^{\log_4N-1}\left(\frac{3}{16}\right)^icN^2+\Theta(N^{\log_43})<\sum_{i=0}^\infty\left(\frac{3}{16}\right)^icN^2+\Theta(N^{\log_43}) \\ & =\frac{cN^2}{1-3/16}+\Theta(N^{\log_43})=O(N^2)\end{aligned}
$$

### Master Method

$$T(N)=aT(N/b)+f(N)$$
其中$f(N)$是一个关于$N$的函数
#### 形式一

1. 如果对于某一常数 $\epsilon$，$f(N)=O(N^{log_b a-\epsilon}))$，则 $T(N)=\Theta(N^{log_b a})$
2. 如果 $f(N)=\Theta(N^{log_b a})$，则 $T(N)=\Theta(N^{log_b a}logN)$
3. 如果对于某一常数 $\epsilon$，$f(N)=\Omega(N^{log_b a+\epsilon}))$，且**正则条件**成立——对于常数 $c$，有 $af(\frac{N}{b})<cf(N)$，那么$T(N)=\Theta(f(N))$

#### 形式二

1. 若对于常数 $\kappa<1$，有 $af(N/b)=\kappa f(N)$ , 则 $T(N)=\Theta(f(N))$
2. 若对于常数 $K>1$, 有 $af(N/b)=K f(N)$, 则 $T(N)=\Theta(N^{log_b a})$
3. 若 $af(N/b)=f(N)$，有