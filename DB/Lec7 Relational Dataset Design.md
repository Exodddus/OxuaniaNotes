# 函数依赖

设关系模式 **R(U)**，其中 **U** 是所有属性的集合。对于 **U** 的两个子集 **X ⊆ U** 和 **Y ⊆ U**，如果在关系 R 的**任意一个可能的关系实例 r** 中都满足：
- 对于任意两个元组 t1、t2 ∈ r，如果 t1[X] = t2[X]（X 的值相同），则必然有 t1[Y] = t2[Y]（Y 的值也相同）
则称 **Y 函数依赖于 X**，记作 **X → Y**。**X** 称为**决定因子**（determinant）或**左部**。**Y** 称为**依赖因子**或**右部**。

**平凡函数依赖**（Trivial FD）： 如果 Y ⊆ X，则 X → Y 一定是平凡的（如 A → A、AB → A）。
**非平凡函数依赖**（Non-trivial FD）：Y 不完全包含于 X（如 A → B，其中 B ∉ A）。 一般这才是我们真正关心的依赖。

# Normal Forms
## BCNF

一个关系模式 **R** 处于 **BCNF**，当且仅当对于 **R** 中的**每一个非平凡函数依赖（non-trivial FD）**     X → Y，都满足：**X 是 R 的一个 superkey（超键）**。

也就是说，任何一个决定其他任一属性的“决定因子”（determinant）都必须能唯一标识整条记录。
任何只有两个属性的模式都遵守 BCNF。

**依赖保留**(dependency preservation)：假设关系 $R$ 有函数依赖 $F$，分解得到的子关系 $R1,R2,…Rn$ 以及对应的函数依赖 $F1,F2,…Fn$ ，若$(F1∪F2∪⋯∪Fn)^+=F^+$，那么称该分解是依赖保留的。

## Third NF

关于函数依赖集合 $F$ 的关系 $R$ ，对于所有的在 $F^+$ 的函数依赖 $α→β$，其中 $α⊆R,β⊆R$，至少满足以下条件之一时，称 $R$ 遵循**第三范式**(third normal form，以下简称 **3NF**)：

- $α→β$ 是一个平凡的函数依赖
- $α$ 是模式 $R$ 的超键
- 在 $β−α$ 内的每个属性 $A$ 是 $R$ 的（可能是不同的）候选键的一个成员
	- **候选键**：**最小**的超键 - 它本身是一个超键，而且去掉任何一个属性后都不是超键了
 3NF 的前两个条件和 BCNF 相同，只是新增了第 3 个条件，可以让那些左侧不是超键的非平凡函数依赖也有机会符合这个范式。

# Functional-Dependency Theory

**阿姆斯特朗公理**
- **自反律**(reflexivity rule)：如果 $α$ 是一组属性，且 $β⊆α$，那么 $α→β$ 有效
- **增广律**(augmentation rule)：如果 $α→β$ 有效，且 $γ$ 是一组属性，那么 $γα→γβ$ 有效
- **传递律**(transitivity rule)：如果 $α→β$, $β→γ$ 均有效，那么 $α→γ$ 有效


# 练习题

The following functional dependency set F holds on the relation $R(A,B,C,D,E)$:

$F={AB→C, C→D}$, what is the **candidate key** of R?

A. AB     B. ABE     C. AC     D. AE

答案：B

The following functional dependency set F holds on the relation $R(A,B,C,D,E)$: $F={AB\rightarrow C, C\rightarrow D}$, Which decomposition is **lossless-join**? 
A. $R(A,B,C,D,E) \rightarrow R1(A,B,C,E)\ R2(C,D)$
B. $R(A,B,C,D,E) \rightarrow R1(A,B,D)\ R2(A,B,C,E)$
C. $R(A,B,C,D,E) \rightarrow R1(A,B,C)\ R2(A,B,D,E)$
D. $R(A,B,C,D,E) \rightarrow R1(A,B,C)\ R2(C,D,E)$

答案：A B C

The following functional dependency set F holds on the relation $R(A,B,C,D,E)$: $F={A->B , A->C , AB->D, C->D }$. The **Canonical Cover** of F is 
A. $F={A->BC , AB->D, C->D}$
B. $F={A->BC , AB->D}$
C. $F={A->B , A>C, C->D}$
D. $F={A->BC , C->D}$

答案：D

Which relation is not in BCNF? 
A. $R(A,B,C,D,E),\ F=\{AB\rightarrow C,\ AB\rightarrow DE\}$ 
B. $R(A,B,C,D,E),\ F=\{ABC\rightarrow DE\}$ 
C. $R(A,B,C,D,E),\ F=\{AB\rightarrow CD\}$ 
D. $R(A,B,C,D,E),\ F=\{ABC\rightarrow D,\ D\rightarrow E\}$

答案：C D

Following functional dependency set F holds on the relation $R(A,B,C,D,E)$: $F=\{AB\rightarrow CD,\ C\rightarrow D,\ D\rightarrow E\}$, The Canonical Cover of F is 
A. $F=\{AB\rightarrow CD,\ C\rightarrow D,\ D\rightarrow E\}$ 
B. $F=\{A\rightarrow CD,\ D\rightarrow E\}$ 
C. $F=\{AB\rightarrow C,\ C\rightarrow D,\ D\rightarrow E\}$ 
D. $F=\{AB\rightarrow C,\ AB\rightarrow D,\ C\rightarrow D,\ D\rightarrow E\}$

答案：C

Following functional dependency set F holds on the relation $R(A,B,C,D,E)$: $F=\{AB\rightarrow CD,\ C\rightarrow D,\ D\rightarrow E\}$, Which decomposition is dependency preserving ? 
A. $R(A,B,C,D,E)$ $\rightarrow$ $R1(A,B,C,D)$ $R2(A,B,E)$ 
B. $R(A,B,C,D,E)$ $\rightarrow$ $R1(C,D)$ $R2(A,B,C,E)$ 
C. $R(A,B,C,D,E)$ $\rightarrow$ $R1(D,E)$ $R2(A,B,C,D)$ 
D. $R(A,B,C,D,E)$ $\rightarrow$ $R1(A,B,C)$ $R2(C,D,E)$

答案：C D