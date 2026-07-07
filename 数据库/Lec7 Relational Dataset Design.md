# Lec7 Relational Database Design

## 1. 什么是“好”的关系模式

E-R 图转换成关系模式后，仍要检查表结构是否合理。课件关注三个核心目标：

- 每个关系模式本身处于良好范式；
- 分解必须是**无损连接（lossless join）**；
- 最好还能**保持依赖（dependency preservation）**。

一个糟糕的模式通常会产生：

- **信息重复**：同一事实存储多次；
- **插入异常**：没有另一类事实时，无法插入当前事实；
- **更新异常**：同一事实的多份副本必须同时修改；
- **删除异常**：删除一个事实时意外丢失另一个事实。

例如把教师与院系合成：

```text
inst_dept(ID, name, salary, dept_name, building, budget)
```

若满足：

$$
ID \to name, salary, dept\_name
$$

$$
dept\_name \to building, budget
$$

则一个院系的 `building、budget` 会随每位教师重复存储。根源是 `dept_name` 能决定其他属性，却不是 `inst_dept` 的超键。

合理分解为：

```text
instructor(ID, name, salary, dept_name)
department(dept_name, building, budget)
```


---

## 2. Decomposition 与 Lossless Join

### 2.1 分解不一定正确

若把：

```text
employee(ID, name, street, city, salary)
```

分为：

```text
employee1(ID, name)
employee2(name, street, city, salary)
```

当不同员工同名时，自然连接可能产生原关系中不存在的组合，即**伪元组（spurious tuples）**。这是一种有损分解。

注意：连接结果出现更多元组，意味着不确定性更大，因此反而丢失了信息。

### 2.2 无损连接的定义

把关系模式 $R$ 分解成 $R_1$ 和 $R_2$。对于每一个满足约束的合法实例 $r(R)$，若：

$$
\pi_{R_1}(r) \bowtie \pi_{R_2}(r)=r
$$

则该分解是**无损连接分解**。

如果：

$$
r \subsetneq \pi_{R_1}(r) \bowtie \pi_{R_2}(r)
$$

则连接产生伪元组，分解有损。

### 2.3 二元分解的无损判定

对 $R=R_1\cup R_2$，在函数依赖集 $F$ 下，分解无损当且仅当下列至少一个依赖属于 $F^+$：

$$
(R_1\cap R_2)\to R_1
$$

或

$$
(R_1\cap R_2)\to R_2
$$

直观解释：公共属性必须能唯一决定至少一个子关系的全部属性。也就是公共属性至少是某个子关系的超键。

---

## 3. First Normal Form（1NF）

若关系中每个属性的域都是**原子的（atomic）**，每个单元格只保存一个不可再分的值，则关系满足 1NF。

不符合 1NF 的典型设计：

- 在一列中存逗号分隔的多个电话号码；
- 重复列 `phone1、phone2、phone3`；
- 单元格中嵌套一张表。

多值属性应拆成独立关系，例如：

```text
instructor_phone(ID, phone_number)
```

“原子”取决于应用语义。若系统只整体使用电话号码，它可以视为原子值；若要分别查询国家码与区号，则应进一步拆分。

课件后续重点是 3NF、BCNF 与 4NF；2NF 只出现在范式层级中，没有展开算法。

---

## 4. Functional Dependency（函数依赖）

### 4.1 定义

设关系模式为 $R$，属性集 $\alpha\subseteq R$、$\beta\subseteq R$。若对 $R$ 的任意合法实例 $r$ 中任意两个元组 $t_1,t_2$，都有：

$$
t_1[\alpha]=t_2[\alpha]\Rightarrow t_1[\beta]=t_2[\beta]
$$

则称 $\beta$ 函数依赖于 $\alpha$，记作：

$$
\alpha\to\beta
$$

- $\alpha$：决定因素（determinant），即左部；
- $\beta$：被决定属性集，即右部。

函数依赖是对**所有合法关系实例**的业务约束，而不是从当前一小份数据“碰巧没有冲突”就能断定的规律。

### 4.2 与键的关系

$K$ 是 $R$ 的超键，当且仅当：

$$
K\to R
$$

$K$ 是候选键，当且仅当：

- $K\to R$；
- $K$ 的任何真子集都不能决定 $R$。

函数依赖比键约束更一般。例如 `dept_name -> building` 可以成立，但 `dept_name` 不必是整个 `inst_dept` 的键。

### 4.3 Trivial FD

若 $\beta\subseteq\alpha$，则 $\alpha\to\beta$ 对所有关系都成立，称为**平凡函数依赖**。

例如：

$$
\{ID,name\}\to\{ID\}
$$

非平凡依赖才真正表达额外业务约束。

---

## 5. Closure of Functional Dependencies

### 5.1 $F^+$

由函数依赖集 $F$ 逻辑蕴含的所有函数依赖构成 $F$ 的闭包，记作 $F^+$。

例如：

$$
F=\{A\to B,\ B\to C\}
$$

则 $F^+$ 中还包括 $A\to C$、$AB\to C$ 等依赖。

### 5.2 Armstrong 公理

Armstrong 公理是正确且完备的：只会推出成立的依赖，并且能推出所有逻辑蕴含的依赖。

1. **自反律（reflexivity）**

   若 $\beta\subseteq\alpha$，则：

   $$
   \alpha\to\beta
   $$

2. **增补律（augmentation）**

   若 $\alpha\to\beta$，则对任意 $\gamma$：

   $$
   \gamma\alpha\to\gamma\beta
   $$

3. **传递律（transitivity）**

   若 $\alpha\to\beta$ 且 $\beta\to\gamma$，则：

   $$
   \alpha\to\gamma
   $$

### 5.3 常用派生规则

- **合并律（union）**

  $$
  \alpha\to\beta,\ \alpha\to\gamma
  \Rightarrow \alpha\to\beta\gamma
  $$

- **分解律（decomposition）**

  $$
  \alpha\to\beta\gamma
  \Rightarrow \alpha\to\beta,\ \alpha\to\gamma
  $$

- **伪传递律（pseudotransitivity）**

  $$
  \alpha\to\beta,\ \gamma\beta\to\delta
  \Rightarrow \alpha\gamma\to\delta
  $$

---

## 6. Attribute Closure（属性集闭包）

### 6.1 定义与算法

$\alpha$ 在 $F$ 下的闭包记作 $\alpha^+$，表示由 $\alpha$ 能函数决定的全部属性。

算法：

```text
result := α
repeat
    for each β -> γ in F
        if β ⊆ result
            result := result ∪ γ
until result 不再变化
return result
```

### 6.2 例子

设：

$$
R=(A,B,C,G,H,I)
$$

$$
F=\{A\to B,\ A\to C,\ CG\to H,\ CG\to I,\ B\to H\}
$$

求 $(AG)^+$：

1. 初始：$AG$
2. 用 $A\to B$、$A\to C$：得到 $ABCG$
3. 用 $CG\to H$：得到 $ABCGH$
4. 用 $CG\to I$：得到 $ABCGHI$

所以 $(AG)^+=R$，`AG` 是超键。再检查 $A^+$ 和 $G^+$ 都不等于 $R$，可知 `AG` 是候选键。

### 6.3 属性闭包的用途

- 判断 $\alpha$ 是否为超键：检查 $\alpha^+\supseteq R$；
- 判断 $\alpha\to\beta$ 是否属于 $F^+$：检查 $\beta\subseteq\alpha^+$；
- 枚举 $F^+$；
- 寻找候选键；
- 检查正则覆盖中的无关属性。

寻找候选键时，一个实用观察是：**从未出现在任何 FD 右部的属性，不能由其他属性推出，因此必须出现在每个候选键中。** 先以这些属性为起点求闭包，再按需补充属性。

---

## 7. Canonical Cover（正则覆盖）

### 7.1 定义

函数依赖集可能包含冗余依赖或冗余属性。$F$ 的正则覆盖 $F_c$ 应满足：

- $F$ 与 $F_c$ 互相逻辑蕴含，即 $F^+=F_c^+$；
- 没有函数依赖含无关属性；
- 同一左部只出现一次。

它可以理解为与 $F$ 等价的一组精简依赖。

### 7.2 Extraneous Attribute（无关属性）

对 $\alpha\to\beta$：

- 若删除左部某属性后，新依赖仍由 $F$ 蕴含，则该属性在左部无关；
- 若删除右部某属性后，剩余依赖集仍能蕴含原 $F$，则该属性在右部无关。

例子：

$$
F=\{A\to C,\ AB\to C\}
$$

在 `AB -> C` 中，`B` 无关，因为 `A -> C` 已经成立。

### 7.3 计算步骤

重复执行直到 $F$ 不再变化：

1. 用合并律把相同左部的依赖合并；
2. 检查并删除左部无关属性；
3. 检查并删除右部无关属性；
4. 删除可由其他依赖推出的冗余依赖；
5. 删除后可能出现新的相同左部，因此再次合并。

### 7.4 课件例子

$$
F=\{A\to BC,\ B\to C,\ A\to B,\ AB\to C\}
$$

- `A -> BC` 与 `A -> B` 合并后仍为 `A -> BC`；
- `AB -> C` 中 `A` 无关，因为已有 `B -> C`；
- `A -> BC` 中的 `C` 可由 `A -> B` 和 `B -> C` 推出，因此右部 `C` 冗余。

最终：

$$
F_c=\{A\to B,\ B\to C\}
$$

### 7.5 综合例题

设：

$$
R(A,B,C,D,E),\quad
F=\{A\to BC,\ AD\to E,\ B\to C,\ D\to E\}
$$

- 正则覆盖：

  $$
  F_c=\{A\to B,\ B\to C,\ D\to E\}
  $$

- $(AE)^+=ABCE$；
- 候选键为 `AD`。

其中 `AD -> E` 冗余，因为 `D -> E` 已成立；`A -> BC` 中的 `C` 可经 `A -> B -> C` 推出。

---

## 8. Boyce-Codd Normal Form（BCNF）

### 8.1 定义

关系模式 $R$ 关于函数依赖集 $F$ 属于 BCNF，当且仅当对 $F^+$ 中每个依赖 $\alpha\to\beta$，至少满足：

- $\alpha\to\beta$ 是平凡依赖；
- $\alpha$ 是 $R$ 的超键。

一句话：**每个非平凡函数依赖的左部都必须是超键。**

`inst_dept` 不属于 BCNF，因为：

$$
dept\_name\to building,budget
$$

但 `dept_name` 不是整个关系的超键。

### 8.2 BCNF 分解公式

若 $R$ 中存在违反 BCNF 的非平凡依赖 $\alpha\to\beta$，则分解为：

$$
R_1=\alpha\cup\beta
$$

$$
R_2=R-(\beta-\alpha)
$$

若先保证 $\alpha\cap\beta=\varnothing$，也可写为：

$$
R_1=\alpha\beta,\qquad R_2=R-\beta
$$

该分解无损，因为公共属性 $\alpha$ 能决定 $R_1$。

### 8.3 分解算法

```text
result := {R}
while result 中存在不满足 BCNF 的 Ri
    选择 Ri 上违反 BCNF 的非平凡依赖 α -> β
    用 (α ∪ β) 和 (Ri - (β - α)) 替换 Ri
return result
```

检查子关系是否满足 BCNF 时，应使用 $F^+$ 在该子关系上的**投影**，不能只看原依赖集中恰好写出的 functional dependencies。

### 8.4 例题

$$
R(A,B,C,D),\quad F=\{A\to B,\ B\to CD\}
$$

- $A^+=ABCD$，候选键为 `A`；
- `B -> CD` 违反 BCNF，因为 `B` 不是 $R$ 的超键；
- 分解为：

  ```text
  R1(B, C, D)
  R2(A, B)
  ```

- `B` 是 $R_1$ 的键，`A` 是 $R_2$ 的键，因此两者都属于 BCNF；
- 公共属性 `B` 能决定 $R_1$，所以分解无损。

---

## 9. Dependency Preservation（依赖保持）

设 $R$ 分解为 $R_1,R_2,\dots,R_n$，$F_i$ 是 $F^+$ 在 $R_i$ 上的投影。若：

$$
(F_1\cup F_2\cup\cdots\cup F_n)^+=F^+
$$

则称分解**保持依赖**。

意义：每个约束都能通过检查某一个子关系，或由这些局部依赖直接推导得到，不必先连接多张表。

例子：

$$
R(A,B,C),\quad F=\{A\to B,\ B\to C\}
$$

分解一：

```text
R1(A, B), F1 = {A -> B}
R2(B, C), F2 = {B -> C}
```

- BCNF；
- 无损；
- 保持依赖。

分解二：

```text
R1(A, B), F1 = {A -> B}
R2(A, C), F2 = {A -> C}
```

- 仍然无损且属于 BCNF；
- 但无法从局部依赖推出 `B -> C`，因此不保持依赖。

结论：

- **BCNF 分解总能做到无损；**
- **BCNF 分解不一定能保持依赖。**

经典反例：

$$
R(J,K,L),\quad F=\{JK\to L,\ L\to K\}
$$

候选键是 `JK` 和 `JL`。按 `L -> K` 分解为 `(L, K)` 与 `(J, L)` 后会丢失对 `JK -> L` 的局部检查能力。

---

## 10. Third Normal Form（3NF）

### 10.1 Prime Attribute

- **候选键**：最小超键；
- **主属性（prime attribute）**：至少属于某个候选键的属性。

### 10.2 定义

关系模式 $R$ 属于 3NF，当且仅当对 $F^+$ 中每个 $\alpha\to\beta$，至少满足下列一个条件：

1. 依赖是平凡的；
2. $\alpha$ 是 $R$ 的超键；
3. $\beta-\alpha$ 中每个属性都是主属性。

前两个条件与 BCNF 相同，*第三个条件是 3NF 对 BCNF 的放宽。*

因此：

$$
BCNF\Rightarrow 3NF
$$

反之不一定成立。

### 10.3 课件例子

```text
dept_advisor(s_ID, i_ID, dept_name)
```

$$
F=\{s\_ID,dept\_name\to i\_ID,\ i\_ID\to dept\_name\}
$$

候选键为：

- `(s_ID, dept_name)`；
- `(s_ID, i_ID)`。

`i_ID -> dept_name` 的左部不是超键，因此违反 BCNF；但 `dept_name` 属于候选键，是主属性，所以关系仍属于 3NF。

### 10.4 3NF 综合分解算法

1. 求 $F$ 的正则覆盖 $F_c$；
2. 对每个 $\alpha\to\beta\in F_c$，建立关系模式 $R_i=\alpha\cup\beta$；
3. 若没有任何 $R_i$ 包含原关系的候选键，则额外加入一个候选键关系；
4. 删除被其他关系完全包含的冗余关系。

算法保证：

- 每个结果关系属于 3NF；
- 分解无损；
- 分解保持依赖。

### 10.5 BCNF 与 3NF 的取舍

| 性质 | BCNF | 3NF 综合分解 |
|---|---|---|
| 冗余控制 | 更强 | 略弱 |
| 总能无损分解 | 是 | 是 |
| 总能保持依赖 | 否 | 是 |
| 主要用途 | 尽量消除 FD 引起的异常 | 在无损、依赖保持之间折中 |

---

## 11. Multivalued Dependency（多值依赖）

### 11.1 为什么 BCNF 仍可能不够

考虑：

```text
inst_info(ID, child_name, phone_number)
```

一位教师可以有多个孩子和多个电话号码，并且两组值彼此独立。若某教师有 2 个孩子、2 个电话，就必须保存 4 个组合：

```text
(99999, David,   512-555-1234)
(99999, David,   512-555-4321)
(99999, William, 512-555-1234)
(99999, William, 512-555-4321)
```

该关系可能没有非平凡 FD，因此自动属于 BCNF，却仍有明显冗余和插入异常。这说明需要比 BCNF 更高的范式。

### 11.2 MVD 的直观含义

若对给定的 $\alpha$，属性集 $\beta$ 的取值集合与剩余属性集合相互独立，则有多值依赖：

$$
\alpha\twoheadrightarrow\beta
$$

例如：

$$
ID\twoheadrightarrow child\_name
$$

$$
ID\twoheadrightarrow phone\_number
$$

表示给定教师后，孩子集合与电话号码集合独立组合。

### 11.3 Tuple-Swapping 定义

设 $R$ 的属性划分为 $Y,Z,W$。若对任意两个元组：

$$
(y_1,z_1,w_1),(y_1,z_2,w_2)\in r
$$

必有交叉组合：

$$
(y_1,z_1,w_2),(y_1,z_2,w_1)\in r
$$

则：

$$
Y\twoheadrightarrow Z
$$

由于 $Z$ 与 $W$ 在定义中对称，也同时有 $Y\twoheadrightarrow W$。

### 11.4 FD 与 MVD 的关系

每个函数依赖都是多值依赖：

$$
\alpha\to\beta\Rightarrow\alpha\twoheadrightarrow\beta
$$

但反过来不成立。FD 表示给定 $\alpha$ 后 $\beta$ 只有一个确定值；MVD 表示给定 $\alpha$ 后有**一组独立取值**，且保证这组独立取值都可以取得到。

---

## 12. Fourth Normal Form（4NF）

### 12.1 定义

关系模式 $R$ 关于函数依赖与多值依赖集合 $D$ 属于 4NF，当且仅当对 $D^+$ 中每个多值依赖：

$$
\alpha\twoheadrightarrow\beta
$$

至少满足：

- 该 MVD 是平凡的，即 $\beta\subseteq\alpha$ 或 $\alpha\cup\beta=R$；
- $\alpha$ 是 $R$ 的超键。

一句话：**每个非平凡多值依赖的左部都必须是超键。**

因为每个 FD 都是 MVD，所以：

$$
4NF\Rightarrow BCNF\Rightarrow 3NF
$$

### 12.2 4NF 分解

若 $\alpha\twoheadrightarrow\beta$ 非平凡且 $\alpha$ 不是超键，则把 $R$ 分解为：

$$
R_1=\alpha\cup\beta
$$

$$
R_2=R-(\beta-\alpha)
$$

重复分解直到每个子关系都属于 4NF。该算法保证无损连接。

对 `inst_info`：

```text
inst_child(ID, child_name)
inst_phone(ID, phone_number)
```

两组独立多值事实不再形成笛卡尔组合。

### 12.3 课件例子

若实体关系：

```text
E(A, B, C, D)
```

其中 `A` 是实体标识，`B、C` 是相互独立的多值属性，`D` 是普通属性，则：

$$
D=\{A\to D,\ A\twoheadrightarrow B,\ A\twoheadrightarrow C\}
$$

`A ->> B` 非平凡，且 `A` 不是该展开关系的超键，所以不属于 4NF。分解为：

```text
R1(A, B)
R2(A, C)
R3(A, D)
```

---

## 13. 范式层级与设计目标

课件给出的主线是：

$$
1NF\to 2NF\to 3NF\to BCNF\to 4NF
$$

本章重点可以概括为：

- 1NF：每个属性值原子化；
- 3NF：允许右侧为主属性，以换取总能保持依赖；
- BCNF：每个非平凡 FD 的左侧都是超键；
- 4NF：每个非平凡 MVD 的左侧都是超键。

规范化的最终目标不是单纯追求“最高范式”，而是同时考虑：

1. 减少冗余和更新异常；
2. 保证无损连接；
3. 尽量保持依赖；
4. 维持合理的查询与更新成本。

---

## 14. E-R 建模与规范化

如果 E-R 图正确识别了实体和联系，由它转换出的关系通常已经具有较好的范式。

例如员工实体若同时包含：

```text
employee(..., department_name, building)
```

且有：

$$
department\_name\to building
$$

说明 `department` 很可能本应被单独识别为实体，而不是把院系属性塞进员工实体。

真实设计仍可能不完美，因此 E-R 建模和规范化应相互校验：

- E-R 模型从业务语义出发识别对象与联系；
- 规范化从依赖出发检测冗余和异常。

---












## 15. Denormalization（反规范化）

为了读取性能，有时会有意识地保存冗余数据。例如展示课程及先修课信息需要连接 `course` 与 `prereq`，可以选择：

- 建立反规范化关系，预先保存课程与先修课的组合；
- 建立物化视图（materialized view）。

收益：

- 读取更快；
- 减少频繁连接。

代价：

- 占用额外空间；
- 更新更慢；
- 必须维护副本一致性；
- 手工维护还会增加程序错误风险。

反规范化应是**基于性能测量的有意识选择**，而不是跳过正确逻辑设计的借口。

规范化也不能发现所有坏设计。例如按年份建 `earnings_2024、earnings_2025`，或把年份变成 `earnings_2024、earnings_2025` 等列，即使每张表形式上属于 BCNF，也会导致跨年查询困难，并要求每年修改模式。更合理的通用结构通常是：

```text
earnings(company_id, year, amount)
```

---

## 16. 解题模板

### 16.1 求候选键

1. 找出未出现在任何 FD 右部的属性，把它们作为候选键必含部分；
2. 求其闭包；
3. 若闭包未覆盖 $R$，尝试加入其他属性；
4. 一旦闭包覆盖 $R$，检查最小性；
5. 更换可替代属性，寻找其他候选键。

### 16.2 判断 BCNF / 3NF

1. 先求候选键与主属性；
2. 对每个非平凡 FD $X\to Y$ 求 $X^+$；
3. 若 $X^+=R$，左部是超键，满足 BCNF；
4. 若左部不是超键，则违反 BCNF；
5. 对 3NF 再检查 $Y-X$ 是否全部为主属性。

### 16.3 判断二元分解是否无损

1. 求公共属性 $X=R_1\cap R_2$；
2. 求 $X^+$；
3. 若 $R_1\subseteq X^+$ 或 $R_2\subseteq X^+$，分解无损。

### 16.4 判断是否保持依赖

1. 求 $F^+$ 在各子关系上的投影 $F_i$；
2. 检查原 $F$ 中每个依赖能否由 $\bigcup F_i$ 推出；
3. 若某个原依赖必须连接子关系后才能验证，则不保持依赖。

### 16.5 做 BCNF 分解

1. 找一个违反 BCNF 的 $X\to Y$；
2. 分成 $XY$ 与 $R-(Y-X)$；
3. 分别计算各子关系上的依赖；
4. 对仍不满足 BCNF 的子关系继续分解。

### 16.6 做 3NF 综合分解

1. 求正则覆盖；
2. 每个 $X\to Y$ 建一个 $XY$ 关系；
3. 检查是否有关系包含原关系候选键；
4. 若没有，补一个候选键关系；
5. 删除被其他关系包含的子关系。

---

## 17. 课件练习整理

### 练习 1：候选键

$$
R(A,B,C,D,E),\quad F=\{AB\to C,\ C\to D\}
$$

`A、B、E` 不会由其他属性推出，必须属于候选键；

$$
(ABE)^+=ABCDE
$$

所以候选键是 `ABE`。

### 练习 2：无损分解

同样对：

$$
F=\{AB\to C,\ C\to D\}
$$

下列分解为无损：

```text
A. R1(A,B,C,E), R2(C,D)
B. R1(A,B,D),   R2(A,B,C,E)
C. R1(A,B,C),   R2(A,B,D,E)
```

原因分别是：

- A 的公共属性 `C` 能决定 `CD`；
- B 的公共属性 `AB` 能经 `AB -> C -> D` 决定 `ABD`；
- C 的公共属性 `AB` 能决定 `ABC`。

而 `R1(A,B,C), R2(C,D,E)` 的公共属性 `C` 不能决定任一完整子关系，因此不能由给定 FD 保证无损。

### 练习 3：正则覆盖

$$
F=\{A\to B,\ A\to C,\ AB\to D,\ C\to D\}
$$

因为 $A\to C\to D$，`AB -> D` 冗余。合并相同左部后：

$$
F_c=\{A\to BC,\ C\to D\}
$$

### 练习 4：判断 BCNF

$$
R(A,B,C,D,E),\quad F=\{ABC\to D,\ D\to E\}
$$

由 `ABC -> D -> E` 可知 `ABC` 能决定全部属性，因此是超键；但 `D -> E` 的左部 `D` 不是超键，所以该关系不属于 BCNF。

### 练习 5：依赖保持

$$
R(A,B,C,D,E),\quad F=\{AB\to CD,\ C\to D,\ D\to E\}
$$

以下两种分解都保持依赖：

```text
C. R1(D,E),   R2(A,B,C,D)
D. R1(A,B,C), R2(C,D,E)
```

- C 中 `D -> E` 在 $R_1$ 检查，`AB -> CD` 与 `C -> D` 在 $R_2$ 检查；
- D 中 `AB -> C` 在 $R_1$ 检查，`C -> D`、`D -> E` 在 $R_2$ 检查，并可推出 `AB -> CD`。

---

## 18. 本章最重要的结论

- 函数依赖是对所有合法实例成立的业务约束。
- $X^+$ 是判断超键、FD 蕴含、候选键与正则覆盖的核心工具。
- 二元分解无损的关键：公共属性能决定至少一个子关系。
- BCNF 要求每个非平凡 FD 的左部都是超键。
- BCNF 分解总能无损，但不一定保持依赖。
- 3NF 比 BCNF 弱一点，却总能得到无损且保持依赖的分解。
- BCNF 仍无法消除独立多值事实形成的组合冗余，因此需要 MVD 与 4NF。
- 4NF 要求每个非平凡 MVD 的左部都是超键。
- 规范化是为了减少异常；反规范化只能在明确衡量性能与一致性代价后进行。


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
