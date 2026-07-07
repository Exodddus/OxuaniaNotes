# Query Optimization

查询优化的目标是：对同一条 SQL 的多种等价执行方式进行比较，选出代价最低的执行计划。  
同一个查询的不同执行计划，性能差距可能非常大，极端情况下甚至可能从几秒变成几天。

## 1. Query Optimization 在做什么

一个查询通常有两类“可优化空间”：

- **等价表达式**：把原始关系代数表达式改写成逻辑上等价、但更适合执行的形式
- **物理算法选择**：对每个算子选择不同的实现算法，如 nested-loop join、merge join、hash join、索引扫描等

优化思路分成两类：

- **Transformation-based optimization**
  - 重点是使用等价规则改写查询
  - 更偏逻辑层
- **Cost-based optimization**
  - 在多个候选执行计划中估算代价并选最便宜的
  - 更偏物理执行层

## 2. Cost-Based Optimization 的基本流程

代价优化的核心步骤：

1. 用等价规则生成一批**逻辑等价**的表达式
2. 对每个表达式补充不同的物理执行方式，形成多个**query plan**
3. 根据统计信息估算每个计划的代价
4. 选择总代价最小的计划

代价估计依赖两类信息：

- **基础统计信息**
  - 关系中的元组数
  - 块数
  - 元组长度
  - 某属性的 distinct values 数量
- **中间结果统计信息**
  - 某个 selection / join / aggregation 后大约还剩多少行
  - 这就是 **cardinality estimation**

## 3. 关系代数等价规则

这些规则的作用是：在不改变语义的前提下，把查询改写成更高效的执行形态。

### 3.1 Selection 与 Projection

- 合取选择可以拆开：$$\sigma_{\theta_1 \land \theta_2 \land \cdots \land \theta_n}(E)=\sigma_{\theta_1}\bigl(\sigma_{\theta_2}(\cdots \sigma_{\theta_n}(E)\cdots)\bigr)$$
- 多个 selection 可交换：$$\sigma_{\theta_1}(\sigma_{\theta_2}(E))=\sigma_{\theta_2}(\sigma_{\theta_1}(E))$$
- 多个 projection 连续出现时，只保留最后一个即可：$$\pi_{L_1}\bigl(\pi_{L_2}(E)\bigr)=\pi_{L_1}(E), \qquad L_1 \subseteq L_2$$
- selection 可以和笛卡尔积 / theta join 结合改写：$$\sigma_{\theta}(E_1 \times E_2)=E_1 \bowtie_{\theta} E_2$$
- 当选择条件只涉及某个关系时，可以下推到笛卡尔积一侧：$$\sigma_{\theta_1}(E_1 \times E_2)=\sigma_{\theta_1}(E_1)\times E_2$$，其中 $\theta_1$ 只涉及 $E_1$ 的属性
- 若选择条件可拆成分别作用于两侧的部分，则：$$\sigma_{\theta_1 \land \theta_2}(E_1 \times E_2)=\sigma_{\theta_1}(E_1)\times \sigma_{\theta_2}(E_2)$$，其中 $\theta_1$ 只涉及 $E_1$，$\theta_2$ 只涉及 $E_2$

最重要的经验是：

- **尽早做 selection**
  - 先过滤，再 join，可以显著缩小中间结果
- **尽早做 projection**
  - 尽早丢弃不用的列，可以减小元组宽度和 I/O 成本

### 3.2 Join 的等价性质

- theta join 和 natural join 都满足**交换律**
  - theta join：$$E_1 \bowtie_{\theta} E_2 = E_2 \bowtie_{\theta} E_1$$
  - natural join：$$E_1 \bowtie E_2 = E_2 \bowtie E_1$$
- natural join 满足**结合律**：$$ (E_1 \bowtie E_2) \bowtie E_3 = E_1 \bowtie (E_2 \bowtie E_3) $$
- theta join 在满足条件时也可重排：$$ (E_1 \bowtie_{\theta_1} E_2) \bowtie_{\theta_2} E_3 = E_1 \bowtie_{\theta_1} (E_2 \bowtie_{\theta_2} E_3) $$，这里通常要求 $\theta_2$ 只涉及 $E_2$ 与 $E_3$ 的属性

这给优化器带来的价值是：

- 可以尝试不同的 join 顺序
- 可以把“过滤性更强”的关系放前面
- 可以为某些更便宜的 join 算法创造条件

### 3.3 Selection / Projection Pushdown

这是课件反复强调的启发式规则。

- 如果选择条件只涉及某个输入关系的属性，就把它下推到该关系之上
  - 对应公式是 $$\sigma_{\theta}(E_1 \bowtie E_2)=\sigma_{\theta}(E_1)\bowtie E_2$$，其中 $\theta$ 只涉及 $E_1$ 的属性
- 若连接条件两侧各自还有局部过滤条件，则：$$\sigma_{\theta_1 \land \theta_2 \land \theta_3}(E_1 \bowtie E_2)=\sigma_{\theta_1}(E_1)\bowtie_{\theta_3}\sigma_{\theta_2}(E_2)$$，其中 $\theta_1$ 只涉及 $E_1$，$\theta_2$ 只涉及 $E_2$，$\theta_3$ 是连接条件
- 如果 projection 只保留最终需要的列，以及 join 条件中必须用到的列，就尽量提前做：$$\pi_{L_1 \cup L_2}(E_1 \bowtie_{\theta} E_2)=\pi_{L_1 \cup L_2}\bigl(\pi_{L_1 \cup J_1}(E_1)\bowtie_{\theta}\pi_{L_2 \cup J_2}(E_2)\bigr)$$，其中 $J_1,J_2$ 是参与连接条件 $\theta$ 的属性集合

经典收益：

- 降低 join 输入规模
- 降低中间结果的列数
- 降低块传输和 CPU 代价

### 3.4 其他等价规则

课件还给出了更多规则，思路上记住即可：

- union / intersection 满足交换律和结合律：$$E_1 \cup E_2 = E_2 \cup E_1, \qquad E_1 \cap E_2 = E_2 \cap E_1$$；$$ (E_1 \cup E_2)\cup E_3 = E_1 \cup (E_2 \cup E_3) $$；$$ (E_1 \cap E_2)\cap E_3 = E_1 \cap (E_2 \cap E_3) $$
- selection 可分配到 union / intersection 上：$$\sigma_{\theta}(E_1 \cup E_2)=\sigma_{\theta}(E_1)\cup \sigma_{\theta}(E_2)$$；$$\sigma_{\theta}(E_1 \cap E_2)=\sigma_{\theta}(E_1)\cap \sigma_{\theta}(E_2)$$
- projection 可分配到 union 上：$$\pi_L(E_1 \cup E_2)=\pi_L(E_1)\cup \pi_L(E_2)$$
- selection 在满足条件时可下推到 aggregation：$$\sigma_{\theta}\bigl(\gamma_{G,F}(E)\bigr)=\gamma_{G,F}\bigl(\sigma_{\theta}(E)\bigr)$$，其中 $\theta$ 只涉及分组属性 $G$
- outer join 有自己的一套规则
  - full outer join 可交换
  - left / right outer join 一般**不可交换**
  - outer join 通常**不满足普通 join 的很多等价性质**
  - 在某些条件下可以把 outer join 替换成 inner join

所以一条很重要的考试/理解结论是：

- **inner join 的重排空间远大于 outer join**

## 4. Heuristic Optimization（启发式优化）

启发式优化不一定严格比较所有候选计划，而是优先套用经验上“通常更好”的规则。

常见规则：

1. 尽早做 selection
2. 尽早做 projection
3. 避免产生大的中间结果
4. 尽量把选择性强的条件提前
5. 先做能显著缩小结果规模的 join

课件中的例子本质上是在说明：

- 对 $instructor \bowtie teaches \bowtie course$ 这类查询
- 如果先对 `instructor` 做 `dept_name='Music'`
- 再和其他表 join
- 往往比先把大表 join 完再筛选要好得多

## 5. 代价估计与基数估计

### 5.1 常用统计量

课件使用的符号：

- `n_r`：关系 `r` 的元组数
- `b_r`：关系 `r` 占用的块数
- `l_r`：关系 `r` 的元组长度
- `f_r`：blocking factor，即每块可容纳的元组数
- `V(A, r)`：关系 `r` 中属性 `A` 不同取值的数目

这些量是优化器做估算的基础。

### 5.2 Selection 的大小估计

#### 等值选择

若条件为 `A = v`：

- 若 `A` 是 key，则结果大小估计为 `1`
- 若 `A` 不是 key，则结果大小通常估计为：$$\frac{n_r}{V(A,r)}$$

直觉是：

- 假设 `A` 的不同取值大致均匀分布
- 那么某个具体值会命中约 `1 / V(A,r)` 的记录

#### 范围选择

若有 `min(A,r)` 和 `max(A,r)`，则可以根据区间比例估算。  
若还有 histogram，则估算会更准确。

#### 缺省估计

若完全没有统计信息，课件给出的保守假设是：

- 选择后剩余约 $$\frac{n_r}{2}$$

### 5.3 Selectivity

设条件 `θ_i` 的 selectivity 为 `s_i`，表示一条元组满足该条件的概率。

则：

- 对单个条件：结果大小约为 $$s_i \cdot n_r$$
- 对合取 $\theta_1 \land \theta_2 \land \cdots \land \theta_n$：
  - 在**独立性假设**下，selectivity 近似为各自相乘
  - 对应公式是 $$s(\theta_1 \land \cdots \land \theta_n)\approx \prod_{i=1}^{n} s_i$$
- 对析取 $\theta_1 \lor \theta_2$：
  - 用概率公式估计
  - 对应公式是 $$s(\theta_1 \lor \theta_2)\approx s(\theta_1)+s(\theta_2)-s(\theta_1)s(\theta_2)$$
- 对否定 $\neg \theta$：selectivity 约为 $$1-s(\theta)$$

注意：

- 独立性假设常常并不真实
- 但数据库优化器通常必须在“可计算”和“够准确”之间折中

### 5.4 Join 结果大小估计

几个重要结论：

- 笛卡尔积 $r \times s$ 的大小为：$$n_r \cdot n_s$$
- 若是等值连接 $r \bowtie s$，常见估计为：$$\frac{n_r \cdot n_s}{\max(V(A,r),V(A,s))}$$
  - 这里假设按属性 `A` 连接

特殊情况更容易估计：

- 若连接属性是某一侧的 key
  - 每个元组最多匹配一个
- 若某一侧是外键引用另一侧
  - 结果行数常常接近外键所在关系的行数

课件强调：这些估计未必精确，但对比较计划代价已经足够有用。

### 5.5 Projection / Aggregation / Set Operations

- $\pi_A(r)$ 的结果大小可估计为 $V(A,r)$
- $\gamma_A(r)$（按 $A$ 分组聚合）的组数也通常估计为 $V(A,r)$
- union / intersection / difference 的大小估计往往较粗糙，很多时候只能给上界

## 6. Distinct Values 的估计

优化器不仅要估计“有多少行”，还要估计“属性还有多少不同值”。

### 6.1 Selection 后

如果 $\sigma_{A=v}(r)$：

- 则有 $$V(A,\sigma(r))=1$$

如果是 `A` 取若干离散值：

- $$V(A,\sigma(r))$$ 约等于被指定值的个数

如果是范围或一般条件：

- 常用近似：$$V(A,\sigma(r)) \approx V(A,r)\cdot s$$
- 或保守写成：$$\min\bigl(V(A,r), n_{\sigma(r)}\bigr)$$

### 6.2 Join 后

若属性只来自某一个输入关系，则 join 后 distinct values 一般不会增加，常估计为：

- $$\min\bigl(V(A,r), n_{r\bowtie s}\bigr)$$

这些 distinct-values 估计会继续影响后续 selection、join、aggregation 的代价估算。

## 7. Join Order Optimization

### 7.1 为什么难

`n` 个关系的 join 顺序数量增长极快。  
课件列出了一个非常直观的事实：

- `n = 3` 时就有 12 种
- `n = 4` 时有 120 种
- `n = 5` 时有 1680 种
- `n = 10` 时已经大到无法暴力穷举

所以优化器必须避免“把所有 join 顺序全部算一遍”。

### 7.2 Dynamic Programming

核心思想：

- 一个关系子集的最优连接计划，只需要计算一次
- 之后反复复用

也就是说：

- 先求单表的最佳访问方式
- 再求两表子集的最佳 join
- 再扩展到三表、四表……
- 每个子集的最优计划都缓存起来

这就是**动态规划优化 join 顺序**。

课件给出的复杂度结论：

- 若考虑 bushy tree：
  - 时间复杂度约 $O(3^n)$
  - 空间复杂度约 $O(2^n)$
- 若只考虑 left-deep tree：
  - 时间复杂度约 $O(n2^n)$
  - 空间复杂度仍约 $O(2^n)$

left-deep tree 更受欢迎，因为：

- 搜索空间更小
- 更适合流水执行
- 更容易与 indexed nested-loop join 等算法结合

### 7.3 Interesting Sort Orders

有时一个子计划虽然当前更贵，但它产出的**排序顺序**对后续操作有帮助。

例如：

- 用 merge join 算 $r_1 \bowtie r_2$ 也许比 hash join 更贵
- 但结果可能已经按连接键排好序
- 这样后面再和 $r_3$ join 或做 `group by / order by` 时更便宜

所以优化器不能只记录“某个子集的最便宜计划”，还要记录：

- **在若干 interesting order 下的最优计划**

## 8. Nested Subquery 与 Decorrelate

课件把“相关子查询”也放进查询优化里讨论。

示例思想：

- `where exists (...)`
- 子查询里引用了外层变量
- 这种变量叫 **correlation variables**

SQL 的概念执行方式是：

- 外层查询每产生一条候选元组
- 就执行一次内层子查询

这叫 **correlated evaluation**。  
问题是它可能非常慢，因为：

- 子查询调用次数很多
- 容易产生大量随机 I/O

因此优化器通常会尝试：

- 把嵌套子查询改写成 join

这类改写叫：

- **decorrelation**

它的本质是：

- 把“逐行调用子查询”
- 改成“集合化 join 求值”

## 9. Materialized Views

### 9.1 基本概念

**Materialized view** 是指：

- 视图结果不是临时现算
- 而是**预先计算并存储**

适合的场景：

- 某类聚合或连接结果被频繁查询
- 现算代价高
- 基表变化相对没那么频繁

例如：

```sql
create view department_total_salary(dept_name, total_salary) as
select dept_name, sum(salary)
from instructor
group by dept_name
```

如果经常查“每个系的总工资”，把它物化会非常划算。

### 9.2 维护问题

物化视图的核心难点不是“建出来”，而是：

- **基表更新后如何维护**

课件重点讲的是 **incremental view maintenance**：

- 不重算整个视图
- 只根据新增/删除的元组，增量更新视图结果

### 9.3 常见维护思路

#### Join 的维护

如果某个输入关系新增了 `ΔR`，则连接结果新增部分通常可写成：

- 即 $$\Delta R \bowtie S$$

这就是把“变化量”继续向上层表达式传播。

#### 聚合的维护

- `count`
  - 插入时对应组 `+1`
  - 删除时对应组 `-1`
  - 若减到 `0`，删除该组
- `sum`
  - 插入/删除时加减对应值
  - 同时通常还要维护 `count`
- `avg`
  - 不能直接维护平均值
  - 应维护 `sum` 和 `count`，最后再相除
- `min / max`
  - 插入容易
  - 删除麻烦
  - 因为删掉当前最小/最大值时，可能要重新扫描该组找新的极值

## 10. 物化视图与查询优化

如果系统里已经存在物化视图，优化器还会做一件事：

- 判断某个查询能不能直接改写成对物化视图的访问

这会带来：

- 更少的 join
- 更少的 aggregation
- 更少的 I/O

所以查询优化不只是“在基本表上找最优计划”，也包括：

- **view selection / query rewrite using materialized views**

## 11. 其他扩展优化主题

### 11.1 Top-K Queries

例如：

```sql
select *
from r, s
where r.B = s.B
order by r.A asc
limit 10
```

优化重点不是求出**所有结果**，而是尽快找到前 `k` 个。

可能的思路：

- 选取有利于尽早得到前几条结果的 join 算法
- 根据排序列估计阈值，先加入额外过滤条件
- 如果结果不够，再放宽阈值重试

### 11.2 Optimization of Updates 与 Halloween Problem

典型问题：

```sql
update R
set A = 5 * A
where A > 10
```

如果利用 `A` 上的索引扫描满足条件的元组，并且边扫边改：

- 某条元组更新后可能仍满足条件
- 甚至被再次访问、再次更新

这就是 **Halloween problem**。

常见解决方案：

- **延迟更新**
  - 先找出所有待更新元组
  - 再统一回写
- **只在必要时延迟**
  - 若更新不会影响筛选条件相关属性，可立即更新
  - 否则延迟

### 11.3 Join Minimization

有些 join 实际上是冗余的，可以删掉。

例如：

- 连接只是为了验证外键存在性
- 但结果列里并不需要另一张表的属性
- 且另一张表上没有额外过滤条件

这时 join 可能可以安全消除。

### 11.4 Multiquery Optimization

如果一批查询共享公共子表达式，可以考虑：

- 公共部分只算一次
- 多个查询共享结果

但共享并不总是更便宜，因为：

- 公共子表达式本身也可能代价很高
- 单独优化每个查询有时反而更划算

实际系统中常见折中是：

- 先分别优化每个查询
- 再检测是否存在可共享的公共子计划

### 11.5 Parametric Query Optimization

某些查询在编译时参数未知：

```sql
select *
from r natural join s
where r.a < $1
```

不同参数值可能对应不同最优计划。

常见方案：

- 每次运行时重新优化
- 预先生成一组对不同参数区间最优的计划
- 若系统判断一个计划对大多数参数都够好，就直接缓存并复用

### 11.6 Adaptive Query Processing

如果运行时发现真实情况与估计偏差很大，系统可以：

- 动态切换算子
- 中途重优化
- 重新选择执行计划

这说明：

- 查询优化并不一定只发生在执行前
- 也可以在执行过程中自适应修正

## 12. 这一讲最该记住什么

### 12.1 主线

查询优化 = **逻辑改写 + 物理计划选择 + 代价估计**

### 12.2 三个最高频考点

- **等价规则**
  - selection / projection pushdown
  - join 的交换与结合
- **基数估计**
  - `n_r`、`V(A,r)`、selectivity
  - join / selection 后结果大小的估算
- **join order optimization**
  - 搜索空间爆炸
  - 动态规划求最优连接顺序

### 12.3 工程上最关键的直觉

- 越早过滤越好
- 越早删掉无用列越好
- 避免大中间结果
- 优化器做决定时严重依赖统计信息质量
- 统计信息不准，优化器就可能选错计划
