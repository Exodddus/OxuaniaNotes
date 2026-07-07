# Lec12 Query Processing

本节讨论 SQL 查询从输入到执行的过程，以及数据库如何为 selection、sort、join 等操作选择物理算法并估算代价。

核心思想：

> 同一个关系代数表达式可以有很多执行方法，查询处理器要选择代价较低的 evaluation plan。

---

## 1. Basic Steps in Query Processing

查询处理通常分为三步。

### 1.1 Parsing and Translation

解析 SQL：

- 检查语法；
- 验证 relation、attribute 是否存在；
- 把 SQL 转换成内部表示；
- 进一步转换成 relational algebra。

例如：

```sql
select name, title
from instructor natural join (teaches natural join course)
where dept_name = 'Music' and year = 2009;
```

可转换为：

$$
\Pi_{name,title}
\left(
\sigma_{dept\_name='Music' \land year=2009}
(instructor \bowtie (teaches \bowtie course))
\right)
$$

### 1.2 Optimization

优化器在多个等价 evaluation plans 中选择代价最低的。

一个 evaluation plan 不仅说明关系代数操作顺序，还说明：

- 每个 operation 用什么算法；
- 中间结果是否物化；
- 是否使用索引；
- join 顺序是什么；
- buffer 怎么使用。

### 1.3 Evaluation

query-execution engine 执行选中的 plan，并返回结果。

---

## 2. Measures of Query Cost

总代价通常可以看成 elapsed time，但实际估算时主要关注 disk I/O。

影响因素：

- disk accesses；
- CPU；
- network communication；
- buffer 命中情况；
- 并发查询影响。

课件中为简化，主要使用：

- block transfers；
- seeks。

定义：

- $t_T$：传输一个 block 的时间；
- $t_S$：一次 seek 的时间。

若某算法需要 $b$ 次 block transfer 和 $S$ 次 seek，则代价为：

$$
b \cdot t_T + S \cdot t_S
$$

写 block 通常比读 block 更贵，因为写后可能要读回验证。

课件中很多公式会忽略最终输出写盘成本，因为中间结果可能直接传给父操作。

---

## 3. Selection Operation

### 3.1 A1: Linear Search

线性扫描每个 file block，检查所有 records 是否满足条件。

最坏情况：

$$
b_r \cdot t_T + t_S
$$

其中 $b_r$ 是 relation $r$ 占用的 block 数。

如果 selection 条件在 key attribute 上，找到后可以停止，平均代价约为：

$$
(b_r/2)\cdot t_T + t_S
$$

linear search 适用于任何选择条件，不依赖排序或索引。

注意：没有索引时，对磁盘文件做 binary search 通常不划算，因为会引入大量 seek。

---

## 4. Selections Using Indices

设：

- $h_i$：索引高度；
- $b$：满足条件的记录所在 block 数；
- $n$：满足条件的记录数；
- $m$：保存记录指针的 index blocks 数。

### 4.1 A2: Primary B+-Tree Index, Equality on Key

聚簇 B+-tree 索引，等值查询，且条件属性是 key。

只会返回一条记录：

$$
(h_i + 1)(t_T + t_S)
$$

含义：

- 访问索引路径；
- 再读目标数据 block。

### 4.2 A3: Primary B+-Tree Index, Equality on Nonkey

聚簇 B+-tree 索引，等值查询，但 search key 非唯一。

匹配记录物理上连续，代价：

$$
h_i(t_T+t_S) + t_S + b \cdot t_T
$$

含义：

- 先通过索引定位第一条；
- 然后顺序扫描包含匹配记录的连续 blocks。

### 4.3 A4: Secondary B+-Tree Index, Equality on Key

辅助索引，等值查询，且 search key 唯一。

代价与 A2 类似：

$$
(h_i + 1)(t_T + t_S)
$$

因为只取一条记录。

### 4.4 A4': Secondary B+-Tree Index, Equality on Nonkey

辅助索引，等值查询，但 search key 非唯一。

匹配的 $n$ 条记录可能分布在不同 blocks 上，代价可能很高：

$$
(h_i + m + n)(t_T + t_S)
$$

这体现了 non-clustering index 的问题：

> 匹配很多记录时，可能变成大量随机 I/O。

---

## 5. Selections Involving Comparisons

考虑：

$$
\sigma_{A \ge V}(r)
$$

或：

$$
\sigma_{A \le V}(r)
$$

### 5.1 A5: Primary / Clustering B+-Tree Index, Comparison

如果 relation 按 $A$ 排序：

- 对 $\sigma_{A \ge V}(r)$：
  - 用索引找到第一条 $\ge V$ 的 tuple；
  - 从那里开始顺序扫描。
- 对 $\sigma_{A \le V}(r)$：
  - 直接从文件开头顺序扫描到第一条 $>V$；
  - 不一定需要用索引。

代价类似 A3：

$$
h_i(t_T+t_S) + t_S + b \cdot t_T
$$

### 5.2 A6: Secondary B+-Tree Index, Comparison

使用 secondary index 做范围查询：

- 先扫描 index leaf；
- 再根据 pointers 去取 records。

如果结果很多，每条记录可能一次随机 I/O，因此可能比线性扫描更贵。

---

## 6. Complex Selections

### 6.1 Conjunction

形如：

$$
\sigma_{\theta_1 \land \theta_2 \land \cdots \land \theta_n}(r)
$$

#### A7: 使用一个索引

选择其中最便宜的一个条件先用索引取出候选 tuples，再在内存中检查其他条件。

#### A8: 使用 Composite Index

如果有复合索引，并且条件匹配复合索引的前缀，则可以直接使用。

例如索引 `(dept_name, salary)` 可以很好支持：

```sql
where dept_name = 'Finance' and salary = 80000
```

#### A9: Record Identifiers 求交

如果多个条件都有索引，且索引能返回 record pointers：

1. 分别用每个索引得到 pointer 集合；
2. 求交集；
3. 再读取对应 records；
4. 没有索引的条件在内存中检查。

### 6.2 Disjunction

形如：

$$
\sigma_{\theta_1 \lor \theta_2 \lor \cdots \lor \theta_n}(r)
$$

#### A10: Record Identifiers 求并

只有当所有 disjunct 都有可用索引时才适用。

否则通常使用 linear scan。

### 6.3 Negation

形如：

$$
\sigma_{\neg \theta}(r)
$$

通常用 linear scan。

如果满足 $\neg\theta$ 的记录很少，并且 $\theta$ 有索引，也可以借助索引找到需要排除或保留的部分。

---

## 7. Sorting

排序可以通过：

- 建索引后按索引顺序读取；
- 内存排序；
- external sort-merge。

如果 relation 太大，无法放入内存，通常使用 **external sort-merge**。

---

## 8. External Sort-Merge

设内存有 $M$ 个 page。

### 8.1 Phase 1: Create Sorted Runs

重复执行：

1. 读入 $M$ 个 blocks；
2. 在内存中排序；
3. 写出一个 sorted run。

初始 run 数：

$$
\lceil b_r/M\rceil
$$

### 8.2 Phase 2: Merge Runs

如果 run 数 $N < M$，可以一次 merge 完：

- 每个 run 用一个 input buffer page；
- 另用一个 output buffer page。

如果 $N \ge M$，需要多轮 merge。

每轮最多合并 $M-1$ 个 runs，因此 merge pass 数：

$$
\lceil \log_{M-1}(b_r/M)\rceil
$$

### 8.3 Block Transfer Cost

简单版本中，总 block transfers：

$$
b_r\left(2\lceil \log_{M-1}(b_r/M)\rceil + 1\right)
$$

解释：

- run generation 读写一次：$2b_r$；
- 每个中间 merge pass 读写一次：$2b_r$；
- final pass 只算读，不算最终输出写盘。

### 8.4 Seek Cost

简单版本：

$$
2\lceil b_r/M\rceil + b_r\left(2\lceil \log_{M-1}(b_r/M)\rceil - 1\right)
$$

课件还给出 advanced version：每个 run 使用 $b_b$ 个 buffer blocks，减少 seek。

此时每轮可合并：

$$
\lfloor M/b_b\rfloor - 1
$$

个 runs。

---

## 9. Join Operation

常见 join 算法：

- nested-loop join；
- block nested-loop join；
- indexed nested-loop join；
- merge-join；
- hash-join。

选择哪种算法取决于：

- 是否有索引；
- 是否是 equi-join / natural join；
- relation 大小；
- buffer 大小；
- 是否已经排序；
- 代价估算。

---

## 10. Nested-Loop Join

计算：

$$
r \bowtie_\theta s
$$

伪代码：

```text
for each tuple tr in r:
  for each tuple ts in s:
    if tr and ts satisfy theta:
      output tr joined with ts
```

$r$ 称为 outer relation，$s$ 称为 inner relation。

优点：

- 不需要索引；
- 可用于任意 join 条件。

缺点：

- 检查每一对 tuple，非常贵。

最坏情况下，如果内存只能容纳每个 relation 一个 block：

block transfers：

$$
n_r \cdot b_s + b_r
$$

seeks：

$$
n_r + b_r
$$

如果较小 relation 能完全放入内存，把它作为 inner relation，代价可降为：

$$
b_r + b_s
$$

block transfers。

---

## 11. Block Nested-Loop Join

不是按 tuple 嵌套，而是按 block 嵌套：

```text
for each block Br of r:
  for each block Bs of s:
    compare tuples in Br and Bs
```

最坏 block transfers：

$$
b_r \cdot b_s + b_r
$$

seeks：

$$
2b_r
$$

### 11.1 使用更多 buffer

如果内存有 $M$ 个 blocks：

- 用 $M-2$ 个 blocks 存 outer relation 的 chunk；
- 1 个 block 给 inner relation；
- 1 个 block 给 output。

block transfers：

$$
\lceil b_r/(M-2)\rceil \cdot b_s + b_r
$$

seeks：

$$
2\lceil b_r/(M-2)\rceil
$$

如果 $r$ 可以放入内存，即 $b_r \le M-2$：

$$
b_r + b_s
$$

block transfers，约 2 次 seek。

---

## 12. Indexed Nested-Loop Join

适用条件：

- join 是 equi-join 或 natural join；
- inner relation 的 join attribute 上有索引。

做法：

```text
for each tuple tr in outer relation r:
  use index on s to find matching tuples
```

代价：

$$
b_r(t_T+t_S) + n_r \cdot c
$$

其中 $c$ 是对 inner relation 用 join 条件做一次 index lookup 并取回匹配 tuples 的代价。

如果两个 relation 的 join attributes 上都有索引，通常选择 tuple 数更少的 relation 作为 outer relation。

---

## 13. Merge-Join

适用于：

- equi-join；
- natural join。

步骤：

1. 若两个 relation 尚未按 join attribute 排序，先排序；
2. 类似 merge-sort 的 merge 阶段，同时扫描两个有序 relation；
3. 对 join attribute 相同的 tuples 输出匹配结果。

如果已排序，并且同一 join value 的 tuples 能放入内存：

block transfers：

$$
b_r + b_s
$$

seeks：

$$
\lceil b_r/b_b\rceil + \lceil b_s/b_b\rceil
$$

若未排序，还要加上排序成本。

### 13.1 Buffer 分配

若总内存为 $M$，分给 $r$ 的 buffer 为 $x_r$，分给 $s$ 的 buffer 为 $x_s$：

$$
x_r + x_s = M
$$

seek 代价近似：

$$
\lceil b_r/x_r\rceil + \lceil b_s/x_s\rceil
$$

最优分配近似为：

$$
x_r = \frac{\sqrt{b_r}M}{\sqrt{b_r}+\sqrt{b_s}}
$$

$$
x_s = \frac{\sqrt{b_s}M}{\sqrt{b_r}+\sqrt{b_s}}
$$

---

## 14. Hybrid Merge-Join

如果：

- 一个 relation 已按 join attribute 排序；
- 另一个 relation 在 join attribute 上有 secondary B+-tree index；

可以：

1. 把有序 relation 与 B+-tree leaf entries merge；
2. 得到未排序 relation 的 tuple addresses；
3. 按物理地址排序这些 addresses；
4. 顺序扫描未排序 relation，取回实际 tuples。

这样可以避免大量随机 lookup。

---

## 15. Hash-Join

适用于：

- equi-join；
- natural join。

### 15.1 基本思想

用 hash function $h$ 按 join attributes 对两个 relation 分区。

如果：

$$
h(t_r[JoinAttrs]) = i
$$

则 $t_r$ 放入 $r_i$。

如果：

$$
h(t_s[JoinAttrs]) = i
$$

则 $t_s$ 放入 $s_i$。

满足 join 条件的 tuple 必须落在同一编号分区中，所以只需要比较：

$$
r_i \text{ 与 } s_i
$$

### 15.2 Build Input 与 Probe Input

通常选择较小 relation 作为 build input。

算法：

1. 用 hash function $h$ 对 build input $s$ 分区；
2. 同样对 probe input $r$ 分区；
3. 对每个 $i$：
   1. 把 $s_i$ 读入内存，并在 join attribute 上建内存 hash index；
   2. 顺序读取 $r_i$；
   3. 用内存 hash index 找匹配的 $s_i$ tuples；
   4. 输出 join 结果。

要求：

> 每个 build partition $s_i$ 能放入内存。

通常分区数量 $n$ 取：

$$
\lceil b_s/M\rceil \times f
$$

其中 $f$ 是 fudge factor，常取约 1.2。

---

## 16. Recursive Partitioning

如果分区数超过内存可支持的数量，或者 build partition 仍放不下内存，就需要 recursive partitioning。

课件给出近似判断：

如果：

$$
M > \sqrt{b_s}
$$

则通常不需要 recursive partitioning。

直觉：

- 第一轮用 $M-1$ 个输出 buffer 做分区；
- 若内存足够大，每个 build partition 大小约小于 $M$，即可直接 build。

---

## 17. Hash-Join Cost

### 17.1 不需要 Recursive Partitioning

block transfers：

$$
3(b_r+b_s) + 4n_h
$$

解释：

- 读 $r,s$ 并写出分区：$2(b_r+b_s)$；
- build/probe 阶段再读一次所有分区：$b_r+b_s$；
- partially filled partition blocks 带来最多 $4n_h$ 的额外开销。

seeks：

$$
2(\lceil b_r/b_b\rceil + \lceil b_s/b_b\rceil) + 2n_h
$$

### 17.2 需要 Recursive Partitioning

partition build relation 所需 pass 数：

$$
\left\lceil \log_{\lfloor M/b_b\rfloor -1}(b_s/M)\right\rceil
$$

总 block transfers 估计：

$$
2(b_r+b_s)
\left\lceil \log_{\lfloor M/b_b\rfloor -1}(b_s/M)\right\rceil
+ b_r+b_s
$$

如果整个 build input 都能放入内存，则不需要分区，代价可降为：

$$
b_r + b_s
$$

---

## 18. Other Operations

### 18.1 Duplicate Elimination

可以用 sorting 或 hashing。

- sorting：重复 tuples 排在一起，删除多余副本；
- hashing：重复 tuples 落入同一 bucket。

优化：

- external sort 的 run generation 阶段就可以删除部分重复；
- merge 中间阶段也可以继续删除重复。

### 18.2 Projection

projection 通常分两步：

1. 对每个 tuple 取出需要属性；
2. 做 duplicate elimination。

因为关系代数中的 projection 默认去重。

### 18.3 Aggregation

aggregation 可以用 sorting 或 hashing 把同组 tuples 聚在一起。

优化：

- 在 run generation 或 intermediate merge 中提前计算 partial aggregate。

对不同聚合：

- `count`：部分 count 相加；
- `sum`：部分 sum 相加；
- `min/max`：取部分最小/最大；
- `avg`：保存 `sum` 和 `count`，最后相除。

### 18.4 Set Operations

集合操作：

$$
\cup,\quad \cap,\quad -
$$

可以用：

- sort 后类似 merge-join；
- hash 后按 partition 处理。

以 hashing 为例：

1. 用同一 hash function 分区 $r$ 和 $s$；
2. 对每个 partition：
   - 为 $r_i$ 建内存 hash index；
   - 处理 $s_i$。

规则：

- $r \cup s$：把 $s_i$ 中不在 hash index 的 tuple 加入，最后输出全部；
- $r \cap s$：输出 $s_i$ 中已经在 hash index 的 tuple；
- $r - s$：从 hash index 删除 $s_i$ 中出现的 tuple，最后输出剩余。

### 18.5 Outer Join

outer join 可以通过：

- inner join 后补充 null-padded non-participating tuples；
- 修改 join 算法直接产生 outer join。

例如 left outer join：

- merge join 中，若 $r$ 中某 tuple 没有匹配 $s$，输出该 tuple 并在 $s$ 属性位置补 null；
- hash join 中，如果 $r$ 是 probe relation，probe 后未匹配的 tuples 直接补 null 输出；
- 如果 $r$ 是 build relation，则 probing 时记录哪些 build tuples 被匹配，最后输出未匹配的 build tuples。

---

## 19. Evaluation of Expressions

前面讨论的是单个 operation 的算法。完整查询通常是一个 expression tree。

执行 expression tree 有两类方式：

- materialization；
- pipelining。

### 19.1 Materialization

materialized evaluation 自底向上一次执行一个 operation，并把中间结果存成临时关系。

优点：

- 总是适用；
- 实现简单。

缺点：

- 写中间结果到磁盘，再读回来，代价高。

总体代价：

```text
各个 operation 代价之和
+ 中间结果写盘/读盘代价
```

优化：

- double buffering：每个 operation 使用两个 output buffers；
- 一个 buffer 写盘时，另一个继续填充，从而重叠 I/O 和计算。

### 19.2 Pipelining

pipelined evaluation 不把中间结果写盘，而是把一个 operation 的输出 tuple 直接传给父 operation。

优点：

- 避免中间结果落盘；
- 通常比 materialization 便宜。

限制：

- 并非所有操作都能 pipeline；
- sort、hash-join 等阻塞型操作往往需要先看完整输入或建立结构。

---

## 20. Demand-Driven 与 Producer-Driven Pipeline

### 20.1 Demand-Driven / Lazy / Pull

顶层 operation 反复请求下一条 tuple。

每个 operation 在需要时向 child 请求 tuple。

常用 iterator 接口：

- `open()`：初始化；
- `next()`：返回下一条 tuple；
- `close()`：释放资源。

例子：

- file scan 的 `next()` 返回下一条记录；
- merge join 的 `next()` 从上次状态继续 merge，直到产生下一条 join 结果。

### 20.2 Producer-Driven / Eager / Push

child operation 主动产生 tuples 并推给 parent。

operator 之间有 buffer：

- child 把 tuple 放入 buffer；
- parent 从 buffer 取；
- 如果 buffer 满，child 等待。

系统调度那些输出 buffer 有空间、且可以继续处理输入的 operators。

---

## 21. Query Processing in Memory

现代数据库在主存查询处理中还会考虑：

### 21.1 Query Compilation

传统解释执行有开销，例如：

- 反复从 metadata 找 attribute 位置；
- 表达式逐步解释求值；
- 函数调用开销。

query compilation 可以把查询编译成机器码或中间码，例如：

- Java bytecode；
- LLVM；
- JIT compilation。

### 21.2 Column-Oriented Storage

列式存储配合 vectorized execution，可以一次处理一批同类型值，提高 CPU cache 和 SIMD 利用率。

### 21.3 Cache Conscious Algorithms

目标：

> 减少 cache miss，让一次 cache line 读取带来更多有用数据。

例子：

- sorting：run 大小尽量适合 L3 cache；
- hash join：
  - 先分区到 build+probe partitions 能放入内存；
  - 再进一步 subpartition，使 build subpartition + hash index 能放入 L3 cache；
- 经常一起访问的 tuple attributes 应相邻存储；
- 使用多线程隐藏 cache miss 的 stall。

---

## 22. 常用代价公式速查

| 操作 | 主要条件 | Block transfers |
| --- | --- | --- |
| Linear selection | 无索引 | $b_r$ |
| A2 primary index equality key | 返回单条 | $h_i + 1$ 个相关 I/O |
| A3 clustering index equality nonkey | 匹配连续 blocks | 索引路径 + $b$ |
| A4' secondary index equality nonkey | 匹配多条且分散 | 可能接近 $h_i + m + n$ 次随机 I/O |
| External sort | $M$ pages memory | $b_r(2\lceil\log_{M-1}(b_r/M)\rceil+1)$ |
| Nested-loop join | $r$ outer | $n_r b_s + b_r$ |
| Block nested-loop join | $r$ outer | $b_r b_s + b_r$ |
| Block nested-loop with M pages | $M-2$ pages for outer chunk | $\lceil b_r/(M-2)\rceil b_s + b_r$ |
| Merge join | 已排序 | $b_r+b_s$ |
| Hash join | 不递归分区 | $3(b_r+b_s)+4n_h$ |
| In-memory build join | build input fits memory | $b_r+b_s$ |

---

## 23. 本节速记

- query processing 包括 parsing/translation、optimization、evaluation。
- cost model 主要估算 block transfers 和 seeks。
- selection 是否有索引、索引是否聚簇、结果是否多且分散，会极大影响代价。
- secondary non-clustering index 在返回大量记录时可能非常慢。
- external sort-merge 先生成 sorted runs，再多轮 merge。
- nested-loop join 通用但贵；block nested-loop 用 block 降低代价。
- indexed nested-loop 依赖 inner relation 的 join attribute 索引。
- merge join 适合已排序的 equi-join / natural join。
- hash join 适合 equi-join / natural join，关键是 build partitions 能放入内存。
- materialization 总是可行但中间结果 I/O 重。
- pipelining 避免中间结果落盘，但不适合所有操作。
- 主存查询处理中，cache、vectorization、JIT compilation 也很重要。
